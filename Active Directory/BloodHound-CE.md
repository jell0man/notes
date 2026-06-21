Okay I'm finally going to move on from legacy to CE

Setup
```bash
pipx install bloodhound-ce

bloodhound-ce &
# http://localhost:1030  
# Password will be printed on first run. If lost, run
bloodhound-ce-reset
```

Collection
```bash
# Collection
bloodhound-ce-python -u <user> -p '<pass>' -d <domain> -ns <dc_ip> -c All --zip

rusthound -d <domain> -u <user> -k \
      -i 10.129.14.105 -n <FQDN> -f <FQDN> -z
```

Converter https://github.com/szymex73/bloodhound-convert

Enumeration
```bash
CYPHER
FOLDER
```