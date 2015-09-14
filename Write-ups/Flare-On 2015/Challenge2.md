# Flare-On Challenge #2
#### By: Sage

You can single step the first instruction and arrive at `0x00401000`. The code here looks identical to the previous challenge except that the loop has been replaced with a function call.

The function performs a check at `0x0040108E` to confirm that your input is at least 0x25, or 37 characters. If you fail this check you go to the badboy message. If you pass the check then you go onto a loop that further scrutinizes your input.

The loop is doing a much more complicated XOR than in the last challenge. In my opinion assembly is harder to understand than Python. So I read each line of assembly and put the code that would be semantically equivalent in a Python script named algorithm.py. The retarded and utterly hideous code is below for you to laugh at.

```python
#!/usr/bin/env python
import sys

# setup
stack = []
if len(sys.argv) == 2:
    esi_input = list(sys.argv[1])
else:
    esi_input = list("a_Litubbbbcccccdddddeeeeefffffggggg\r\n")

edi_unknownbytes = "00 00 00 39 4D 10 7C 3F 8B 75 0C 8B 7D 08 8D 7C 0F FF 66 89 DA 66 83 E2 03 66 B8 C7 01 50 9E AC 9C 32 44 24 04 86 CA D2 C4 9D 10 E0 86 CA 31 D2 25 FF 00 00 00 66 01 C3 AE 66 0F 45 CA 58 E3 07 83 EF 02 E2 CD EB 02 31 C0 5E 5F 89 EC 5D C3 E8 1C FF FF FF AF AA AD EB AE AA EC A4 BA AF AE AA 8A C0 A7 B0 BC 9A BA A5 A5 BA AF B8 9D B8 F9 AE 9D AB B4 BC B6 B3 90 9A A8"
edi_unknownbytes = list(reversed(list(edi_unknownbytes.split())))
EAX = 0x0012FFF0
EBX = 0
ECX = 0x25

# loop itself
while ECX != 0:
    # status each loop
    print "%s: EBX & 3 - %08x" % (esi_input[0],(1 << (EBX & 3)) + 1)

    EDX = EBX & 3 # DX = BX & 3 equivalent
    AX = 0x01C7

    stack.append(int(hex((EAX & 0xFFFF0000) | AX)[2:],16))
    #eflags lower byte has carry set
    EAX = int(hex((0x001201FF & 0xFFFFFF00) | ord(esi_input.pop(0)))[2:],16) # move letter to AL
    #eflags pushed onto stack
    EAX = int(hex((EAX & 0xFFFFFF00) | ((EAX & 0x000000FF) ^ 0x000000C7)),16) # high-order bytes ORed with XORed low-order bytes. Stack always ends in C7 do to earlier move so that is kludged by hardcoding here
    EDX,ECX = ECX,EDX
    EAX = (EAX & 0xFFFF00FF) | ((EAX & 0x0000FF00) << ECX) # using ECX as CL
    # pop saved EFLAGS off stack back into EFLAGS
    EAX = (EAX & 0xFFFFFF00) | (((EAX & 0x0000FF00) >> 8) + (EAX & 0x000000FF) + 0x01) # hardcoded +1 but should be only if CF set
    #EAX = (EAX & 0xFFFFFF00) | AL # part of above line, preserving whole EAX
    EDX,ECX = ECX,EDX
    EDX = 0 # same as XORing with one's self
    EAX = EAX & 0x0FF
    EBX = EBX + EAX # using extended register should have no side effect
    if int(edi_unknownbytes.pop(0),16) != EAX: # makes up for not implementing NE flag in EFLAGS
        ECX = EDX
    EAX = stack.pop()
    if ECX == 0:
        print "Leaving loop premature, EAX will be set to zero"
        break
    # suppose to sub 2 from edi now but the pop earlier makes up for that
    ECX = ECX - 1
```

If you actually decide to try running this you will see that miraculously it works. It should be featured on Ripley's Believe It Or Not. If you enter the correct letter the loop continues. If you enter the wrong letter it lets you see the letter it failed on and exits. The output given no arguments is as follows:

```bash
$ ./algorithm.py
a: EBX & 3 - 00000002
_: EBX & 3 - 00000002
L: EBX & 3 - 00000005
i: EBX & 3 - 00000005
t: EBX & 3 - 00000003
u: EBX & 3 - 00000009
Leaving loop premature, EAX will be set to zero
```

From this point onward you can just brute force it one letter at a time using your favorite brute forcing tool (John The Ripper anyone...).

However, I felt unsatisfied because I discovered a working solution but I did not thoroughly understand the problem; and by the problem I mean the algorithm. I felt like I could do better! I kept printing out registers values at various points in algorithm.py as well as checking my script against breakpoints in Ollydbg. Like a math problem I began to gradually simplify one piece at a time, removing parts that were superfluous to the end goal. Also, I noticed the number subtracted from the current hex value was always changing; I began calling that the "Variance Number". Ultimately, my gradual simplifications resulted in the script below.

```python
#!/usr/bin/python

extracted = "AF AA AD EB AE AA EC A4 BA AF AE AA 8A C0 A7 B0 BC 9A BA A5 A5 BA AF B8 9D B8 F9 AE 9D AB B4 BC B6 B3 90 9A A8"
extracted = ' '.join(list(reversed(list(extracted.split()))))

def variance_number(prevPosHex):
    return ((1 << (prevPosHex & 3)) + 1)

extractedList = extracted.split()
saved_ebx = 0
temp = []
for i,c in enumerate(extractedList):
    h = int(c, 16)
    saved_ebx = sum([int(x,16) for x in extractedList[:i]])
    print "%s,%d,%04x,%04x,%04x" % (c, variance_number(saved_ebx), saved_ebx, saved_ebx & 3, i)
    temp.append ((h - variance_number(saved_ebx)) ^ 0xC7)
print ''.join(map(chr,temp))
```

When I ran this version of the script it completely decrypted the email and presented me with a_Little_b1t_harder_plez@flare-on.com.

I wish I could provide a more detailed write up for this challenge but my experience was that I learned through experimenting and refining my experiments. I would not recommend this approach to anyone. But if nothing else it was a unique approach that took lots of imagination so I wanted to share. I'm sure others came with much better approaches.
