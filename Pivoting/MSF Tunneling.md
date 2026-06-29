See [[msfvenom]] for meterpreter initial rev shell

#### MSF Port Forwarding
```bash
# catch an msf shell
# background session

# check /etc/proxychains.conf
...SNIP...
socks4 	127.0.0.1 9050

# setup in msfconsole
use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set version 4a
run -j

# add routes
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
run
# ALTERNATIVE
run autoroute -s 172.16.5.0/23

# List active routes
run autoroute -p

```