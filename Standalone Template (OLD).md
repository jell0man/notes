## Reconnaissance
```
If we add a host to /etc/hosts, RESCAN
```
#### TCP
Initial Scan
`sudo nmap <ip> -sC -sT -sV -p<ports> -oN ~/proving_grounds/<standalone>/<standalone>_TCP_ALL`


Deep Scan
`sudo nmap <ip> -A -p<ports> -oN ~/proving_grounds/<standalone>/<standalone>_TCP_ALL`

#### UDP
Initial Scan
`sudo nmap <ip> -sU --top-ports=100 -oN ~/proving_grounds/<standalone>/<standalone>_UDP`


Deep Scan
`sudo nmap <ip> -sU -A -p<ports> -oN ~/proving_grounds/<standalone>/<standalone>_UDP_ALL`

#### Autorecon
`sudo env "PATH=$PATH" autorecon <ip_address>

#### Nikto
`nikto -host <ip_address>   # or /etc/hosts hostname`



## Parking Lot
```
# Annotate sensitive files here that we do not have access to, in case we GAIN access later
```

```
Park creds/key/domain info here:
```
## Footprinting
```
1. Unknown Port? Search for it DIRECTLY on HackTricks
2. google '<service> exploits' AND '<service> RCE'
3. Any ports that do not reveal much, redo scan with -A and specify port
4. Redo scan if host added to /etc/hosts
5. Public exploit not working? Verify, then search for others on github
```


#### 80 http
```
# Notes
```

nmap findings
	

nikto
	-

Visit website
	

Wappalyzer
	versions (including name of site)
		-
	searchsploit
		-
	`google "<version> exploits", "<version> rce", etc
		-

Manual Enumeration
	poke around
	-

Page Source
	Inspect page source
		-
	html2markdown -- `curl -s <target> | html2markdown`
		-

login portal
	default creds
		admin:admin
		Admin:Admin
		source config documentation
			-
	sqli `# try ' and also auth bypass`
		-
	forgot password abuse
		-
	check KALI local repo
		-
	hydra brute force ([[Web App Bruteforcing]])
		rockyou.txt
			-
		`cewl` wordlist
			-
		other
			-
	user/cred reuse
		-


**Some Common pages to try**
/robots.txt
	-

/.htaccess `# ALSO via file upload`
	-

/.env
	-

/.git
	-

/api/
	also try /api
	-


[[SQLi]] -- [Payload All The Things](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md) `# For reference`
	Manual Findings
		-
	SQLMap `# always try if stuck...`
		-


**File Inclusion**
[[Methodology/Web App/File Inclusion/Local File Inclusion (LFI)|Local File Inclusion (LFI)]] Identification  -- `# /index.php?view=<file_name>`
	yes or no?

LFI Enumeration -- Manual + Fuzzing
	-

**File Inclusion RCEs**
[[Methodology/Web App/File Inclusion/RCE/Remote File Inclusion (RFI)|Remote File Inclusion (RFI)]]
	-
[[LFI Log Poisoning]]
	-
[[PHP Wrappers]] -- Also config file disclosure
	-
[[LFI and File Uploads]]
	-



file upload -- see [[File Upload Attacks]]
	-

Command injection
	-

HTTP Verb Tampering
	-

IDOR
	-

XXE Injection `# not in scope for OSCP`
	-

Read php files -- see [[PHP Wrappers]] `# php://filter...`
	`index.php?page=php://filter/resource=<file>
		-
	`index.php?file=php://filter/convert.base64-encode/resource=<file>
		-

Hacktricks
	-

Automated scanning
	wpscan
	git-dumper
	etc...

Weird/out of place pictures
	-

Duplicate webpages `# ie: /old, compare page sources`
	-

Exposed Config files
	-

Found hashes? `# crackstation, john, hashcat...`
	-

Enumerate webpages
[[Feroxbuster]] + Extension scan `# sh files? shellshock`
	-

[[Feroxbuster]] Re-scan MANUALLY identified directories
	-

[[Domain Fuzzing]] `# 2 wordlists -- Don't forget to add subdomains to /etc/hosts -- ffuf AND wfuzz
	-

[[Parameter Fuzzing]]
	-

Check github documentation for interesting files
	if you get directory redirects from directories found from ferox, try accessing files within anyway by referencing the github documentation of the webapp

Guess directories (last resort)
	Try names we have enumerated
	ie: if hostname = `<name>`, then try `http://<ip>/<name>`



## Foothold
```
Need to compile something? Ideally we are able to compile on target...
	yes: great
	no: search for precompiled online OR compile locally 

Public exploit not working? Verify, then search for others on github
```


## Internal Enumeration
```
# If in container, do FULL enumeration anyway
# Initial Impressions (if any):

```
#### Windows Manual:
hostname
	-

whoami
	-

whoami /priv
	-

whoami /groups
	-

set
	-

`dir \Users
	-

local users `# net user`
	-
	specific users `# net user <user>`    `# do for EVERY user, check comments, etc...`
		-

domain users `# net user /domain
	-
	specific users
		-

net localgroup
	-

net group
	-

systeminfo
	-

network information
	`ipconfig /all
		-
	`route print
		-
	`netstat -ano     # internally exposed ports? chisel! webapps? conf files, LFI + rev shell, etc..` 
		-

installed apps
	32 bit `--- # Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname`
		-
	64 bit `--- # Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname`
		-
	`reg query` any interesting software
		-

Running Apps `# Get-Process`
	-

User History `Get-History`
	-
	PSReadline `# (Get-PSReadlineOption).HistorySavePath`
		-
	All Users (see [[Windows History]])
		-

switch user `# also consider runas /user:<domain>\<user> cmd.exe`
	`user:user` cred combinations
		-
	credential reuse
		-
	ssh reuse
		-


Services w/ binary path `# Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}`
	-

DLL Hijacking `#procmon`
	-

Unquoted Service Paths `# Remove 'auto' filters if we can start/stop services`
	CMD `# wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """`
		-
	PowerShell `# Get-CimInstance -ClassName Win32_Service | Where-Object { $_.PathName -notlike '"*' and $_.StartMode -eq 'Auto' -and $_.PathName -like '* *' -and $_.PathName -notlike 'C:\Windows\*'} | Select-Object Name, PathName`
		-
	Manually Check (Look at Installed Apps and probe)
		-

ChangeConfig Service Right Abuse (See [[Services]])
	`Import-Module .\Get-ServiceAcl.ps1`
	`"<Service>" | Get-ServiceAcl | select -ExpandProperty Access`
		-


Scheduled Tasks  `# schtasks  |  schtasks /query /fo LIST /v  |  Get-ScheduledTask
	-
	Hunt for specific users -- `Get-ScheduledTask | Where-Object -Property Author -match <user> | fl *`
		-
	`Get-ScheduledTask | Select-Object Author,Trigger,Action,TaskName,TaskPath,Source,Principal | fl`
		-
	TaskScheduler -- `#GUI - look at author, trigger, action, etc...`
		-


Manual Enum: `# dir /a /q`
`\directory\we\spawn\in    # and adjacents`
	-
`\Users\<user>\AppData\stuff
	-
`\Users\<user>\<all_files>
	-
`\Users\<users>
	-
`\
	-
`\xampp`
	-

Search for specific files `# Use Get-ChildItem premade queries from master checklist` See [[Looting]]
	Config Files
		-
	kdbx files
		-
	.txt, .ini files
		-
	home dir recursive search for creds/databases
		-
	zip files
		-
	SAM/SYSTEM/SECURITY
		-

.\winPEASany.exe `# also winPEAS.ps1... PARSE CAREFULLY... AlwaysInstallElevated...
	-

.\LaZagne.exe all
	-

.\jaws-enum.ps1
	-

Import-Module .\PowerUp.ps1
	`Invoke-AllChecks`
		-
	`Get-ModifiableServiceFile
		-
	`Get-UnquotedService`
		-

windows exploit suggester
	-

#### Linux Manual:
hostname
	-

whoami
	-

id
	-

sudo -V
	-

sudo -l
	-

env
	-

history
	-

/etc/passwd
	users
		-
	file perms
		-

/etc/shadow
	-

/etc/sudoers
	-

switch user `# su -l <user>`
	`user:user` cred combinations
		-
	credential reuse
		-
	ssh reuse
		-

os/kernel version info `# is gcc present?` [[OSCP Prep/Methodology Notes/PrivEsc/Kernel Exploits|Kernel Exploits]]
	-
	`searchsploit #.#.#-###`
		-
	google
		-

system processes `# ps aux`
	-

interfaces `# ip a`
	-

routes `# routel`
	-

ports `# internally exposed ports? chisel! webapps? conf files, LFI + rev shell, etc..
	-

multiple webservers `# check /var/www and check /etc/apache2/sites-enabled or /etc/nginx/sites-enabled`
	NOTE: vhost could share same port but only be accessible from localhost `# ie: port-forward`
	-

scheduled tasks `# pspy64 automated, crontab -l, cat /var/log/syslog | grep -i cron`
	-

installed packages `# dpkg -l`
	-

world writable directories `# find / -writable -type d 2>/dev/null`
	-

mounted drives `# cat /etc/fstab, lsblk`
	-

list loaded drivers `# lsmod`
	-

SUID files `# find / -perm -u=s -type f 2>/dev/null`
	-

Password Reuse `# identified passes/strings go here`
	-
	redo `sudo -l` if password was required
		-
	local db auth
		-
	switch users `# su -l <user>   AND   ssh <user>@<ip>`
		-
	misc
		-

Manual Enum:  `# consider ls -laiRtu

`/directory/we/spawn/in`    `# and adjacent ones`
	-
`/home/<user>
	-
`/home
	-
`/`
	-
`/opt`
	-
`/var`      `# www/, mail/, etc... cred hunting galore`
	-
anything else of interest
	-


Config files and information within:
	-

.db search `# find / -name *.db 2>/dev/null      ## also *db*, *database*, etc...`
	-

.sql search `# find / -name *.sql 2>/dev/null      ## also *sql*, etc...
	-

.conf search `# find / -name *.conf 2>/dev/null      ## also *conf*, *cfg*, etc...`
	-

Fire `./linpeas.sh > /tmp/linpeas.txt`
	manually parse
		-
	`grep -i pass
		-
#### Automated:
```

```

## Lateral Movement #1
```
NOTE: if using su, specify -l or - to simulate user logon

# Use this section if we switch users
```

sudo -l  `# or whoami /priv...`
	-

env
	-

repeat enumeration process as user -- focus on user specific directories, groups, etc...



## PrivEsc
```
# Be sure to check common vectors first (to ALL users, including root) 
	password reuse, user:user cred combos, ssh key reuse
# Try polkit (Pwnkit), especially when attempting kernel exploits github.com/ly4k/PwnKit | /joeammond/CVE-2021-4034 (python version)
# /tmp might be mounted with noexec
# sudo -l binaries if you cannot find anything, try googling <binary> CVE
```
#### Attempt 1: <>



## Lessons Learned