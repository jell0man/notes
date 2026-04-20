Shortcuts
```PowerShell
TAB            # autocomplete
Ctrl + SPACE   # menu of options (try after '-')
powershell -ep bypass  # launch with execution policy bypass
```

Cmdlets
```powershell
$env:<VAR>     # print variable from environment
Clear-Host     # clear screen
Get-Alias      # list aliases
Get-ChildItem  # list files/folders (dir/ls)
Where-Object   # filter objects in the pipeline
Select-Object  # filter properties (cmdlet | Select-Object -Property Name)
Out-File       # send output to a file
Format-List    # format output as a list (* for all properties)
Export-Csv .\<name>.csv -NoTypeInformation # export data to CSV
```

Common Imports
```PowerShell
Import-Module ActiveDirectory  # AD interaction
$session = New-PSSession      # Remote management (WinRM)
# Common Net namespaces:
# [System.Net.Http.HttpClient] for Web APIs
# [System.Net.Sockets.TcpClient] for low-level networking
```

Variables and Typing
```PowerShell
# Basic types (Variables always start with $)
$name = "jell0man"            # String
$count = 10                   # Integer
$pi = 3.14                    # Double/Float
$isActive = $true             # Boolean ($true/$false)

# Type Casting
$stringNum = [string]100
$integerVal = [int]"50"

# String Interpolation (Use double quotes to expand variables)
Write-Host "User: $name | Active: $isActive"
```

Computing and Comparisons
```PowerShell
# Arithmetic
$result = 10 / 3              # 3.3333...
$remainder = 10 % 3           # 1 (Modulo)

# Comparisons (Word-based operators)
# -eq (==), -ne (!=), -gt (>), -lt (<), -ge (>=), -le (<=)
if ($name -eq "root" -or $isActive) {
    Write-Output "Access Granted"
}

# Membership testing
if ("root" -in @("admin", "root", "operator")) {
    Write-Output "Found"
}
```

Command Line Arguments
```PowerShell
# Param block (Must be at the very top of the script)
Param(
    [Parameter(Mandatory=$true)]
    [string]$TargetIp,

    [int]$Port = 80
)

Write-Host "Scanning $TargetIp on port $Port..."
```

Functions
```PowerShell
function Get-PwnCheck {
    param($Target, $Port = 80)

    if ($Target -eq "127.0.0.1") {
        return "Localhost skipped"
    }
    return "Scanning $Target on $Port..."
}

# Calling (No parentheses or commas!)
$msg = Get-PwnCheck -Target "10.10.10.1" -Port 443
```

Loops
```PowerShell
# Range loop
for ($i=0; $i -lt 5; $i++) {
    Write-Host "Attempt $i"
}

# Collection loop (foreach)
$targets = @("10.0.0.1", "10.0.0.2")
foreach ($ip in $targets) {
    Write-Host "Pinging $ip"
}

# While loop
$count = 0
while ($count -lt 3) {
    Write-Host "Checking..."
    $count++
}
```

Arrays
```PowerShell
$ips = @("127.0.0.1", "192.168.1.1")
$ips += "10.10.10.10"         # Add item
$firstIp = $ips[0]            # Access
$subset = $ips[1..2]          # Slicing (inclusive)
```

Hash Tables (Dictionaries)
```PowerShell
# Creation
$userData = @{ username = "jell0man"; uid = 1000 }

# Access/Update
$userData.username
$userData["shell"] = "powershell.exe"

# PSCustomObject (Best for structured engineering output)
$obj = [PSCustomObject]@{
    Target = "DC01"
    Status = "Pwned"
}
```

Errors and Exceptions
```PowerShell
try {
    # -ErrorAction Stop is required to trigger 'catch' for most cmdlets
    $content = Get-Content "targets.txt" -ErrorAction Stop
}
catch [System.IO.FileNotFoundException] {
    Write-Error "[-] Error: targets.txt not found."
}
catch {
    # $_ represents the current error object
    Write-Error "[-] An unexpected error occurred: $_"
}
finally {
    Write-Host "[*] Script cleanup finished."
}
```

Classes and Initialization
```PowerShell
class Target {
    [string]$IP
    [string]$Hostname
    hidden [string]$SecretKey # Hidden from standard output

    Target([string]$ip, [string]$hostname) {
        $this.IP = $ip
        $this.Hostname = $hostname
    }

    [void] Display() {
        Write-Host "Target: $($this.Hostname) ($($this.IP))"
    }
}

# Usage
$machine = [Target]::new("10.10.10.5", "DC01")
$machine.Display()
```

Inheritance
```PowerShell
class Scanner {
    [void] Run() { Write-Host "Generic scan..." }
}

# Inherit using the colon
class PortScanner : Scanner {
    [void] Run() { Write-Host "Starting Nmap scan..." }
}

# Usage
$tool = [PortScanner]::new()
$tool.Run()
```