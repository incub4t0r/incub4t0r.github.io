---
layout: post
title: Image Encryption - Angstrom
subtitle: My solution to the Image Encryption challenge on AngstromCTF
image: /assets/img/angstrom/postimage.png
tags: angstromctf
---

This was a fun one, it required me to annotate the source code and figure out a decryption method for the given picture.


## TL;DR

![](/assets/img/angstrom/enc.png)

The file image-encryption.py is opening a target image file, in this case, img.png.

The program then creates an array of the pixels that the image contains, then parses through each pixel. Upon arriving at each pixel, the program then stores into a variable what is assumably the RGB values of each pixel. Each RGB value is then multiplied by the respective key value, with R = 41, G = 37, B = 23, and the a modulo operator is performed on the product. The RGB values are then stored into a new image of the same size, creating the "encrypted" image.


I created a brute force image decryption program by parsing through 0-255, multiplying each value by the respective key operator, performing a modulo operation of 251, and then finally comparing the result to each encrypted pixel in the encrypted image. If the parsed value equated the encrypted image value, then it was stored into an pixel list until each RGB value was completed. The RGB value was then stored into a new image called dec.png.

Flag was present in dec.png

![](/assets/img/angstrom/dec.png)

---
## Solve

image-encryption.py 

enc.png

## Hints


---
## Work + Notes

### img_dec_final.py

```
from numpy import *
from PIL import Image

target = Image.open(r"enc.png")
img = array(target)

key = [41,37,23]
a, b, c = img.shape

for x in range (0, a): 
    for y in range (0, b): 
        pixel = img[x, y] 
        for i in range(0,3):
            h=0
            while h<255: 
                if (h * key[i] % 251 == pixel[i]): 
                    pixel[i] = h
                h+=1
        img[x][y] = pixel

dec = Image.fromarray(img)
dec.save('dec.png')


```
 
