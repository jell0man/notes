`msfconsole`
#### Sessions
Useful when we want to run additional modules on already exploited system

Summary:
	background current session
	search for second module
	specify session and fire (if Module can be used with another session, there will be a SESSION option we can modify to specify)

Background session
```
background / [CTRL] + [Z]
```

Listing Active Sessions
```
sessions
```

Interact with a Session
```
sessions -i <no.>
```

Specify SESSION of secondary module and fire
```
set SESSION <no.>
run
```

#### Jobs
Summary:
	If we terminate a session (CTRL + C), the port is still in use
	we can use `jobs` to terminate old tasks running in background

Jobs help menu
```
jobs -h
```

Run an exploit as a background job
```
exploit -j
or
run -j
```

Listing Running Jobs
```
jobs -l
```


#### Meterpreter

Dump hashes (requires adequate permissions)
```
hashdump
```

Upload Files
```
upload <file>
```

#### post/multi/recon
	0  post/multi/recon/multiport_egress_traffic  .                normal  No     Generate TCP/UDP Outbound Traffic On Multiple Ports
	1  post/multi/recon/local_exploit_suggester   .                normal  No     Multi Recon Local Exploit Suggester
	2  post/multi/recon/reverse_lookup            .                normal  No     Reverse Lookup IP Addresses
	3  post/multi/recon/sudo_commands

#### Errors
getuid
```
ps
migrate <wmiprvse_PID>
```