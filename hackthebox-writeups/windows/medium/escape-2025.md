# Escape (2025)



<figure><img src="../../../.gitbook/assets/image (221).png" alt=""><figcaption></figcaption></figure>



```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Escape]
└─$ nmap -sC -sV 10.129.2.114 -oN nmap.out
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-03 11:37 EST
Nmap scan report for 10.129.2.114
Host is up (0.046s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-12-04 00:37:45Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2025-12-04T00:39:05+00:00; +8h00m00s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-12-04T00:39:05+00:00; +8h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   10.129.2.114:1433: 
|     Target_Name: sequel
|     NetBIOS_Domain_Name: sequel
|     NetBIOS_Computer_Name: DC
|     DNS_Domain_Name: sequel.htb
|     DNS_Computer_Name: dc.sequel.htb
|     DNS_Tree_Name: sequel.htb
|_    Product_Version: 10.0.17763
|_ssl-date: 2025-12-04T00:39:05+00:00; +8h00m00s from scanner time.
| ms-sql-info: 
|   10.129.2.114:1433: 
|     Version: 
|       name: Microsoft SQL Server 2019 RTM
|       number: 15.00.2000.00
|       Product: Microsoft SQL Server 2019
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2025-12-04T00:30:46
|_Not valid after:  2055-12-04T00:30:46
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
|_ssl-date: 2025-12-04T00:39:05+00:00; +8h00m00s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: sequel.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-12-04T00:39:05+00:00; +8h00m00s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc.sequel.htb, DNS:sequel.htb, DNS:sequel
| Not valid before: 2024-01-18T23:03:57
|_Not valid after:  2074-01-05T23:03:57
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h59m59s, deviation: 0s, median: 7h59m59s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-12-04T00:38:25
|_  start_date: N/A
```







```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Escape]
└─$ nxc smb 10.129.2.114 -u 'guest' -p '' 
SMB         10.129.2.114    445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.114    445    DC               [+] sequel.htb\guest: 
                                                                                                                                                                
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Escape]
└─$ nxc smb 10.129.2.114 -u 'guest' -p '' --shares
SMB         10.129.2.114    445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:sequel.htb) (signing:True) (SMBv1:False)
SMB         10.129.2.114    445    DC               [+] sequel.htb\guest: 
SMB         10.129.2.114    445    DC               [*] Enumerated shares
SMB         10.129.2.114    445    DC               Share           Permissions     Remark
SMB         10.129.2.114    445    DC               -----           -----------     ------
SMB         10.129.2.114    445    DC               ADMIN$                          Remote Admin
SMB         10.129.2.114    445    DC               C$                              Default share
SMB         10.129.2.114    445    DC               IPC$            READ            Remote IPC
SMB         10.129.2.114    445    DC               NETLOGON                        Logon server share 
SMB         10.129.2.114    445    DC               Public          READ            
SMB         10.129.2.114    445    DC               SYSVOL                          Logon server share
```











