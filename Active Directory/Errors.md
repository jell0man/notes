KRB_AP_ERR_SKEW Clock Skew Error
```powershell
## Windows
# Retrieve Windows time (UTC)
(Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")

# Set Kali time to that
sudo date -u -s "<timestamp>"


## Exegol
faketime "$(rdate -n $DC_IP -p | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh
	# this enters a subshell. to exit, run 'exit'
```

Blank DCSync (In exegol)
```bash
# On Exegol → always add -use-ntds to any secretsdump.py command that expects to dump NTDS / do DCSync. On Kali / Fortra Impacket → don't need it.
-use-ntds
```