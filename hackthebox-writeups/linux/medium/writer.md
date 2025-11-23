---
hidden: true
---

# Writer

<figure><img src="../../../.gitbook/assets/image (151).png" alt=""><figcaption></figcaption></figure>

## Ключевые моменты прохождения:

✅

## Разведка

Сканируем Nmap'ом:

```
$ nmap -sC -sV 10.10.11.101
```

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:20:b9:d0:52:1f:4e:10:3a:4a:93:7e:50:bc:b8:7d (RSA)
|   256 10:04:79:7a:29:74:db:28:f9:ff:af:68:df:f1:3f:34 (ECDSA)
|_  256 77:c4:86:9a:9f:33:4f:da:71:20:2c:e1:51:10:7e:8d (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Story Bank | Writer.HTB
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_clock-skew: 1s
| smb2-time: 
|   date: 2021-12-16T13:54:01
|_  start_date: N/A
|_nbstat: NetBIOS name: WRITER, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
```

Итак, дефолтные 22 и 80, а также 139 и 445. Можно посмотреть доступные шары:

```
$ smbmap -H 10.10.11.101
```

```
[+] IP: 10.10.11.101:445        Name: writer.htb                                        
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        writer2_project                                         NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (writer server (Samba, Ubuntu))
```

К сожалению, доступных шар нету.

```
$ enum4linux -a 10.10.11.101
```

```
...
============================= 
|   Users on 10.10.11.101   |
============================= 
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: kyle     Name: Kyle Travis       Desc:

user:[kyle] rid:[0x3e8]
...
```

Большинство вывода enum4linux я убрал, оставил только самое полезное - юзеров, которых он нашел.

На 80 порту у нас стоит сайт:

```
$ whatweb -v http://10.10.11.101/
```

```
WhatWeb report for http://10.10.11.101/
Status    : 200 OK
Title     : Story Bank | Writer.HTB
IP        : 10.10.11.101
Country   : RESERVED, ZZ

Summary   : HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], Apache[2.4.41], JQuery, Script, HTML5
```

В принципе, ничего особенного.

<figure><img src="../../../.gitbook/assets/image (152).png" alt=""><figcaption></figcaption></figure>

Можно пробрутить диры:

```
$ gobuster dir -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -u http://10.10.11.101/ -t 20
```

```
/contact              (Status: 200) [Size: 4905]
/logout               (Status: 302) [Size: 208] [--> http://10.10.11.101/]
/about                (Status: 200) [Size: 3522]                          
/static               (Status: 301) [Size: 313] [--> http://10.10.11.101/static/]
/.                    (Status: 200) [Size: 11971]                                
/dashboard            (Status: 302) [Size: 208] [--> http://10.10.11.101/]       
/server-status        (Status: 403) [Size: 277]                                  
/administrative       (Status: 200) [Size: 1443]
```

На странице /administrative нас встречает окно авторизации:

<figure><img src="../../../.gitbook/assets/image (153).png" alt=""><figcaption></figcaption></figure>

Его можно обойти самой простой SQL-инъекцией:

<figure><img src="../../../.gitbook/assets/image (154).png" alt=""><figcaption></figcaption></figure>

Дальше, лазая по сайту, я ничего не нашел и решил вернуться к SQL-инъекции:

<figure><img src="../../../.gitbook/assets/image (155).png" alt=""><figcaption></figcaption></figure>

Данным способом мы можем перебрать количество столбцов в таблице.

<figure><img src="../../../.gitbook/assets/image (156).png" alt=""><figcaption></figcaption></figure>

Ошибки нету, значит их 6.

Мы можем посмотреть юзера:

<figure><img src="../../../.gitbook/assets/image (158).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (159).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (160).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (161).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (162).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (163).png" alt=""><figcaption></figcaption></figure>

```ini
Welcome # Virtual host configuration for writer.htb domain
&lt;VirtualHost *:80&gt;
        ServerName writer.htb
        ServerAdmin admin@writer.htb
        WSGIScriptAlias / /var/www/writer.htb/writer.wsgi
        &lt;Directory /var/www/writer.htb&gt;
                Order allow,deny
                Allow from all
        &lt;/Directory&gt;
        Alias /static /var/www/writer.htb/writer/static
        &lt;Directory /var/www/writer.htb/writer/static/&gt;
                Order allow,deny
                Allow from all
        &lt;/Directory&gt;
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
&lt;/VirtualHost&gt;

# Virtual host configuration for dev.writer.htb subdomain
# Will enable configuration after completing backend development
# Listen 8080
#&lt;VirtualHost 127.0.0.1:8080&gt;
#	ServerName dev.writer.htb
#	ServerAdmin admin@writer.htb
#
        # Collect static for the writer2_project/writer_web/templates
#	Alias /static /var/www/writer2_project/static
#	&lt;Directory /var/www/writer2_project/static&gt;
#		Require all granted
#	&lt;/Directory&gt;
#
#	&lt;Directory /var/www/writer2_project/writerv2&gt;
#		&lt;Files wsgi.py&gt;
#			Require all granted
#		&lt;/Files&gt;
#	&lt;/Directory&gt;
#
#	WSGIDaemonProcess writer2_project python-path=/var/www/writer2_project python-home=/var/www/writer2_project/writer2env
#	WSGIProcessGroup writer2_project
#	WSGIScriptAlias / /var/www/writer2_project/writerv2/wsgi.py
#        ErrorLog ${APACHE_LOG_DIR}/error.log
#        LogLevel warn
#        CustomLog ${APACHE_LOG_DIR}/access.log combined
#
#&lt;/VirtualHost&gt;
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

После 15 картинки: Часть кода из **init**.py

```python
...
if request.form.get('image_url'):
            image_url = request.form.get('image_url')
            if ".jpg" in image_url:
                try:
                    local_filename, headers = urllib.request.urlretrieve(image_url)
                    os.system("mv {} {}.jpg".format(local_filename, local_filename))
                    image = "{}.jpg".format(local_filename)
                    try:
                        im = Image.open(image) 
                        im.verify()
                        im.close()
                        image = image.replace('/tmp/','')
                        os.system("mv /tmp/{} /var/www/writer.htb/writer/static/img/{}".format(image, image))
                        image = "/img/{}".format(image)
                    except PIL.UnidentifiedImageError:
                        os.system("rm {}".format(image))
                        error = "Not a valid image file!"
                        return render_template('add.html', error=error)
...
```

<...Должно быть продолжение...>
