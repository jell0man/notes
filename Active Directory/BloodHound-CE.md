Okay I'm finally going to move on from legacy to CE

Setup
```bash
pipx install bloodhound-ce

bloodhound-ce &
# http://localhost:1030  
# Set pass to whatever you want  
```

Collection
```bash
# Collection
bloodhound-ce-python -u <user> -p '<pass>' -d <domain> -ns <dc_ip> -c All --zip
```

Enumeration
```bash
CYPHER
FOLDER
```