---
layout: post
title: All Your Base Are Belong To Us
subtitle: My solution to the All Your Base Are Belong To Us challenge on ACICTF
image: /assets/img/acictf/Aybabtu.png
tags: acictf
published: true
---

This problem cost me 5 years of my life. It shouldn't have been this hard, but here we go.


## TL;DR


When you netcat to the provided web address, you are given this output.
```
  This program is going to ask you to convert among 6 different bases a total 
  of 5 times.  Each question is placed inside of lines delimited by 78
  '-' characters.  The first line of each question indicates the base we are
  giving to you as well as the base we expect the result in and looks like:
          [src_base] -> [answer base]
  The next line of the question is the source text that we want you to convert
  into the new base. Your answer should be followed by a newline character.

  All of the encodings treat an underlying printable ASCII string as a
  big-endian number.  If that doesn't make a lot of sense, don't worry about
  it: most of the tools you'd look to use (Python, websites, etc.) generally
  assume this anyways.  Except for 'raw' and 'b64', there will never be
  leading 0s at the start of the answer.

  Formatting key:
          raw = the unencoded ASCII string (contains only printable characters
                    that are not whitespace)
          b64 = standard base64 encoding (see 'base64' unix command)
          hex = hex (base 16) encoding (case insensitive)
          dec = decimal (base 10) encoding
          oct = octal (base 8) encoding
          bin = binary (base 2) encoding (should consist of ASCII '0' and '1')
    
------------------------------------------------------------------------------
b64 -> raw
KXhYKiE3Q2trK3diay5fLVt8ekdHVUM0QjQ0LnF7Szg8KTp0MDt0bXhieXwxSCgpQG1Kek1FU0dbais2Zj1leA==
------------------------------------------------------------------------------
answer: 

```
So we need to decode and re-encode the given output 5 times. I opted to not use the starter_code given to us and manually encrypt/decrypt

The way I looked at it, we need to decode the given string to a raw format, and then encode back to the specified format. This can be done by using a whole bunch of if and elif statements. I particularly struggled with the format of each encryption and decryption. 

I learned a lot about the format that each encryption and decryption uses, and below is the code.

```
#!/usr/bin/python3
import argparse
import socket
import base64
import binascii

#python3 starter_code.py challenge.acictf.com 52062
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect((args.ip, args.port))

f = sock.makefile()
parseNum = 0
iterNum = 0
while True:
    line = f.readline().strip()
    srcdecoded =""
    if line == ('------------------------------------------------------------------------------'):
        if parseNum == 0:
            line = f.readline().strip()
            #print(line)
            cfrom = line[0:3]
            cto = line[7:]
            #print ("source encoding: " + cfrom)
            #print ("destination encoding: " + cto)
            parseNum+=1
        else:
            sock.send((answer + "\n").encode())
            parseNum = 0
            if iterNum == 5:
                line = f.readline().strip()
                line = f.readline().strip()
                line = f.readline().strip()

                print (line)
            continue
        iterNum += 1   
        print(iterNum)
        line = f.readline().strip()

        answer = ''
        value = ''
        pv = line

        if (cfrom == 'b64'):
            value = binascii.a2b_base64(pv)
            value = int.from_bytes(value, "big")
        elif (cfrom == 'raw'):# or cfrom == 'b64'):
             value = sum(ord( pv[byte])<<8*(len(pv)-byte-1) for byte in range(len(pv)) )
        elif (cfrom == 'hex'):
            value = int(line, 16)
        elif (cfrom == 'dec'):
            value = int(line)
        elif (cfrom == 'oct'):
            value = int(line, 8)
        elif (cfrom == 'bin'):
            value = int(line, 2)

        if (cto == 'raw' or cto == 'b64'):
            answer = value.to_bytes((value.bit_length()+7)//8, 'big')
        elif (cto == 'hex'):
            answer = hex(value)[2:]
        elif (cto == 'dec'):
            answer = str(value)
        elif (cto == 'oct'):
            answer = oct(value)[2:]
        elif (cto == 'bin'):
                
            answer = bin(value)[2:]
        if (cto == 'b64'):
            answer = base64.b64encode(answer)   
        try:
            answer = answer.decode()
        except (UnicodeDecodeError, AttributeError):
            pass    
```


---

## Solve
```
In honor of 30 years of terrible translations, we figured we'd give you a try at a series of (easier) translation problems. All you have to do is to translate bases by connecting to challenge.acictf.com:52062. In case you're new to network programs, we even have some Python starter code you can use.
```

## Hints
```
You could do this by hand, but is it really worth that much effort?

While we only want the final encoding, it's probably easier to break that into separate decode and encode steps for each question.

Don't overthink 'raw' encoding...

Your code for encoding/decoding will probably be very similar for 4 out of 6 encodings.
```