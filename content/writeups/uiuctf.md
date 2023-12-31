---
title: UIUCTF23
image: assets/writeups/deathwolf.jpeg
description: Writeups for challenges i solved in UIUctf23 
date: 2023-07-04
categories: [CTFwriteups]
tags: [pwn, vimjail]
author: trace
---

# UIUCTF23

![](https://i.imgur.com/XZhjv7X.jpg)
## Corny Kernel

Welcome to the "Corny Kernel" challenge, where you can use our custom driver to tinker with the Linux kernel at runtime!

To get started, establish a connection with the challenge server using the following command:

```bash
$ socat file:$(tty),raw,echo=0 tcp:corny-kernel.chal.uiuc.tf:1337
```

### Challenge Overview

In this challenge, you are provided with a C file named `pwnymodule.c`. Here is the content of `pwnymodule.c`:

```c
// SPDX-License-Identifier: GPL-2.0-only

#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

extern const char *flag1, *flag2;

static int __init pwny_init(void)
{
    pr_alert("%s\n", flag1);
    return 0;
}

static void __exit pwny_exit(void)
{
    pr_info("%s\n", flag2);
}

module_init(pwny_init);
module_exit(pwny_exit);

MODULE_AUTHOR("Nitya");
MODULE_DESCRIPTION("UIUCTF23");
MODULE_LICENSE("GPL");
MODULE_VERSION("0.1");
```

This code represents a basic Linux kernel module. When the module is loaded, it prints the first part of the flag:

```c
static int __init pwny_init(void)
{
    pr_alert("%s\n", flag1);
    return 0;
}
```

Similarly, when the module is unloaded, it prints the second part of the flag:

```c
static void __exit pwny_exit(void)
{
    pr_info("%s\n", flag2);
}
```

As part of the challenge, you need to establish a connection to the server. Inside the server, you will find a file named `pwnymodule.ko.gz`.

### Solution

To solve the challenge, follow these steps:

1. First, unzip the `pwnymodule.ko.gz` file using the `gunzip` command:

    ```bash
    gunzip pwnymodule.ko.gz
    ```

2. Next, load the module inside the server using the `insmod` command:

    ```bash
    insmod pwnymodule.ko
    ```

![](https://i.imgur.com/2F7kaR6.png)

	As you can see, the first part of the flag is displayed.

3. To retrieve the second part of the flag, unload the module using the `rmmod` command:

    ```bash
    rmmod pwnymodule.ko
    ```

4. Finally, print the kernel messages using the `dmesg` command to obtain the second part of the flag.

![](https://i.imgur.com/e5YLEza.png)


The final flag is revealed:

```
uiuctf{m4ster_k3rNE1_haCk3r}
```

----

## VimJail1

To participate in the VimJail1 challenge, you need to establish a connection using the following command:

```bash
socat file:$(tty),raw,echo=0 tcp:vimjail1.chal.uiuc.tf:1337
```

If you don't have the 'socat' tool installed, you may need to install it.

### Challenge Overview

The challenge involves entering 'insert mode' in Vim and escaping from it to obtain the flag. The files related to this challenge are provided in a zip file. Upon examination of the `entry.sh` file, we can see the following script:

```bash
#!/usr/bin/env sh

chmod -r /flag.txt

vim -R -M -Z -u /home/user/vimrc
```

This script launches Vim with the options `-R` (read-only mode), `-M` (modified option enabled), and `-Z` (restricted mode active). It also utilizes a custom vimrc file located at `/home/user/vimrc`. The contents of the vimrc file are as follows:

```vim
set nocompatible
set insertmode

inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\><c-n> nope
```

Let's break down the purpose of each section:

```vim
set nocompatible
set insertmode
```

These commands disable compatibility mode and set Vim to insert mode.

The next section:

```vim
inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\><c-n> nope
```

This section maps the key combinations `Ctrl + o`, `Ctrl + l`, `Ctrl + z`, and `Ctrl + \ -> Ctrl + n` to insert the word `nope` in insert mode. However, since we cannot insert any characters in insert mode, these shortcuts will be blocked by Vim. When random characters or the mentioned key combinations are pressed, Vim will not respond, as shown in the following preview:

![](https://i.imgur.com/sDcXlXh.png)

**Solution:**

Let's examine the `vimrc` file further:

```vim
inoremap <c-\><c-n> nope
```

According to this mapping, we should not be able to use `Ctrl + \ -> Ctrl + n`. However, it can be bypassed by pressing `Ctrl + \` twice, followed by `Ctrl + n`. By using this sequence, we gain the ability to input commands into Vim.

To obtain the flag, we can read the contents of any file using the `:e` command. The `:e` command allows us to edit a file, and the final payload will look like this:

```vim
:e flag.txt
```

Upon executing this command, we can see the flag:

![](https://i.imgur.com/Yjnd22D.png)

```
uiuctf{n0_3sc4p3_f0r_y0u_8613a322d0eb0628}
```

By executing the provided payload, we successfully retrieve the flag for the challenge.

------

## VimJail2

To participate in the VimJail2 challenge, you need to establish a connection using the following command:

```bash
socat file:$(tty),raw,echo=0 tcp:vimjail2.chal.uiuc.tf:1337
```

If you don't have the 'socat' tool installed, you may need to install it.

### Challenge Overview

The challenge provides a zip file containing several files related to the challenge. In this challenge, the goal is to escape from the insert mode in Vim and obtain the flag. Let's analyze the `entry.sh` file to understand the setup:

```bash
#!/usr/bin/env sh

vim -R -M -Z -u /home/user/vimrc -i /home/user/viminfo

cat /flag.txt
```

The script launches Vim with the options `-R` (read-only mode), `-M` (modified option enabled), and `-Z` (restricted mode active). It also uses a custom `vimrc` file located at `/home/user/vimrc` and specifies `/home/user/viminfo` as the Viminfo file. After executing Vim, it proceeds to print the contents of the `flag.txt` file. Now let's examine the `vimrc` file:

```vim
set nocompatible
set insertmode

inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\><c-n> nope

cnoremap a _
cnoremap b _
cnoremap c _
cnoremap d _
cnoremap e _
cnoremap f _
cnoremap g _
cnoremap h _
cnoremap i _
cnoremap j _
...
```

Let's break down each part:

```vim
set nocompatible
set insertmode
```

These commands disable compatibility mode and set Vim to insert mode.

The next section:

```vim
inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\><c-n> nope
```

This section maps the key combinations `Ctrl + o`, `Ctrl + l`, `Ctrl + z`, and `Ctrl + \ -> Ctrl + n` to insert the word `nope` in insert mode. However, since we cannot insert any characters in insert mode, these shortcuts will be blocked by Vim.

The final section:

```vim
cnoremap a _
cnoremap b _
cnoremap c _
cnoremap d _
cnoremap e _
cnoremap f _
cnoremap g _
cnoremap h _
cnoremap i _
cnoremap j _
...
```

This section converts any input starting with `:` in command-line mode to a series of underscores (`_`). For example, if you input `:abcde`, it will be converted to `:_____`. This further restricts our ability to input commands.

When attempting random characters or the mentioned key combinations, Vim will not respond, as shown in the following preview:

![](https://i.imgur.com/Sc7qpxE.png)

**Solution:**

Upon reviewing the `vimrc` file, pay close attention to this part:

```vim
inoremap <c-\><c-n> nope
```

According to this mapping, we shouldn't be able to use `Ctrl + \ -> Ctrl + n`. However, we can bypass this restriction by pressing `Ctrl + \` twice, followed by `Ctrl + n`. This sequence allows us to input commands into Vim.

![](https://i.imgur.com/PHkczDE.png)

Considering the limitations imposed by the `vimrc` file, we can only use the letter 'q'. Surprisingly, entering the `:q` command will lead us to the flag. But how does this work? Let's revisit the `entry.sh` file:

```bash
#!/usr/bin/env sh

vim -R -M -Z -u /home/user/vimrc -i /home/user/viminfo

cat /flag.txt
```

The last line of the file instructs the shell to print the flag. By using the `:q` command to quit Vim, the shell script will execute and display the flag for us.

![](https://i.imgur.com/deBFyLU.png)

```
uiuctf{<left><left><left><left>_c364201e0d86171b}
```

----

## VimJail2.5

The VimJail2.5 challenge has been updated to address unintended solves in the previous version. To participate, establish a connection using the following command:

```bash
socat file:$(tty),raw,echo=0 tcp:vimjail2-5.chal.uiuc.tf:1337
```

If you don't have the 'socat' tool installed, you may need to install it.

### Challenge Overview

The challenge provides a zip file containing several files related to the challenge. In this challenge, the objective is to escape from the insert mode in Vim and obtain the flag. Let's examine the `entry.sh` file:

```bash
#!/usr/bin/env sh

vim -R -M -Z -u /home/user/vimrc -i /home/user/viminfo

cat /flag.txt
```

This script launches Vim in read-only mode with the modified option enabled and restricted mode active. It uses a custom `vimrc` file located at `/home/user/vimrc` and specifies `/home/user/viminfo` as the Viminfo file. After executing Vim, it proceeds to print the contents of the `flag.txt` file. Let's also take a look at the `vimrc` file:

```vim
set nocompatible
set insertmode

inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\> nope

cnoremap a _
cnoremap b _
cnoremap c _
cnoremap d _
cnoremap e _
cnoremap f _
cnoremap g _
cnoremap h _
cnoremap i _
cnoremap j _
...
```

Let's break down each part:

```vim
set nocompatible
set insertmode
```

These commands disable compatibility mode and set Vim to insert mode.

The next section:

```vim
inoremap <c-o> nope
inoremap <c-l> nope
inoremap <c-z> nope
inoremap <c-\> nope
```

This section maps the key combinations `Ctrl + o`, `Ctrl + l`, `Ctrl + z`, and `Ctrl + \` in insert mode to insert the word `nope`. However, since we cannot insert any characters in insert mode, these shortcuts will be blocked by Vim.

The final section:

```vim
cnoremap a _
cnoremap b _
cnoremap c _
cnoremap d _
cnoremap e _
cnoremap f _
cnoremap g _
cnoremap h _
cnoremap i _
cnoremap j _
...
```

This section converts any input starting with `:` in command-line mode to a series of underscores (`_`). For example, if you input `:abcde`, it will be converted to `:_____`. This further restricts our ability to input commands.

When attempting random characters or the mentioned key combinations, Vim will not respond, as shown in the following preview:

![](https://i.imgur.com/Sc7qpxE.png)

### Solution

In the updated challenge, a new approach is required to escape the insert mode and execute commands in Vim. We can utilize the builtin functions by pressing `Ctrl + r` followed by `=` in insert mode. This allows us to call these functions to execute commands. To view the list of available builtin functions, you can visit the Vim website or access the `builtin.txt` file.

![](https://i.imgur.com/A458aMd.png)

Since the `vimrc` file limits the characters we can input, we can use the Tab key to trigger autocompletion in Vim. This autocompletion feature enables us to select and use any builtin function effectively.

![](https://i.imgur.com/6mwbcC5.png)

To execute a command, we need to use the `=Execute` command, which allows us to execute Vim commands through the builtin functions. However, due to the limitations, we can only use the letter 'q'. Therefore, the solution becomes to input the command `:q` to obtain the flag. Let's revisit the `entry.sh` file to understand why this works:

```bash
#!/usr/bin/env sh

vim -R -M -Z -u /home/user/vimrc -i /home/user/viminfo

cat /flag.txt
```

In the last line of the script, the shell prints the flag. When we use the `:q` command to quit Vim, it triggers the execution of the shell script, which subsequently displays the flag.

To accomplish this, we can use the following payload to exit Vim:

```vim
=execute(":q")
```

![](https://i.imgur.com/pVULfY4.png)

The final flag is displayed:

```
uiuctf{1_kn0w_h0w_7o_ex1t_v1m_7661892ec70e3550}
```


----
