my creds
	neo4j:neo

Neo4j Default creds
	neo4j:neo4j

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
bloodhound
```
