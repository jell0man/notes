```
Make sure to use compatible versions of sharphound with legacy bloodhound

https://github.com/SpecterOps/BloodHound-Legacy

https://github.com/SpecterOps/BloodHound-Legacy/tree/master/Collectors
```

#### Neo4j Setup
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

#### Usage
Start
	`bloodhound`
	or
	`~/tools/BloodHound/BloodHound --no-sandbox

Collection:
Run sharphound
	Sharphound.exe
	or
	`Import-Module .\sharphound.ps1
		`Get-Help Invoke-Bloodhound
		`Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\joe\Documents\windows -OutputPrefix "audit"`
Bloodhound-python (ldap)
```
bloodhound-python -c All -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' -ns 10.80.80.2

# via proxy
proxychains -q bloodhound-python -c All -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' -ns 10.80.80.2 --dns-tcp
```

consider netexec


Enum computers
	MATCH (m:Computer) RETURN m
		save to computers.txt file
		`nslookup <FQDN>
			with ligolo...
				
Enum users
	MATCH (m:User) Return m
		save to users.txt file


Clear database
![[Pasted image 20250226191828.png]]

