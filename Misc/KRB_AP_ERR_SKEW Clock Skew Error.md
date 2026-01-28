```powershell
# Retrieve Windows time (UTC)
(Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")

# Set Kali time to that
sudo date -u -s "<timestamp>"
```
