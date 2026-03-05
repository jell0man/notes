_krb5_ : Kerberos network authentication protocol version 5

Prior to engaging with Kerberos on a target, it is good practice to configure the krb5.conf file point to the correct KDC and realm. Otherwise, Kerberos-based tools such as getTGT, getST may fail.

Modify krb5.conf
```bash
# Replace <> entries based on environment. Capitalization matters!
sudo bash -c 'cat > /etc/krb5.conf << EOF
[libdefaults]
    default_realm = <DOMAIN.LOCAL>
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    <DOMAIN.LOCAL> = {
        kdc = <dc_ip>
        admin_server = <dc_ip>
    }

[domain_realm]
    <.domain.local> = <DOMAIN.LOCAL>
    <domain.local> = <DOMAIN.LOCAL>
EOF'
```
