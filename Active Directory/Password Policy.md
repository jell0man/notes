#### Password Policy Enumeration from Linux
```bash
# Credentialed
nxc smb <ip> -u <user> -p <pass> --pass-pol

# SMB NULL
rpcclient -U "" -N <ip>
$> querydominfo

# enum4linux
enum4linux -P <ip>
or...
enum4linux-ng -P <ip>

# LDAP Anonymous bind
ldapsearch -h <ip> -x -b "DC=<DOMAIN>,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength

```
#### Password Policy Enumeration from Windows
```cmd
# Null Session
net use \\DC01\ipc$ "" /u:""

# net.exe
net accounts
```

```powershell
import-module .\PowerView.ps1
Get-DomainPolicy
```
