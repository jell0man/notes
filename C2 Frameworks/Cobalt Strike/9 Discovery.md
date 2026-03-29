_Discovery_ is a tactic where an adversary attempts to enumerate information about the environment they're operating in.

Why we don't just run a BloodHound collector like [SharpHound](https://github.com/SpecterOps/SharpHound) or [BloodHound.py](https://github.com/dirkjanm/BloodHound.py)?
	Default collectors will almost certainly get you caught by a mature security team.

What's the solution?
	Slower, more targeted data collection, LDAP queries, etc....
	A popular approach is to run your own data collection queries using a tool like [ldapsearch](https://github.com/trustedsec/CS-Situational-Awareness-BOF/tree/master) or [pyldapsearch](https://github.com/Tw1sm/pyldapsearch), and then use [BOFHound](https://github.com/coffeegist/bofhound) to parse the output into BloodHound-compatible JSON files. Then load into BloodHound to query usual.

Running BloodHound from Windows Lab
	Start Menu > Docker Desktop
	Containers
	Start all Containers and Wait
	Browse to `http://localhost:8080/ui/login`
	`admin` : `eA%N4frBrnn2` 
## Lightweight Directory Access Protocol (LDAP)
LDAP is an industry standard protocol for accessing directory service information across a network.

Ports: 389 (LDAP) and 636 (LDAPS)

Each 'object' in a directory has an associated **objectClass**. Some common ones:
	_computer_ : computer account in the domain
	_domain_ : domain info
	_group_ : stores list of user names, applies security principals to resources
	_person_ : contains info about a user
	_groupPolicyContainer_ : stores Group Policy Objects
	_organizationalUnit_ : Container for storing users, computers, and account objects
	_trustedDomain_ : represents a domain trust relationship
#### Querying LDAP

Indexes
	Searching by an indexed field will be much faster than a non-indexed field.  Microsoft have [documented](https://learn.microsoft.com/en-us/windows/win32/adschema/attributes-indexed) which directory attributes are indexed, so we should use these where possible.

Filters
	Searching LDAP requires we use filters.
	[sAMAccountType](https://learn.microsoft.com/en-us/windows/win32/adschema/a-samaccounttype) shows user accounts in HEX. We need to convert these to decimal FIRST
	[userAccountControl](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties) 

OPSEC
	A pain point even for custom data collection, is tripping a detection that looks for expensive, inefficient, or slow queries. All Org dependent.
	Expensive Search Results
		Queries that return more results than a given threshold.
		`(objectClass=*)` = BAD
	Search Time Threshold
		Queries that take longer than N milliseconds to run. Size and number of attributes.
		`*,ntsecuritydescriptor` = BAD
		`samaccounttype,distinguishedname,objectsid,ntsecuritydescriptor` = BETTER
	Inefficient Search Results Threshold
		These are queries that return less than 10% of the visited objects _if_ that number of visited objects is more than the given threshold.
		`(&(samAccountType=805306368)(servicePrincipalName=*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))` 
			find kerberoastable users
			probably inefficient...

NOTE: Returning the security descriptor is essential if you want BloodHound to identify ACL-based attack paths.

ldapsearch Usage -- [userAccountControl Property Flags](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/useraccountcontrol-manipulate-account-properties)
```powershell
## PREFACE - IF ENUMERATING A DOMAIN THAT ISNT YOUR OWN, YOU NEED TO ADD THE FOLLOWING TO A QUERY
	--dn DC=partner,DC=com --hostname partner.com # this is an example...

# syntax
beacon> ldapsearch [filter(s)] --[options]

# operators
&             # AND
|             # OR
!             # NOT

# bitwise filters -- these are OIDs. Used for userAccountControl
1.2.840.113556.1.4.803    # LDAP_MATCHING_RULE_BIT_AND
1.2.840.113556.1.4.804    # LDAP_MATCHING_RULE_BIT_OR
1.2.840.113556.1.4.1941   # LDAP_MATCHING_RULE_IN_CHAIN, querying object ancestry
(userAccountControl:1.2.840.113556.1.4.803:=FLAG) # Example usage
	# Some common UAC Property Flags
	DONT_REQ_PREAUTH 4194304                 # AS-REP ROASTABLE
	TRUSTED_FOR_DELEGATION 524288            # UNCONSTRAINED DELEGATION
	TRUSTED_TO_AUTH_FOR_DELEGATION 16777216  # S4U2self - Protocol Transition

# options
--attributes        # similar to -Properties in powershell. Filters out attributes.
--attributes [attributes...],ntsecuritydescriptor # read security descriptor (ACL). Required for BOFHound. Specify any attributes before it, or ALL with *

_________________________________________________________________________________

## example usage 

# Users
beacon> ldapsearch (samAccountType=805306368)   
	# Make note of ATTRIBUTES returned as we can filter by these

# Administrators
beacon> ldapsearch (&(samAccountType=805306368)(adminCount=1)) 

# users that are Administrator OR had 'adm' in name
beacon> ldapsearch (&(samAccountType=805306368)(|(description=*admin*)(samaccountname=*adm*)))   

# all admins but NOT krbtgt user
beacon> ldapsearch (&(samAccountType=805306368)(adminCount=1)(!(name=krbtgt))) 

# Administrators and only displaying their name and groups
beacon> ldapsearch (&(samAccountType=805306368)(adminCount=1)) --attributes name,memberof 


## Some more useful queries
# All computers with uncontrained delegation
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname # 

# Return ancestors of Domain Admins Group
beacon> ldapsearch "(memberof:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=[domain],DC=com)" --attributes samaccountname

# Find kerberoastable users -- likely ineffiencint and BAD OPSEC
beacon> ldapsearch (&(samAccountType=805306368)(servicePrincipalName=*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))

# Enumerate ALL user important details -- possibly expensive and BAD OPSEC
beacon> ldapsearch (samAccountType=805306368) --attributes samaccounttype,distinguishedname,objectsid,serviceprincipalname,useraccountcontrol,ntsecuritydescriptor

# Identify objects with UNCONSTRAINED DELEGATION
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname

# Identify objects with CONSTRAINED DELEGATION
beacon> ldapsearch (&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo

# Identify objects w/ PROTOCOL TRANSITION
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=16777216)) --attributes samaccountname

# Query MSSQL Servers
beacon> ldapsearch (&(samAccountType=805306368)(servicePrincipalName=MSSQLSvc*)) --attributes name,samAccountName,servicePrincipalName


FOREST AND DOMAIN TRUSTS

# Query TRUST DOMAIN OBJECTS (TDOs) - see 15 Forest & Domain Trusts for TDO chart
beacon> ldapsearch (objectClass=trustedDomain)

# Enumerate Trust in place
beacon> ldapsearch (objectClass=trustedDomain) --attributes trustPartner,trustDirection,trustAttributes,flatName # bidirectional is best!!! (3)

# Query Trust Accounts
beacon> ldapsearch (samAccountType=805306370) --attributes samAccountName

# Enumerate Hosts in domain
beacon> ldapsearch (samAccountType=805306369) --attributes samAccountName --dn DC=contoso,DC=enclave --hostname contoso.enclave

# Obtain domain SID for child domain
beacon> ldapsearch (objectClass=domain) --attributes objectSid

# Obtain domain SID of parent domain (or adjacent domain in general...)
beacon> ldapsearch (objectClass=domain) --attributes objectSid --hostname lon-dc-1.contoso.com --dn DC=contoso,DC=com # set dn to PARENT distignuished name
	# NOTE: when crafting ticket, add RID of 519 to end. This is EA group
	
# Enumerate for Foreign Security Principals
beacon> ldapsearch (objectClass=foreignSecurityPrincipal) --attributes cn,memberOf --hostname partner.com --dn DC=partner,DC=com
```

## BOFHound
BOFHound is not a BOF.  It's a Python script that is capable of parsing the raw output of the ldapsearch BOF (and a few other tools).

General Workflow
```powershell
## PREFACE - IF ENUMERATING A DOMAIN THAT ISNT YOUR OWN, YOU NEED TO ADD THE FOLLOWING TO A QUERY
	--dn DC=partner,DC=com --hostname partner.com # this is an example...

# Collect basic info like domain, computers, users, groups, OUs, and GPOs
beacon> ldapsearch (|(objectClass=domain)(objectClass=organizationalUnit)(objectClass=groupPolicyContainer)) --attributes *,ntsecuritydescriptor
beacon> ldapsearch (|(samAccountType=805306368)(samAccountType=805306369)(samAccountType=268435456)) --attributes *,ntsecuritydescriptor

# Copy logs from team server to desktop (using WSL and SCP)
attacker@DESKTOP-FGSTPS7:/mnt/c/Users/Attacker/Desktop$ scp -r attacker@10.0.0.5:/opt/cobaltstrike/logs .

# run bofhound againts copied log directory
attacker@DESKTOP-FGSTPS7:/mnt/c/Users/Attacker/Desktop$ bofhound -i logs/

# Run Bloodhound
Start Menu > Docker Desktop
Containers
Start all containes
Edge > http://localhost:8080/ui/login

# Log into BloodHound UI and upload parsed JSON files
Login to BloodHound
Administration > File Ingest
upload JSON files

# Cypher Query
# Query data with cypher queries
MATCH (m:Computer) RETURN m                   # computers
MATCH (m:User) Return m                       # users
MATCH (n:User) WHERE n.hasspn=true RETURN n   # kerberoastable users
Match (n:GPO) return n                        # GPOs

# Resolve SIDs with no name
beacon> ldapsearch (objectsid=[SID]) --attributes *,ntsecuritydescriptor

# redo copy log steps
# run bof hound
# Reingest
```
NOTE: any object that only has SID, or 'no name or id', we need to collect more data and ingest it.
	`beacon> ldapsearch (objectsid=[SID]) --attributes *,ntsecuritydescriptor`
#### Restricted groups
These are local group memberships that are defined / applied via GPOs. LDAP cannot collect this data alone. To pull this, you have to query each box with is NOISY and BAD OPSEC. We can workaround this via checking GPOs.

How to find restricted group data?
	Browse to the [gPCFileSysPath](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-ada1/6eb770b7-0c89-4a3e-a41e-2807d46880d8) of the GPO in SYSVOL in the container
	Restricted group data is defined in _GptTmpl.inf_ in Machine's SecEdit folder
		ie: `\\contoso.com\SysVol\contoso.com\Policies\{7A04B429-D293-45E8-A7BA-D630415B3D6A}\Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf`
	Download and open
```powershell
# Identify GPO we want to view. 
Match (n:GPO) return n  

# Identify Gpcpath
Right-Click GPO -> Annotate GPCpath

# Download Restricted Group Data
beacon> ls [Gpcpath]\Machine\Microsoft\Windows NT\SecEdit\
beacon> download [Gpcpath]\Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf

# Sync locally then view
View -> Downloads
```

What to do with restricted group data?
	Check under group membership
	Syntax
		`*[SID]__Memberof = *[SID]`
	First SID = domain group. look in BloodHound to identify
	Second SID = local group. Check microsoft [documentation](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers) to identify.
		ie: `S-1-5-32-544` = local admin
	What is this telling us?
		Members of domain group SID belong to local group SID on computer IDENTIFIED.

Update BloodHound manually with new information
```cypher
// This adds domain group to local admins group for each computer specified
// Do this for every computer GPO is associated with in BloodHound
MATCH (x:Computer{objectid:'S-1-5-21-3926355307-1661546229-813047887-1110'})
MATCH (y:Group{objectid:'S-1-5-21-3926355307-1661546229-813047887-1107'})
MERGE (y)-[:AdminTo]->(x)
```
#### WMI Filters
WMI filters allow for additional criterial to be set on GPO application via a WMI query.
	ie: GPO applies to entire workstations OU, but WMI restricts its app to certain Windows versions.

BloodHound currently does not collect, display, or evaluate WMI filters when building its graphs.

A GPO can have only 1 WMI filter applied at a time.

A WMI filter is applied to a GPO by modifying the GPO's `gPCWQLFilter` attribute.

Viewing WMI filters
```powershell
# Identifying WMI filter presence
beacon> ldapsearch (objectClass=groupPolicyContainer) --attributes displayname,gPCWQLFilter
	# example output
	# displayName: AppLocker
	# gPCWQLFilter: [contoso.com;{E91C83FB-ADBF-49D5-9E93-0AD41E05F411};0]

# Viewing WMI filter
beacon> ldapsearch (objectClass=msWMI-Som) --attributes name,msWMI-Name,msWMI-Parm2 --dn "CN=SOM,CN=WMIPolicy,CN=System,DC=contoso,DC=com"
	# look for the GUID previously identified

_______________________________________________________________________________
# Example WMI filter
name: {E91C83FB-ADBF-49D5-9E93-0AD41E05F411}
msWMI-Name: Windows 10+
msWMI-Parm2: 1;3;10;61;WQL;root\CIMv2;SELECT * from Win32_OperatingSystem WHERE Version like "10.%";
	# 1;3;10 is a constant value, no idea what it's for 😅
    # 61 is the length of the WMI query.
    # WQL is the query language (WMI Query Language).
    # root\CIMv2 is the query namespace.
    # SELECT * from Win32_OperatingSystem WHERE Version like "10.%"**  
	    # is the query itself (61 characters).
```
In the example, the query would mean if this GPO was applied to an OU, it would ONLY apply to Windows 10 computers.


