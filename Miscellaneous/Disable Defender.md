NOTE: On more modern this systems, this may not work
```powershell
# Locally
Set-MpPreference -DisableRealtimeMonitoring $true

# Add exclusion
Add-MpPreference -ExclusionPath 'C:\path\to\dir'
```