---
title: Mindgames
tags: Tryhackme ctf
article_header:
  type: cover
published: true
author: krill
---
# Mindgames 
Just a terrible idea...

## Enumeration

so much emphasis on enumeration, it is the key to successfully exploit a box.
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
![page 80](/assets/images/THM/mindgames/page1.png)

`Brainfuck is an esoteric programming language created in 1993 by a Swiss physics student named Urban Müller, Brainfuck was an attempt to create a language with the smallest possible compiler` 

Also, right beneath, we have what looks like a brainfuck code interpreter; Check it out!   
Pasting the Fibonacci brainfuck code, gives what looks like a Fibonacci sequence of number. .  mmmm..  
![compiler](/assets/images/THM/mindgames/page2.png)

Decoding this with `[dcode](https://www.dcode.fr/brainfuck-language)` gives what looks like a python code; noice :ghost: 

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

## Exploitation
### Foothold: Initial access

