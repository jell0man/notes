```powershell
Enable-PSRemoting -Force
New-NetFirewallRule -DisplayName "WinRM" -Direction Inbound -Protocol TCP -LocalPort 5985,5986 -Action Allow

# Minimal footprint
Set-Service WinRM -StartupType Automatic -Status Running; winrm quickconfig -quiet
New-NetFirewallRule -DisplayName "WinRM" -Direction Inbound -Protocol TCP -LocalPort 5985,5986 -Action Allow
```