---
title: IrisCTF24
description: Writeups for challenges i solved in IrisCTF24 
date: 2024-01-08
categories: [CTFwriteups]
tags: [reversing, forensics, web]
author: trace
---
# IrisCTF24

## What's My Password?[Web]

![enter image description here](https://i.imgur.com/UNhK4kp.png)

We get a resource file and the challenge link,
Extracting the tar file gives us following
![enter image description here](https://i.imgur.com/9IzYQBx.png) 
we can see the Data Manipulation Language (DML) and Data Control Language (DCL) queries of the db in *setup.sql*
> setup.sql
```sql
CREATE DATABASE uwu;
use uwu;

CREATE TABLE IF NOT EXISTS users ( username text, password text );
INSERT INTO users ( username, password ) VALUES ( "root", "IamAvEryC0olRootUsr");
INSERT INTO users ( username, password ) VALUES ( "skat", "fakeflg{fake_flag}");
INSERT INTO users ( username, password ) VALUES ( "coded", "ilovegolang42");

CREATE USER 'readonly_user'@'%' IDENTIFIED BY 'password';
GRANT SELECT ON uwu.users TO 'readonly_user'@'%';
FLUSH PRIVILEGES;
```
column contains the value "password," and the user in question, who has unfortunately forgotten their password, is identified as "skat."

Upon examining the backend code located in ./recurso/whats-my-password/src/main.go, it becomes apparent that the provided code handles user input sanitation but overlooks the password sanitization process.

Here is an excerpt from the code:
>main.go
```go
matched, err := regexp.MatchString(UsernameRegex, input.Username)
if err != nil {
    w.WriteHeader(http.StatusInternalServerError)
    return
}

if matched {
    w.WriteHeader(http.StatusBadRequest)
    w.Write([]byte("Username can only contain lowercase letters and numbers."))
    return
}

qstring := fmt.Sprintf("SELECT * FROM users WHERE username = \"%s\" AND password = \"%s\"", input.Username, input.Password)

query, err := DB.Query(qstring)` 
```

It's worth noting that the code checks and enforces a specific format for the username, restricting it to lowercase letters and numbers. However, the password input does not undergo a similar sanitization process.
Therefore, the query is vulnerable to SQL Injection.

The password to enter will be`" OR username = 'skat'" `

![enter image description here](https://i.imgur.com/F344nSk.png)
and we get the flag,
![enter image description here](https://i.imgur.com/LShSp2X.png)
> Flag: ``irisctf{my_p422W0RD_1S_SQl1}``
---

## Rune? What's that? (Reversing)

![enter image description here](https://i.imgur.com/gqUwgID.png)
we get a resource file
which upon extracting gives us the following content
![enter image description here](https://i.imgur.com/a76OtS2.png)
checking  the `main.go` code,
>main.go
```go
package main

import (
	"fmt"
	"os"
	"strings"
)

var flag = "irisctf{this_is_not_the_real_flag}"

func init() {
	runed := []string{}
	z := rune(0)

	for _, v := range flag {
		runed = append(runed, string(v+z))
		z = v
	}

	flag = strings.Join(runed, "")
}

func main() {
	file, err := os.OpenFile("the", os.O_RDWR|os.O_CREATE, 0644)
	if err != nil {
		fmt.Println(err)
		return
	}

	defer file.Close()
	if _, err := file.Write([]byte(flag)); err != nil {
		fmt.Println(err)
		return
	}
}
```
Following is the code that i documented and will explain below that
```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// The original flag is declared as a package-level variable.
var flag = "irisctf{this_is_not_the_real_flag}"

// The init() function is executed before the main() function.
func init() {
	runed := []string{}
	z := rune(0)

	// Iterate over each character in the flag.
	for _, v := range flag {
		// Append the string representation of the current character plus the value of z to the runed slice.
		runed = append(runed, string(v+z))
		// Update z to the value of the current character.
		z = v
	}

	// Join the strings in the runed slice and assign it to the flag variable.
	flag = strings.Join(runed, "")
}

// The main function opens a file named "the" and writes the content of the flag into it.
func main() {
	file, err := os.OpenFile("the", os.O_RDWR|os.O_CREATE, 0644)
	if err != nil {
		// Print the error message and return if there's an error opening the file.
		fmt.Println(err)
		return
	}

	defer file.Close()

	// Write the content of the flag into the file.
	if _, err := file.Write([]byte(flag)); err != nil {
		// Print the error message and return if there's an error writing to the file.
		fmt.Println(err)
		return
	}
}
```
So, The `init()` function is executed before the `main()` function. It modifies the `flag` variable by applying a transformation to each character in the original flag. The transformation involves adding the current character's Unicode value to the previous character's Unicode value. The `main()` function opens a file named "the" and writes the modified flag into it.

So I will just write a solve script, that Initialize an empty list `runed` to store the reversed flag. Set the initial value of `z` to 0. Iterate over each character in the original flag. Subtract the current value of `z` from the Unicode value of the character. Convert the result to a character using `chr()` and append it to the `runed` list, and then Update `z` to the Unicode value of the current character.

![enter image description here](https://i.imgur.com/i39NXnV.png)
