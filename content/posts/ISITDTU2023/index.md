---
weight: 1
title: "ISITDTU Writeups"
date: 2023-10-16T02:05:00+07:00
lastmod: 2023-10-16T02:05:00+07:00
draft: false
author: "ItayB"
authorLink: "https://itaybel.github.io"
description: "Solutions to the rev challenges from the event."

tags: ["exe", "elf", "rev"]
categories: ["Writeups"]

lightgallery: true

toc:
  enable: true
---

## Overview

ISITDTU was a really nice CTF. I solved a lot of the rev chals and enjoyed a lot.
We finished 12th, and qualified for the final in Vietnam! I don't know yet wether I will come, but I am glad we did it.

Here you can read some of my solutions to the rev challenges from the event.
<!--more-->

## Re01
This was the first reversing challenge in the event. It was a simple crackme elf, which takes a flag as input and tells you if its correct:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/00c81471-b83f-451e-b2bb-02e104a94de2)


Lets put the binary in IDA and start with static analysis.

Firstly, we can see tons of number have been stored in an array in the stack, called `numbers` (I changed it to this name lol)

```c
numbers[0] = 17325;
numbers[1] = 19708;
numbers[2] = 21160;
numbers[3] = 23202;
numbers[4] = 25884;
numbers[5] = 18561;
numbers[6] = 20995;
numbers[7] = 22495;
numbers[8] = 24643;
numbers[9] = 27473;
numbers[10] = 18886;
numbers[11] = 21391;
numbers[12] = 22901;
numbers[13] = 25137;
numbers[14] = 28011;
numbers[15] = 17116;
numbers[16] = 19472;
numbers[17] = 20908;
numbers[18] = 22968;
numbers[19] = 25672;
numbers[20] = 8035;
numbers[21] = 9333;
numbers[22] = 10185;
numbers[23] = 11405;
numbers[24] = 13119;
```
Then the program will ask for our input, and will store it a string called `input`.

then it will check if its length is 25 and if not it will exit:

```c
std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(input, a2, a3);
  std::operator<<<std::char_traits<char>>(&std::cout, "Enter your flag : ");
  std::operator>><char>(&std::cin, input);
  if ( std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::length(input) != 25 )
  {
    std::operator<<<std::char_traits<char>>(&std::cout, "You are wrong :(\n");
    exit(1337);
  }
```

Then this will run:
```
 for ( i = 0; i <= 4; ++i )
  {
    for ( j = 0; j <= 4; ++j )
      input_cpy_1[5 * i + j] = *(char *)std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::operator[](
                                          input,
                                          5 * i + j);
  }
  ```

which basiclly copies the characters of our input string to `input_cpy_1`.

Now, there is this important piece of code:

```c
  do_something((__int64)v8);
  sub_2936((__int64)v9, 0x539uLL);
  sub_3766(v8, v9);
  sub_345E(v9);
  for ( k = 0; k <= 4; ++k )
  {
    for ( m = 0; m <= 4; ++m )
      calc[5 * k + m] = *(_QWORD *)sub_37A2(v8, 5 * k + m);
  }

  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(input_cpy, input);
  check(5, 5u, (__int64)input_cpy_1, 5u, 5, (__int64)calc, (__int64)numbers, (__int64)input_cpy);

__int64 __fastcall do_something(__int64 a1, unsigned __int64 a2)
{
  unsigned __int64 i; // [rsp+10h] [rbp-30h] BYREF
  unsigned __int64 j; // [rsp+18h] [rbp-28h]
  void *s; // [rsp+20h] [rbp-20h]
  unsigned __int64 v6; // [rsp+28h] [rbp-18h]

  v6 = __readfsqword(0x28u);
  sub_32EE(a1);
  s = (void *)operator new[]((a2 >> 3) + 1);
  memset(s, 255, (a2 >> 3) + 1);
  for ( i = 2LL; a2 >= i; ++i )
  {
    if ( ((*((char *)s + (i >> 3)) >> (i & 7)) & 1) != 0 )
    {
      sub_34A6(a1, &i);

      for ( j = 2 * i; j <= a2; j += i )
        *((_BYTE *)s + (j >> 3)) &= ~(1 << (j & 7));
    }
  }
  if ( s )
    operator delete[](s);
  return a1;
}
```

but it seems a bit complex, and it doesn't do anything with our input, so we can just use gdb for dynamic analysis and see what the array `calc` looks like.

I have put a breakpoint on the call to `check` , and saw that the array is basically just prime numbers:

```
pwndbg> x/20wx $r9
0x7fffffffdde0:	0x00000002	0x00000003	0x00000005	0x00000007
0x7fffffffddf0:	0x0000000b	0x0000000d	0x00000011	0x00000013
0x7fffffffde00:	0x00000017	0x0000001d	0x0000001f	0x00000025
0x7fffffffde10:	0x00000029	0x0000002b	0x0000002f	0x00000035
0x7fffffffde20:	0x0000003b	0x0000003d	0x00000043	0x00000047
```
Now lets look at the `check` function. its the function responsible for checking our input and tell us wether we are correct or not.

Here is it:

```c
unsigned __int64 __fastcall check(
        int five,
        unsigned int FIVE,
        __int64 input_crambled,
        unsigned int ANOTHERFIVE,
        int five2,
        __int64 primes,
        __int64 constnums,
        __int64 input)
{
  __int64 v8; // rax
  __int64 v9; // rax
  char v12; // [rsp+37h] [rbp-E9h] BYREF
  int i; // [rsp+38h] [rbp-E8h]
  int j; // [rsp+3Ch] [rbp-E4h]
  char v15[32]; // [rsp+40h] [rbp-E0h] BYREF
  char v16[32]; // [rsp+60h] [rbp-C0h] BYREF
  char v17[32]; // [rsp+80h] [rbp-A0h] BYREF
  int output[26]; // [rsp+A0h] [rbp-80h] BYREF
  unsigned __int64 set_func; // [rsp+108h] [rbp-18h]

  set_func = __readfsqword(0x28u);
  if ( ANOTHERFIVE == FIVE )
  {
    memset(output, 0, 100);
    do_something_recursive(five, FIVE, input_crambled, ANOTHERFIVE, five2, primes, (__int64)output);
    std::allocator<char>::allocator(&v12);
    sub_3526((__int64)v15, (__int64)&unk_501E, (__int64)&v12);
    std::allocator<char>::~allocator(&v12);
    for ( i = 0; i < five; ++i )
    {
      for ( j = 0; j < five2; ++j )
      {
        if ( output[5 * i + j] != *(_DWORD *)(constnums + 20LL * i + 4LL * j) )
        {
          std::operator<<<std::char_traits<char>>(&std::cout, "You are wrong :(\n");
          exit(1337);
        }
      }
    }
```
There is this function `do_something_recursive`, which takes in our input, and the primes, and write to an array called `output`. then, the elements of output will be checked against the constnums array created in `main`.

Lets take a look at this `do_something_recursive`:

```c
__int64 __fastcall do_something_recursive(
        int FIVE,
        int FIVE3,
        __int64 input,
        int a4,
        int FIVE2,
        __int64 primes,
        __int64 a7)
{
  __int64 result; // rax

  result = (unsigned int)index;
  if ( FIVE > index )
  {
    if ( FIVE2 > index1 )
    {
      if ( FIVE3 > index2 )
      {
        *(_DWORD *)(a7 + 20LL * index + 4LL * index1) += *(_DWORD *)(primes + 20LL * index2 + 4LL * index1)
                                                       * *(_DWORD *)(input + 20LL * index + 4LL * index2);
        ++index2;
        do_something_recursive(FIVE, FIVE3, input, a4, FIVE2, primes, a7);
      }
      index2 = 0;
      ++index1;
      do_something_recursive(FIVE, FIVE3, input, a4, FIVE2, primes, a7);
    }
    index1 = 0;
    ++index;
    return do_something_recursive(FIVE, FIVE3, input, a4, FIVE2, primes, a7);
  }
  return result;
}
```

We can see that it'll have 3 indexes, and go through the `input` and `primes` array, and will add multiples of them to the number corrosponding number in the `output` array.

Its hard to follow exactly whats going on just with static analysis, so lets quickly start with dynamic. lets use gdb and put a breakpoint right when it writes to output;

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/2b1eda02-e3e5-46a1-8b18-e1bd7a3e1cde)

Now I have given the program 20 d's , because d's ascii number is 100 so it will be easier to see whats going on.

First iteration, we can see that 0xc8=200 is being written to the first element in the array:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/8078d55e-b109-4dbc-a393-84ad4cb9cc95)

Ok, we'll keep that in mind. now enter `c` and see what it will be added with:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/1058a446-7551-45ce-a25d-389f78943e3c)

Ok, so it wrote 0x5dc. `0x5dc - 0xc8 = 1300`, so it added 1300 to it.

Next:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/399b8c49-d5f1-4520-85bb-d2ad72405992)

Now RCX is 0x11f8. `0x11f8 - 0x5dc = 3100`. Interseting. Next:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/3b38cb3e-0187-40df-879b-22b3c2ddb62f)

RCX is 0x26ac, `0x26ac - 0x11f8 = 5300` , And lastly:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/ef57b8d5-4cfb-4417-b35f-f8415e8c9886)

so RCX is 0x4330, `0x4330 - 0x26ac = 7300`.


So we can understand that it added 200,1300,3100,5300,7300. we saw that the `do_something_recursive` function takes chars from our inputs, and multiplies it with some primes. since our input is just d's, we know that our input has been multiplied with the primes 2, 13, 31, 53, 73!

But why those specific primes you ask? those primes are primes in our array with jumps of five!

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/78eb1619-febf-4d4c-b86d-569aece18c49)


In order to see exactly whats going on, we'll enter this as our input `ddddd22222AAAAAAAAAAAAAAA`, because `ord("d") = 100, ord("2") = 50` and its easy to work with these numbers.

In the first 5 iterations, it will have the same behaviour as the previous example, so we can know that `output[0] = 0x4330`

Now, we'll hit `c` and see whats going on:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/63c0407c-3fc3-410e-8bc1-23d71a648d47)

Oh, so now RCX is `0x12c = 300`. lets continue:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/5e348985-29ee-4b96-aa00-e161ec071eb9)

RCX = 0x7d0, added `0x7d0 - 0x12c = 1700`

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/052b6c7f-b473-4c56-b227-329751200ca3)

RCX = 0x1644, added `0x1644 - 0x7d0 = 3700`

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/58dfa231-ef01-4318-a951-4b564cf2bc14)

RCX = 0x2d50, added `0x2d50 - 0x1644 = 5900`, and lastly:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/ddaedf68-40be-4240-8ee6-f827b6ff03da)

RCX = 0x4c2c , added `0x4c2c - 0x2d50 = 7900`.

So our primes now are 3, 17, 37, 59, 79. the indexes follows the same pattern!

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/70c70504-da92-4ce8-97b3-3ef8ec53fd7a)

The only differnece is that it starts with 1 now. 

We we'll see that the same thing happens 5 times, each time it starts with a different prime.

To show that in a mathematical way:

Lets say that our target numbers is an array called `target`, and that our string is $a_1, a_2, a_3, a_4, a_5$

The code will check if the following system of equations hold.

<br>

$$2a_1 + 13a_2 + 31a_3 + 53a_4 + 73a_5 = target_1$$

$$3a_1 + 17a_2 + 37a_3 + 59a_4 + 79a_5 = target_2$$

$$5a_1 + 19a_2 + 41a_3 + 61a_4 + 83a_5 = target_3$$

$$7a_1 + 23a_2 + 43a_3 + 67a_4 + 89a_5 = target_4$$

$$11a_1 + 29a_2 + 47a_3 + 71a_4 + 97a_5 = target_5$$

<br>


Recall such system can also be written as a coefficient matrix, namely

<br>

$$\begin{pmatrix}
2 & 13 & 31 & 53 & 73\\\
3 & 17 & 37 & 59 & 79\\\
5 & 18 & 41 & 61 & 83\\\
7 & 23 & 43 & 67 & 89\\\
11 & 29 & 47 & 71 & 97
\end{pmatrix} \times 
\begin{pmatrix}
x_1\\
x_2\\
x_3\\
x_4\\
x_5
\end{pmatrix}
=.
\begin{pmatrix}
target1\\\
target2\\\
target3\\\ 
target4\\\
target5 
\end{pmatrix}$$

<br>

So we  can solve this equation, so get $a_1, a_2, a_3, a_4, a_5$ which will be the corrosponding five characters of the flag.

Here is my solve script, which used `numpy.linalg.solve` to solve the matrix:

```py
from pwn import *
import numpy as np


primes = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]


target1 = [17325, 19708, 21160, 23202, 25884]
target2 = [18561, 20995, 22495, 24643, 27473]
target3 = [18886, 21391, 22901, 25137, 28011]
target4 = [17116, 19472, 20908, 22968, 25672]
target5 = [8035, 9333, 10185, 11405, 13119]

targets = [target1, target2, target3, target4, target5]
flag = ""

for target in targets:
	list_of_mekadmin = []
	for idx in range(len(target)):
		curr = []
		for j in range(5):
			curr.append(primes[5*j + idx])
		list_of_mekadmin.append(curr)

	A = np.array(list_of_mekadmin)
	sol = np.array(target)

	for i in np.linalg.solve(A, sol):
		print(round(i))
		flag += chr(round(i))

print(flag)
```

## Dot

Dot was the second reversing challenge in the event.

It was a basic `dot.exe` binary.

When we try to run it, we get an error:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/738f4ead-779b-4958-849e-e39daec1c621)

Lets put the binary in IDA and start reversing it.

Firstly, lets locate where the check is made:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/e8d2afdd-9579-4d0c-a28f-78a2410ceab8)

Ida's decompliation is kinda bad with exe, so looking at assembly is better.

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/e78d87e8-e098-4ef3-8524-9197e2379477)

We can see that this morse code is getting compared with some string. we can guess that this string is the output of some kind of transformation that has been done on our input.

When we look for `xrefs`, we can see that the output of the `main_encodeToMorse` function is saved in this string:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/eed9b324-0b83-4edc-aa04-d4f79b85d9d8)

So we can understand that the function `main_encodeToMorse` is called (probably with the hostname), and then its output is compared with this hardcoded string: `..... ... --- .. --.. ----- --... .---- --. ----- --.. -.-- --...`.
At first I tried to use an online decoder of morse code, to check if thats that easy, but it wasn't the flag:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/84886797-1235-4b29-ad80-281e7e4c93e1)

Then I dug up the code a bit more. so this piece of code:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/f970d283-7569-4409-9fcb-d9467991b259)

Here are those global variables:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/9ce9fb5b-360c-4c4f-8ec8-a2c37c18b335)


![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/212096f6-9065-41be-94fa-130fdc92f51e)

These 2 arrays are of the same size. now, it makes sense that the `morse_codes_dic` variable is the encoded morse code of the corrosponding character in the `characters_dic` array.

With that in mind, I wrote a small python script that decodes the `..... ... --- .. --.. ----- --... .---- --. ----- --.. -.-- --...`. string:

```
morse = ["--..",".--","-..",".---",".-..","_","..---",".--.",".....","---","....","-.--","-...","-....","--.-","-..-","----.","-----","..-.","-.-",".","--","-","--...",".-.",".----","-.","...-","...--",".-","..-","....-","...","--.","..","---..","-.-."]

alpha = ['U', 'S', '2', 'H', 'J', 'L', 'Y', 'G', 'C', 'M', '9', 'W', 'A', 'I', 'F', 'V', '4', 'T', '8', '5', '1', 'E', 'N', '3', 'Z', 'R', ' ', 'Q', 'X', '7', 'K', 'O', '0', 'D', 'P', '6', 'B']


out = '..... ... --- .. --.. ----- --... .---- --. ----- --.. -.-- --...'.split(' ')

for i in out:
	a = alpha[morse.index(i)]
	print(a, end="")
print()
```

This code printed `C0MPUT3RDTUW3` which indeed looks like the beginning of the flag.

I got back to IDA and saw that IDA was tricking us, and the string doesn't end there:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/ce01f2d0-2e8f-4f29-ac5a-25612395ede4)

After adding these last characters we get the flag:

```py

morse = ["--..",".--","-..",".---",".-..","_","..---",".--.",".....","---","....","-.--","-...","-....","--.-","-..-","----.","-----","..-.","-.-",".","--","-","--...",".-.",".----","-.","...-","...--",".-","..-","....-","...","--.","..","---..","-.-."]

alpha = ['U', 'S', '2', 'H', 'J', 'L', 'Y', 'G', 'C', 'M', '9', 'W', 'A', 'I', 'F', 'V', '4', 'T', '8', '5', '1', 'E', 'N', '3', 'Z', 'R', ' ', 'Q', 'X', '7', 'K', 'O', '0', 'D', 'P', '6', 'B']


out = "..... ... --- .. --.. ----- --... .---- --. ----- --.. -.-- --... _ ..... ... --- --...".split(" ")

for i in out:
	a = alpha[morse.index(i)]
	print(a, end="")
print()

#C0MPUT3RDTUW3LC0M3
```

## Sanity Check

The challenge consisted of another go binary, but luckily now its an ELF.
it nicecly asked for the flag, and it will check if its correct.

After putting the binary in IDA, we can see this while loop checking our input:

```c
  while ( i < 0x23 )
  {
    index = i;
    v35 = v4;
    v40 = v5;
    v6 = fibbon();
    if ( v34 <= index )
      runtime_panicIndex();
    if ( !*(_QWORD *)(v39 + 16 * index + 8) )
      runtime_panicIndex();
    if ( v6 == (v37[index] ^ **(unsigned __int8 **)(v39 + 16 * index)) ) (1)
    {
      main_calc();
      v33 = runtime_intstring(v15, v23);
      v7 = v40;
      v34 = runtime_concatstring2(v16, v24, v29, v32, v33);
    }
    else
    {
      v46 = v1;
      runtime_convT64(v15);
      *(_QWORD *)&v46 = &unk_489860;
      *((_QWORD *)&v46 + 1) = v9;
      fmt_Fprintln(v17, v23, v29);
      v44 = &unk_489EA0;
      v45 = &WRONG;
      fmt_Fprintln(v18, v25, v30);
      v32 = os_Exit(v19);
      v7 = v35;
      v8 = v40;
    }
    i = index + 1;
    v4 = v7;
    v5 = v8;
  }
```
Firstly, lets look at the `fibbon` number, because its output gets compared with the xor:

```c
__int64 __usercall fibbon@<rax>(int index)
{
  __int64 result; // rax
  __int64 v2; // r14
  char *v3; // [rsp-8h] [rbp-18h]
  char *v4; // [rsp-8h] [rbp-18h]
  __int64 v5; // [rsp+0h] [rbp-10h]
  void *retaddr; // [rsp+10h] [rbp+0h] BYREF

  if ( (unsigned __int64)&retaddr <= *(_QWORD *)(v2 + 16) )
    runtime_morestack_noctxt_abi0();
  if ( result > 1 )
  {
    v5 = fibbon(v3);
    return v5 + fibbon(v4);
  }
  return result;
}
```
It gets called with the current index, just like that:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/84ee9236-8a29-4995-a256-e0491dbfaffa)

We can see that it basically calculates the fibbonachi number with the corrosponding index.

Now let's use gdb and put a breakpoint on this compare at (1):

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/ea0d0fc5-2a83-4c84-9f16-a0f8ca0fa959)

I have just entered 0x23 a's, because thats the condition of the while loop (it accounts for 0x23 characters)

We can see that the program will xor the next character, in our case, 'a', which is 0x61, with this arbitrary 0x54 which is in the stack.

A quick check shows that there are tons of hardcoded values like 0x54 in the stack:

![image](https://github.com/itaybel/Weekly-CTF/assets/56035342/17d9a168-1fa8-472e-895a-d274008891e0)

After the xor, it will compare it with our character in the input.

So basiclly, the program will take the i'th character of our input, xor it with the i'th fibbonachi number, and check if its equals to the corrosponding number in the hardcoded array.

After exporting all these numbers, and exporting the first 100 fibbonachi numbers, I wrote this script which finds the flag:

```py
nums = [
0x54,
0x69,
0x68,
0x71,
0x5c,
0x4c,
0x7b,
0x52,
0x42,
0x4a,
0x52,
0x2b,
0xf5,
0xb6,
0x0000000000000120	,0x000000000000020d
,0x00000000000003ae,	0x000000000000064f
,0x0000000000000a47	,0x0000000000001007
,	0x0000000000001a28,	0x0000000000002a9d
,	0x0000000000004565	,0x0000000000006f9e
,	0x000000000000b555,	0x0000000000012563
,	0x000000000001da5f	,0x000000000002ff27
,	0x000000000004d90a	,0x000000000007d8ea
,	0x00000000000cb26a,	0x0000000000148ab8
,	0x0000000000213d62	,0x000000000035c78b
,	0x0000000000570489]

flag = ""
idx = 0

fibbon = [...] #its big lol

for i in nums:
	flag += chr(fibbon[idx] ^ i)
	print(flag)
	idx += 1
print(flag)
```


## Appendix

Really nice rev challenges. I especially really likes `re01` since it contained math which I don't see that often.

If you have any question regarding the above solution, you can DM me via my [Twitter](https://x.com/itaybel) or my `Discord` (itaybel).
