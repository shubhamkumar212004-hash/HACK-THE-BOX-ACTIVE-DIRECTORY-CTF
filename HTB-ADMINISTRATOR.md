![](https://miro.medium.com/v2/resize:fit:875/1*BHh4RMba3PfwWc8XvxoF1A.png)

**Introduction**

“Administrator” is an Active Directory (AD) machine on Hack The Box that challenges your Windows enumeration, Kerberos authentication, and privilege escalation skills. This box required a combination of Kerberos-based attacks, credential extraction, and abuse of misconfigured permissions to gain administrative access. The toughest part was finding the right attack path, but persistence paid off. Let’s break it down step by step!

# **Enumeration**

We started by performing an initial scan of the target machine (10.10.11.42) using **Nmap**:

```
nmap -sS -sV -sC -Pn 10.10.11.42
```

The scan revealed multiple open ports, including:

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*OksebW5rmeElJG3c_ojjIg.png)

As we can use we have many interesting ports opened one of which being ftp, we have a simple dns on port 53 and kerberos on port 88 . We also have smb, which can be very helpful in enumerating users.

before that do add administrator.htb to your /etc/hosts

![](https://miro.medium.com/v2/resize:fit:711/1*1A1yLrg7GaUmKY62jil6aw.png)

As is common in real life Windows pentests, you will start the Administrator box with credentials for the following account: Username: **`Olivia`**Password:**`ichliebedich`**

Through the SMB service, the current user information is obtained

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*-XMDsFqPk2AZyqFTQzaNXA.png)

I use here bloodhound

> *In simple terms, **BloodHound helps attackers (and defenders) understand how users and computers are connected in an Active Directory network to find weak spots for privilege escalation.***
> 

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*k-gC53TITeDyk7pAx-U-8Q.png)

**ACTIVE DIRECTORY PARTS**

After analysis, Olivia can control Michael

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*kI3cqwa3pOcqottL07shFg.png)

Michael can force Benjamin to change his password

![](https://miro.medium.com/v2/resize:fit:699/1*6l84PbbQqtppkVidonux8Q.png)

> ***USER***
> 
> 
> ***BloodyAD is used to test and exploit security flaws in Active Directory’s certificate-based authentication system** during red team assessments or penetration tests*
> 

First change Michael’s password

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*xO85JoTiQS_JSkhFMvFjCw.png)

Then use Michael’s account to change Benjamin’s password

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*eJjtmE5LCCGCffMMXFChxw.png)

if you still remember me mentioning that we have port 21 open that is FTP, you are thinking right we will check if we can log into that ftp account using Benjamin’s Credentials

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*R6YZQzh_iUNr9VP6eiryXw.png)

psafe3 files are encrypted password-safe files

Cannot be read directly,

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*gHprppHX9CvSzlxUN1XYbA.png)

need to use **`pwsafe2john`**tools to obtain hash, Kali comes with

![](https://miro.medium.com/v2/resize:fit:456/1*c2sG0rwTldfIxM8HxB_FvA.png)

Get the hash and try to decrypt it

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*PN1yMgIEnh4V9FjbPHvyMQ.png)

The password is**`tekieromucho`**

[Become a member](https://miro.medium.com/v2/da:true/resize:fit:0/60026f4340686a391639ac58864da18070aa773cea45de6e55fa47fd56bfdb74)

Install Passwordsafe in Kali, then open this file with the password

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:715/1*SaJ-foITG79YnCTaaKEPpA.png)

![](https://miro.medium.com/v2/resize:fit:505/1*_u1HCZ_oQEDaMa1krDKZhw.png)

Right click on the file Copy the PASSWORD of the three users

![](https://miro.medium.com/v2/resize:fit:498/1*ygLRF9BZSGiKslerdkwmpQ.png)

Since port 5985 is open on the target machine, you can use evil-winrm to log in remotely

> ***we get first flag here***
> 

bingo 🎉🎯

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*hJC_tz8credHpvi3CyXU5A.png)

# **Privilege Escalation**

Emliy has write access to Ethan

![](https://miro.medium.com/v2/resize:fit:745/1*aAtyIXKWne_EQUBJeViMkw.png)

# **Own Ethan**

Because of Emily’s access to Ethan, he can use Targeted Kerberoasting attack

targetedKerberoast is a Python script that, like many others (e.g. https://github.com/ShutdownRepo/targetedKerberoast/blob/main/targetedKerberoast.py), prints a “kerberoast” hash for user accounts that have SPNs set. This tool brings the following additional functionality: for each user that does not have an SPN, it attempts to set one ( abusing write access to the attribute ), prints the “kerberoast” hash, and deletes the temporary SPN that was set for the operation.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*Bm4qO5AqSoBQ8b4iT_gZew.png)

If the above cannot be executed, you can try to synchronize the time zone, because Kerberos authentication is very strict about time calibration.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*-hAIJhK9F0Yob51NHQwayA.png)

Decrypting the hash

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*4kUTO3pO4csEvFB6fcOQMQ.png)

Get Ethan’s password:**`limpbizkit`**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:1250/1*I9FttClYI6KHLNs-phVKPg.png)

# **Ablout DCSync**

From here: [DCSync(bloodhoundenterprise.io)](https://support.bloodhoundenterprise.io/hc/en-us/articles/17322385609371-DCSync)

This edge represents the combination of GetChanges and GetChangesAll. The combination of these two permissions grants the principal the ability to perform a DCSync attack.

Use this to get the Administrator’s password hash

# **secretsdump**

Secretsdump.py is a script in the Impacket framework that can also export the hash of users on the domain controller through the DCSync technology. The principle of this tool is to first use the provided user login credentials to remotely connect to the domain controller through smbexec or wmiexec and obtain high permissions, and then export the hash of local accounts from the registry, and export the hash of all domain users through Dcsync or from the NTDS.dit file. Its biggest advantage is that it supports connecting to the domain controller from computers outside the domain.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*4IcyXAlCDegnBO-O8vD_VA.png)

Then use the evil-winrm hash to log in

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*rb-l6Hg0Qc0qqrsvRKBbXw.png)

We are root user finally

> ***final flag***
> 

![](https://miro.medium.com/v2/resize:fit:609/1*e2Yr_u40uNMlXYTvGBiOKQ.png)

Congratulation 🎉🎯                                 

We are learn about active directory.
