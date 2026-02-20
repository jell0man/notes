```powershell
# Retrieve Windows time (UTC)
(Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")

# Set Kali time to that
sudo date -u -s "<timestamp>"


## Exegol
faketime "$(rdate -n $DC_IP -p | awk '{print $2, $3, $4}' | date -f - "+%Y-%m-%d %H:%M:%S")" zsh

```
