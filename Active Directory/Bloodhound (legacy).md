Make sure to use compatible versions of sharphound with legacy bloodhound
	https://github.com/SpecterOps/BloodHound-Legacy
	https://github.com/SpecterOps/BloodHound-Legacy/tree/master/Collectors
## Neo4j Setup
```bash
# Initialize db
neo4j start  # as root

# Setup
firefox
http://127.0.0.1:7474
neo4j : neo4j         # default creds
	# if this is not working, you can disable auth by the following
	vim /etc/neo4j/neo4j.conf
	dbms.security.auth_enabled=false  # uncomment this line

# Change creds... If you disable auth, bloodhounw will not require creds

# Now you can run bloodhound and login ;)
```

## Usage
Start
```
bloodhound
```

#### Collection:
NOTE: When you pwn a box, consider running SharpHound too. Might produce different output than collecting as a user via nxc / bloodhound-python

SharpHound
```powershell
# .exe
.\SharpHound.exe [--ldapusername <user>] [--ldappassword <password>]

# .ps1
Import-Module .\sharphound.ps1
Get-Help Invoke-Bloodhound
Invoke-BloodHound -CollectionMethod All -OutputDirectory 
```

Bloodhound-python (ldap)
```bash
bloodhound-python -c All -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' [-k -no-pass] -ns 10.80.80.2 [--zip]

# via proxy
proxychains -q bloodhound-python -c All -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' [-k -no-pass] -ns 10.80.80.2 --dns-tcp
```

#### Analysis
Some Cypher Queries
```cypher
MATCH (m:Domain) RETURN m
MATCH (m:User) Return m
MATCH (m:Computer) RETURN m
MATCH (m:Group) RETURN m

MATCH (n {objectid: 'S-1-5-21-...' }) RETURN n   // Resolve unknown SID
```

Look through EVERYTHING, including computer outbound object control. Even if ingesting multiple domains of info... 
	Cross-domain/cross-forest ACL misconfigurations do exist real world, as strange as it might seem
	So check computer outbound rights...

#### Clear database
![[Pasted image 20250226191828.png]]

