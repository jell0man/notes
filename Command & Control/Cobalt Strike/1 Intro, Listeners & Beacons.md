## Primer
Cobalt Strike is a market leading command & control framework that provides a post-exploitation agent to simulate stealthy, long-term embedded actors.

Components
	Beacon
		Post-exploitation agent.
	Team Server
		Central control and logging system.
	Client
		Used by operators to connect to one or more team servers to manage engagement and interact with Beacon payloads

Distributed Ops
	Cobalt Strike design pattern for distributed ops is to stand up dedicated team servers for each phase of an engagement such as Initial Access, post-exploitation, and persistence

## Listener Management
_Listener_ - defines the protocol and parameters by which a Beacon payload will communicate.
	Egress
		direct communication to team server via network boundary
		DNS and HTTP/S
	Peer-to-Peer
		Traffic routed through another beacon
		SMB and TCP

```
in general, i would say: 

dns -> long-haul persistence 
http/s -> post-ex 
smb -> lateral movement 
tcp-local -> priv esc
```
### HTTP Listener
Directs Beacon to communicate with team server via HTTP GET and/or POST requests

Cobalt Strike > Listeners > Add

Use screenshot as reference
![[cd98408e96475f8313c578129df93d90 1.png]]
#### Options
_HTTP hosts_ - Hosts that beacon sends HTTP requests to

_Host Rotation Strategy_ - Tells Beacon how it should use each one
	_Round robin_ - Beacon loops through list top-bottom
	_Random_ - Beacon randomly selects host for each request
	_Failover_ - Beacon uses same host until failover condition is met
		x - x failed attempts
		m/h/d - fails over time period
	_Rotate_ - Each host used for a given time period

_Max retry strategy_ - Configures Beacons self destruct strategy 
	_None_
	_Exit_
		syntax: exit-`exit-[max_attempts]-[increase_attempts]-[duration][m/h/d]`
		example
			**exit-50-25-1h** tells Beacon to increase its sleep time to 1 hour after 25 failed attempts, and then to exit if it reaches 50 failed attempts.

_HTTP host (stager)_ - Only used by stager payloads

_Profile_ - Malleable C2 profiles can be configured from this dropdown

_HTTP port (C2)_ - This is the HTTP port Beacon will attempt to connect to the team server on

_HTTP port (bind)_ - This is the port that the team server will bind its built-in web server to.  If no option is specified, it will use the same port as above.
	This is useful for running multiple HTTP listeners on the same team server, but keeping all of your Beacons talking out on port 80.

_HTTP host header_ - Setting a host header here will propagate it into the Beacon payload without having to explicitly set it within a Malleable C2 profile

_HTTP proxy_ - By default, Beacon will attempt to use the internet proxy configuration of the system it is running on.  However, you can hardcode HTTP or SOCKS proxy information for the Beacon to use instead if required.

_Guardrails_ - Guardrails prevents stageless Beacon payloads from running unless the specified criteria has been met.
	ie: Blue teamer analyzing the payload in a sandbox


### DNS Listener
The DNS listener directs Beacon to communicate with a team server via DNS requests - specifically A, AAAA, or TXT record lookups

Requirements:
	Necessary DNS records to make team server authoritative for one or more subdomains

![[82e6a89abe525f2da618acf98dd39e6f.png]]
_DNS Resolver_ - Allows for custom DNS resolver to be set

### SMB Listeners
Does not bind or listen on team server VM

![[58595535b398f255108eae9a8ecc083a.png]]
When executed, an SMB Beacon payload will create a SMB named pipe using the **pipename** specified in the listener configuration

Cobalt Strike uses **msagent_##** by default (random hex values)

### TCP Listeners
Does not bind or listen on team server VM

![[bbc4cacd5f1957bc920d9eedc2e878c2.png]]

When executed, a TCP Beacon payload will bind and listen on the C2 port specified in the listener configuration.
	Bind to localhost only:
		Checked = 127.0.0.1
		Not checked = 0.0.0.0

## Beacon Payloads
#### Introduction
Beacon is a Windows DLL - To execute a beacon payload, we need to load its DLL into a process and (usually) kick off a new thread to call its entry point. 

Beacon's DLL exports a function ReflectiveLoader
![[Pasted image 20251218184009.png]]

#### New prepended loader
Since Cobalt Strike 4.11, Beacon has begun to use a new style of reflective loader based on DoublePulsar. More stealthy and flexible
![[Pasted image 20251218184716.png]]

#### Staged vs stageless payloads
Size of payload may be limited due to nature of vulnerability, or the channel its accessible on. Thus resulting in 'payload staging'

Payload Security
	Due to small size, stagers do not carry same security and stageless payloads (ie keypair auth).
	Susceptible to hijacking when first executed

OPSEC
	To keep stagers small, they omit 'good steps practice'
	Size difference
		Beacon stager ~890 bytes
		Full Beacon stage ~ 307200 bytes

#### Generating Payloads
Beacon Payloads generated from Payloads menu

HTML Application Attack
	`.hta` format`
	x86

MS Office macro
	VBA macro
	x86

Stager payload generator
	produce source code files - similar to `msfvenom` 
	Use if dropping into own shellcode runners

Stageless payload generator
	More options than Stager
	Exit Function - controls API that gets called when Beacon's `exit` is executed
		Process = ExitProcess
		Thread = ExitThread
	System Call - tells Beacon to use syscalls instead of Win32 APIs for internal operation
		Direct = Uses Nt* version of function
		Indirect = Jumpts to appropriate instruction with Nt* version of function

Windows stager/stageless payload
	Similar to payload generators BUT
	Provide pre-build executables vs raw source files
		Windows EXE
		Windows Service EXE
		Windows DLL

Windows Stageless Generate All Payloads
	Generates all possible stageless payload variants for every available listener in both x86 and x64 architectures

## Interacting with Beacon

View > Event Log
	Record appears here when new session checks into team server
	![[Pasted image 20251218192029.png]]

Monitor denotes privileges
	Blue monitor = medium-integrity (standard user)
	Red monitor = High-integrity (privileged user or SYSTEM)
	![[Pasted image 20251218192153.png]]

Interaction
	double click on beacon row or `Interact`
	![[Pasted image 20251218192232.png]]

List commands
```bash
help             # List of commands
help <command>   # Help on specific command
```

Tasking the Beacon
	Entering commands places job in queue
	`clear` - Clears the queue

#### Session Graph View
Helps visualize connections when Table View is confusing
![[Pasted image 20251218192923.png]]
The firewall icon and dashed line represent egress Beacons that are talking directly out to the team server

The solid lines represent P2P connections
	Dashed green for HTTP/S.
	Solid yellow for SMB.
	Dashed yellow for DNS.
	Solid green for TCP.