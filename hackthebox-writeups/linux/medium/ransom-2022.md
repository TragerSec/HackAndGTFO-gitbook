# Ransom (2022)

## Прохождение Hackthebox::Ransom

<figure><img src="../../../.gitbook/assets/image (77).png" alt=""><figcaption></figcaption></figure>

### Тэги:

* Linux
* Web
* PHP
* API
* Web Fuzzing

## Сканируем порты с помощью nmap

```
$ nmap -sC -sV 10.10.11.153 -oN nmap
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea:84:21:a3:22:4a:7d:f9:b5:25:51:79:83:a4:f5:f2 (RSA)
|   256 b8:39:9e:f4:88:be:aa:01:73:2d:10:fb:44:7f:84:61 (ECDSA)
|_  256 22:21:e9:f4:85:90:87:45:16:1f:73:36:41:ee:3b:32 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-title:  Admin - HTML5 Admin Template
|_Requested resource was http://10.10.11.153/login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

На `80` порту есть страница авторизации:

<figure><img src="../../../.gitbook/assets/image (78).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>

Отправляем в `Repeater` и начинаем тестировать. Мои попытки с `SQLi` закончились неудачей, поэтому я начал копать глубже, изменив запрос, и нашел то, что `API` передает данные назад в формате `JSON`:

<figure><img src="../../../.gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

Следовательно мы можем добавить заголовки `Content-Length` (ставится автоматически в BurpSuite) и Content-Type и попытаться передать параметр password в `JSON`:

<figure><img src="../../../.gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure>

И он успешно отправился в JSON. Тогда можно отправить в качестве пароля значение `true` и если проверка сделана некачественно, то такой запрос пройдет, так как JSON обрежет кавычки. Вот пример такой проверки:

```php
if ($password == 'SuperSecurePassword') {
    echo "You logged";
} else {
    echo "Nope, try again!";
}
```

Два знака равно не проверяют тип в PHP, следовательно `true == 'SuperSecurePassword'` будет выполняться:

<figure><img src="../../../.gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

При этом, если добавить три знака равно, то будет проверка на тип данных и такой трюк не сработает:

<figure><img src="../../../.gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure>

Пробуем, работает:

<figure><img src="../../../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

Теперь мы можем просто зайти обратно на сайт, перехватить запрос и вставить тот, который мы сейчас использовали:

<figure><img src="../../../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

У нас есть флаг юзера и зашифрованный архив. Качаем последнего и смотрим, что в нем:

```
$ 7z l -slt uploaded-file-3422.zip
```

```
7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,12 CPUs AMD Ryzen 5 2600 Six-Core Processor             (800F82),ASM,AES-NI)

Scanning the drive for archives:
1 file, 7735 bytes (8 KiB)

Listing archive: uploaded-file-3422.zip

--
Path = uploaded-file-3422.zip
Type = zip
Physical Size = 7735

----------
Path = .bash_logout
Folder = -
Size = 220
Packed Size = 170
Modified = 2020-02-25 08:03:22
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = +
Comment = 
CRC = 6CE3189B
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .bashrc
Folder = -
Size = 3771
Packed Size = 1752
Modified = 2020-02-25 08:03:22
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = +
Comment = 
CRC = AB254644
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .profile
Folder = -
Size = 807
Packed Size = 404
Modified = 2020-02-25 08:03:22
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = +
Comment = 
CRC = D1B22A87
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .cache
Folder = +
Size = 0
Packed Size = 0
Modified = 2021-07-02 14:58:14
Created = 
Accessed = 
Attributes = D_ drwx------
Encrypted = -
Comment = 
CRC = 
Method = Store
Host OS = Unix
Version = 10
Volume Index = 0

Path = .cache/motd.legal-displayed
Folder = -
Size = 0
Packed Size = 12
Modified = 2021-07-02 14:58:14
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = +
Comment = 
CRC = 00000000
Method = ZipCrypto Store
Host OS = Unix
Version = 10
Volume Index = 0

Path = .sudo_as_admin_successful
Folder = -
Size = 0
Packed Size = 12
Modified = 2021-07-02 14:58:19
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = +
Comment = 
CRC = 00000000
Method = ZipCrypto Store
Host OS = Unix
Version = 10
Volume Index = 0

Path = .ssh
Folder = +
Size = 0
Packed Size = 0
Modified = 2022-03-07 08:32:54
Created = 
Accessed = 
Attributes = D_ drwxrwxr-x
Encrypted = -
Comment = 
CRC = 
Method = Store
Host OS = Unix
Version = 10
Volume Index = 0

Path = .ssh/id_rsa
Folder = -
Size = 2610
Packed Size = 1990
Modified = 2022-03-07 08:32:25
Created = 
Accessed = 
Attributes = _ -rw-------
Encrypted = +
Comment = 
CRC = 38804579
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .ssh/authorized_keys
Folder = -
Size = 564
Packed Size = 475
Modified = 2022-03-07 08:32:46
Created = 
Accessed = 
Attributes = _ -rw-------
Encrypted = +
Comment = 
CRC = CB143C32
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .ssh/id_rsa.pub
Folder = -
Size = 564
Packed Size = 475
Modified = 2022-03-07 08:32:54
Created = 
Accessed = 
Attributes = _ -rw-------
Encrypted = +
Comment = 
CRC = CB143C32
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0

Path = .viminfo
Folder = -
Size = 2009
Packed Size = 581
Modified = 2022-03-07 08:32:54
Created = 
Accessed = 
Attributes = _ -rw-------
Encrypted = +
Comment = 
CRC = 396B04B4
Method = ZipCrypto Deflate
Host OS = Unix
Version = 20
Volume Index = 0
```

Итак, для архива мы можем применить `атаку на основе открытых текстов`(plaintext attack)(https://ru.wikipedia.org/wiki/Атака\_на\_основе\_открытых\_текстов), так как в архиве есть стандартный файл `.bash_logout`.

Отрывок из википедии про атаку на основе открытых текстов:

***

В то же время различные зашифрованные архивы, такие как ZIP, уязвимы для данной формы атаки. В данном случае злоумышленнику, желающему вскрыть группу зашифрованных ZIP файлов, необходимо знать всего один незашифрованный файл из архива или его часть, который в данном случае будет выполнять функцию открытого текста. Далее, с использованием программ, находящихся в свободном доступе, быстро находится ключ, необходимый для расшифровки всего архива. Взломщик может попытаться найти данный незашифрованный файл в Интернете либо в других архивах, либо может попытаться восстановить открытый текст, зная имя из зашифрованного архива.

***

Смысл атаки в том, что для того, чтобы взломать zip-архив нужно знать 12 байтов одного файла этого архива, 8 из которых должны быть различными. Чем больше байтов знаешь, тем быстрее идет расшифровка архива.

То есть, по факту нам нужен просто стандартный файл `.bash_logout` с размером 220 и с `CRC`-кодом `6CE3189B`. Создаем его и вписываем такие данные:

```bash
# ~/.bash_logout: executed by bash(1) when login shell exits.

# when leaving the console clear the screen to increase privacy

if [ "$SHLVL" = 1 ]; then
    [ -x /usr/bin/clear_console ] && /usr/bin/clear_console -q
fi
```

Запаковываем его в архив:

```
$ zip bash_logout.zip .bash_logout
```

И смотрим его размер и `CRC`-код:

```
$ 7z l -slt bash_logout.zip 
```

```
7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,12 CPUs AMD Ryzen 5 2600 Six-Core Processor             (800F82),ASM,AES-NI)

Scanning the drive for archives:
1 file, 332 bytes (1 KiB)

Listing archive: bash_logout.zip

--
Path = bash_logout.zip
Type = zip
Physical Size = 332

----------
Path = .bash_logout
Folder = -
Size = 220
Packed Size = 158
Modified = 2022-03-16 17:55:54
Created = 
Accessed = 
Attributes = _ -rw-r--r--
Encrypted = -
Comment = 
CRC = 6CE3189B
Method = Deflate
Host OS = Unix
Version = 20
Volume Index = 0
```

Теперь с помощью [данной тулзы](https://github.com/kimci86/bkcrack) можно расшифровать архив:

<figure><img src="../../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

```
$ /opt/bkcrack-1.3.5-Linux/bkcrack -C uploaded-file-3422.zip -c .bash_logout -P bash_logout.zip  -p .bash_logout 
```

```
bkcrack 1.3.5 - 2022-03-06
[18:00:35] Z reduction using 150 bytes of known plaintext
100.0 % (150 / 150)
[18:00:35] Attack on 57097 Z values at index 7
Keys: 7b549874 ebc25ec5 7e465e18
78.8 % (44978 / 57097)
[18:01:03] Keys
7b549874 ebc25ec5 7e465e18
```

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

Вместо anypass можно вставить любое значение, это новый пароль для расшифрованного архива:

```
$ /opt/bkcrack-1.3.5-Linux/bkcrack -C uploaded-file-3422.zip -k 7b549874 ebc25ec5 7e465e18 -U cracked.zip anypass
```

<figure><img src="../../../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

Мы можем узнать имя юзера через authorized\_keys:

```
$ cat .ssh/authorized_keys
```

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDrDTHWkTw0RUfAyzj9U3Dh+ZwhOUvB4EewA+z6uSunsTo3YA0GV/j6EaOwNq6jdpNrb9T6tI+RpcNfA+icFj+6oRj8hOa2q1QPfbaej2uY4MvkVC+vGac1BQFs6gt0BkWM9JY7nYJ2y0SIibiLDDB7TwOx6gem4Br/35PW2sel8cESyR7JfGjuauZM/DehjJJGfqmeuZ2Yd2Umr4rAt0R4OEAcWpOX94Tp+JByPAT5m0CU557KyarNlW60vy79njr8DR8BljDtJ4n9BcOPtEn+7oYvcLVksgM4LB9XzdDiXzdpBcyi3+xhFznFKDYUf6NfAud2sEWae7iIsCYtmjx6Jr9Zi2MoUYqWXSal8o6bQDIDbyD8hApY5apdqLtaYMXpv+rMGQP5ZqoGd3izBM9yZEH8d9UQSSyym/te07GrCax63tb6lYgUoUPxVFCEN4RmzW1VuQGvxtfhu/rK5ofQPac8uaZskY3NWLoSF56BQqEG9waI4pCF5/Cq413N6/M= htb@ransom
```

Подключаемся:

<figure><img src="../../../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

Мы можем посмотреть по стандартному пути, где находится папка сайта:

```
$ cat /etc/apache2/sites-enabled/000-default.conf
```

```apache
<VirtualHost *:80>
	ServerAdmin webmaster@localhost
	DocumentRoot /srv/prod/public

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
	    <Directory /srv/prod/public>
	       Options +FollowSymlinks
	       AllowOverride All
	       Require all granted
	    </Directory>

</VirtualHost>
```

Далее, полазив по дирам, находим файл(`/srv/prod/app/Http/Controllers/AuthController.php`), который как раз-таки и был тем файлом, который использовался для доступа к E Corp Secure File Transfer на 80 порту:

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use App\Http\Requests\RegisterRequest;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;



class AuthController extends Controller
{
    /**
     * Display login page.
     * 
     * @return \Illuminate\Http\Response
     */
    public function show_login()
    {
        return view('auth.login');
    }



    /**
     * Handle account login
     * 
     */
    public function customLogin(Request $request)
    {
        $request->validate([
            'password' => 'required',
        ]);

        if ($request->get('password') == "UHC-March-Global-PW!") {
            session(['loggedin' => True]);
            return "Login Successful";
        }
  
        return "Invalid Password";
    }

}
```

И тут действительно плохая проверка, которая не проверяет тип переменной. Пароль - `UHC-March-Global-PW!`, который подходит для авторизации на сайте, а также для пользователя `root`:

<figure><img src="../../../.gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>
