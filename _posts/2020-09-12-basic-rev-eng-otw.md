---
layout: post
title:  "Basic reverse engineering at over the wire"
date:  2022-05-23 12:00:00 -500
tags: [reverse engineering, pentest]
categories: [reverse engineering, pentest]
---

Today I was getting back to do some CTF on the platform Over The Wire.
You can find there a lot of pentesting challenges.
Either you are interested in web pentesting, linux server, cryptography, you can take a look there
and see if they have what fits you.

So, I was getting challenged by the one of the games (name and level modified to avoid to many spoilers),
in which you need to use some binaries that you find on the server to get the password to the next level.
From the beginning, the `ltrace` linux command is your friend in order to see what string is expected
as an argument or how the functions are working inside the binary.
For one of the level, you just face a binary that gives you this kind on output when you execute it:
```bash
otw@otw:~$ ./otw
usage: ./otw 4 <digit code>;
```
From this, one can easily see that a 4-digit code has to be given as an argument. Usually, a small loop would do it to brute force the binary. However, as I said, from the beginning you have to use ltrace and figure out what is happening inside the binary. This coupled to my new interest in reverse engineering made me totally overlook this option and I started dealing with the binary.

In this case ltrace and strace were of no help. Then, I started diving into `gdb`. I don't know much about `gdb`. This [video](https://invidio.us/watch?v=VroEiMOJPm8) gave me a quick overview of `gdb` that also helped me to start seeing what reverse engineering could be.

Let's then get started with gdb.
Launch `gdb` and your executable

```bash
otw@otw:~$ gdb otw
GNU gdb (Debian 7.12-6) 7.12.0.20161007-git
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
.
Find the GDB manual and other documentation resources online at:
.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from leviathan6...(no debugging symbols found)...done.
(gdb)
```
You get a prompt inside `gdb`. By entering `help`, you can get a list of command that you can enter in gdb.
Let's enter first the `disassembly main` command to get an overview of the instruction inside the main function.

```bash
Dump of assembler code for function main:
   0x0804853b &lt;+0>:	lea    0x4(%esp),%ecx
   0x0804853f &lt;+4>:	and    $0xfffffff0,%esp
   0x08048542 &lt;+7>:	pushl  -0x4(%ecx)
   0x08048545 &lt;+10>:	push   %ebp
   0x08048546 &lt;+11>:	mov    %esp,%ebp
   0x08048548 &lt;+13>:	push   %ebx
   0x08048549 &lt;+14>:	push   %ecx
   0x0804854a &lt;+15>:	sub    $0x10,%esp
   0x0804854d &lt;+18>:	mov    %ecx,%eax
   0x0804854f &lt;+20>:	movl   $0x1bd3,-0xc(%ebp)
   0x08048556 &lt;+27>:	cmpl   $0x2,(%eax)
   0x08048559 &lt;+30>:	je     0x804857b &lt;main+64>
   0x0804855b &lt;+32>:	mov    0x4(%eax),%eax
   0x0804855e &lt;+35>:	mov    (%eax),%eax
   0x08048560 &lt;+37>:	sub    $0x8,%esp
   0x08048563 &lt;+40>:	push   %eax
   0x08048564 &lt;+41>:	push   $0x8048660
   0x08048569 &lt;+46>:	call   0x80483b0 &lt;printf@plt>
   0x0804856e &lt;+51>:	add    $0x10,%esp
   0x08048571 &lt;+54>:	sub    $0xc,%esp
   0x08048574 &lt;+57>:	push   $0xffffffff
   0x08048576 &lt;+59>:	call   0x80483f0 &lt;exit@plt>
   0x0804857b &lt;+64>:	mov    0x4(%eax),%eax
   0x0804857e &lt;+67>:	add    $0x4,%eax
   0x08048581 &lt;+70>:	mov    (%eax),%eax
   0x08048583 &lt;+72>:	sub    $0xc,%esp
   0x08048586 &lt;+75>:	push   %eax
   0x08048587 &lt;+76>:	call   0x8048420 &lt;atoi@plt>
   0x0804858c &lt;+81>:	add    $0x10,%esp
   0x0804858f &lt;+84>:	cmp    -0xc(%ebp),%eax
   0x08048592 &lt;+87>:	jne    0x80485bf &lt;main+132>
   0x08048594 &lt;+89>:	call   0x80483c0 &lt;geteuid@plt>
   0x08048599 &lt;+94>:	mov    %eax,%ebx
   0x0804859b &lt;+96>:	call   0x80483c0 &lt;geteuid@plt>
   0x080485a0 &lt;+101>:	sub    $0x8,%esp
   0x080485a3 &lt;+104>:	push   %ebx
   0x080485a4 &lt;+105>:	push   %eax
   0x080485a5 &lt;+106>:	call   0x8048400 &lt;setreuid@plt>
   0x080485aa &lt;+111>:	add    $0x10,%esp
   0x080485ad &lt;+114>:	sub    $0xc,%esp
   0x080485b0 &lt;+117>:	push   $0x804867a
   0x080485b5 &lt;+122>:	call   0x80483e0 &lt;system@plt>
   0x080485ba &lt;+127>:	add    $0x10,%esp
   0x080485bd &lt;+130>:	jmp    0x80485cf &lt;main+148>
   0x080485bf &lt;+132>:	sub    $0xc,%esp
   0x080485c2 &lt;+135>:	push   $0x8048682
   0x080485c7 &lt;+140>:	call   0x80483d0 &lt;puts@plt>
   0x080485cc &lt;+145>:	add    $0x10,%esp
   0x080485cf &lt;+148>:	mov    $0x0,%eax
```
As you can see, it is not really straightforward to read it. To make this more understandable, we can convert this into the intel format, with the following command

```bash
(gdb) set disassenbly-flavor intel
```
And here is the result
```bash
Dump of assembler code for function main:
0x0804853b &lt;+0>: lea ecx,[esp+0x4]
0x0804853f &lt;+4>: and esp,0xfffffff0
0x08048542 &lt;+7>: push DWORD PTR [ecx-0x4]
0x08048545 &lt;+10>: push ebp
0x08048546 &lt;+11>: mov ebp,esp
0x08048548 &lt;+13>: push ebx
0x08048549 &lt;+14>: push ecx
0x0804854a &lt;+15>: sub esp,0x10
0x0804854d &lt;+18>: mov eax,ecx
0x0804854f &lt;+20>: mov DWORD PTR [ebp-0xc],0x1bd3
0x08048556 &lt;+27>: cmp DWORD PTR [eax],0x2
0x08048559 &lt;+30>: je 0x804857b
0x0804855b &lt;+32>: mov eax,DWORD PTR [eax+0x4]
0x0804855e &lt;+35>: mov eax,DWORD PTR [eax]
0x08048560 &lt;+37>: sub esp,0x8
0x08048563 &lt;+40>: push eax
0x08048564 &lt;+41>: push 0x8048660
0x08048569 &lt;+46>: call 0x80483b0
0x0804856e &lt;+51>: add esp,0x10
0x08048571 &lt;+54>: sub esp,0xc
0x08048574 &lt;+57>: push 0xffffffff
0x08048576 &lt;+59>: call 0x80483f0
0x0804857b &lt;+64>: mov eax,DWORD PTR [eax+0x4]
0x0804857e &lt;+67>: add eax,0x4
0x08048581 &lt;+70>: mov eax,DWORD PTR [eax]
0x08048583 &lt;+72>: sub esp,0xc
0x08048586 &lt;+75>: push eax
0x08048587 &lt;+76>: call 0x8048420
0x0804858c &lt;+81>: add esp,0x10
0x0804858f &lt;+84>: cmp eax,DWORD PTR [ebp-0xc]
0x08048592 &lt;+87>: jne 0x80485bf
0x08048594 &lt;+89>: call 0x80483c0
0x08048599 &lt;+94>: mov ebx,eax
0x0804859b &lt;+96>: call 0x80483c0
0x080485a0 &lt;+101>: sub esp,0x8
0x080485a3 &lt;+104>: push ebx
0x080485a4 &lt;+105>: push eax
0x080485a5 &lt;+106>: call 0x8048400
0x080485aa &lt;+111>: add esp,0x10
0x080485ad &lt;+114>: sub esp,0xc
0x080485b0 &lt;+117>: push 0x804867a
0x080485b5 &lt;+122>: call 0x80483e0
0x080485ba &lt;+127>: add esp,0x10
0x080485bd &lt;+130>: jmp 0x80485cf
0x080485bf &lt;+132>: sub esp,0xc
0x080485c2 &lt;+135>: push 0x8048682
0x080485c7 &lt;+140>: call 0x80483d0
0x080485cc &lt;+145>: add esp,0x10
0x080485cf &lt;+148>: mov eax,0x0
0x080485d4 &lt;+153>: lea esp,[ebp-0x8]
---Type to continue, or q to quit---
```

By taking a look at the first lines, we can spot this comparison:
```bash
0x08048556 &lt;+27>:	cmp    DWORD PTR [eax],0x2
```

Here the program is trying to compare number 2 (0x2) to the value stored where the register eax is pointing to. The DWORD keyword is used to indicate the size of the variable. The program here is checking if the number of given argument is correct. This instruction is similar to the if statement in a programing language such as python. After the comparison, the program goes to the next instruction:

```bash
0x08048559 &lt;+30>:	je     0x804857b &lt;main+64>
```
This instruction will set the behavior depending on the result of the comparison done on the previous line. In the case the comparison is true the je (jump equal) instruction will send the program to the instruction located at the offset of 64 from the main function (<main+64>). If the comparison returns a false, then the program continues to the next intruction (0x0804855b <+32>: mov eax,DWORD PTR [eax+0x4]).

Let's assume that our comparison is true, and that we have given the good number of argument. The program will then jump to the instruction located at <main+64>:0

```bash
0x0804857b &lt;+64>: mov eax,DWORD PTR [eax+0x4]
```
From there, the program will modifiy variables and push value into some registers. I will skip most of the instruction and go directly to what happens at <main+84>:
```bash
0x0804858f &lt;+84>:	cmp    eax,DWORD PTR [ebp-0xc]
```
It seems that, this time, the program is trying to compare the value in the register eax, which is were the argument (the 4 digits, remember) is stored, to the value in the register [ebp-0xc] is pointing at. So, that must be the place inside the code where our argument is checked, and if it is the good one, maybe we get the password for the next level.
Nice, but how do we get the value used to compare to our argument. Well, if you go a bit up, when the program starts, you will see this line:
```bash
0x0804854f &lt;+20>:	mov    DWORD PTR [ebp-0xc],0x1bd3
```

Here, the value the register [ebp-0xc] is pointing at is initialized with the value 0x1bd3. It means that the argument we have entered will be compared to this value. You can easily convert this value into base 10, and it gives you 7123 which was the key to unlock the next level.

That was my first challenge using reverse engineering, and it was interesting to see how to understand the binary you are dealing with instead of just brute-forcing it. Understanding the instruction given by the program is of great help in order to understand what it does, and can give you huge power in order to attack the executable or the server running it.
Assembly instructions appear to follow a certain logic, but it requires some time to get use to them. This [x86 Assembly Guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html) can be used as a cheat sheet when starting to have all the main instructions at hand.
