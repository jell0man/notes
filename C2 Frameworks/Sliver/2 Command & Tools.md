Specific implementation of Sliver commands and tooling can be checked in their respective source files at /sliver/client/command/`<dir>/<command/tool>.go` (mostly).
## Commands

Basic Command Usage
```bash
sliver (http-session) > help

<SNIP>

Sliver - Windows:
=================
  backdoor          Infect a remote file with a sliver shellcode
  dllhijack         Plant a DLL for a hijack scenario
  execute-assembly  Loads and executes a .NET assembly in a child process (Windows Only)
  getprivs          Get current privileges (Windows only)
  getsystem         Spawns a new sliver session as the NT AUTHORITY\SYSTEM user (Windows Only) - requires local admin
  impersonate       Impersonate a logged in user.
  make-token        Create a new Logon Session with the specified credentials
  migrate           Migrate into a remote process
  psexec            Start a sliver service on a remote target
  registry          Windows registry operations
  rev2self          Revert to self: lose stolen Windows token
  runas             Run a new process in the context of the designated user (Windows Only)
  spawndll          Load and execute a Reflective DLL in a remote process
  hashdump          Dump hashes (requires SYSTEM!)


Sliver:
=======
  info               Get system info
  cat                Dump file to stdout
  cd                 Change directory
  chmod              Change permissions on a file or directory
  chown              Change owner on a file or directory
  chtimes            Change access and modification times on a file (timestomp)
  close              Close an interactive session without killing the remote process
  download           Download a file
  execute            Execute a program on the remote system
  execute-shellcode  Executes the given shellcode in the sliver process
  extensions         Manage extensions
  beacons            List beacons and status
  sessions           List interactive sessions and their states
  interactive        Convert beacon -> Session
  tasks              Check status of tasks pending for beacons to execute
<SNIP>
```

## Tools
Sliver offers lots of cool functionality, including `download`, `upload`, and `execute-assembly`. 

`execute-assembly` allows us to run .NET binaries on the target machine, without uploading them. However, one caveat that `execute-assembly` has is it will spawn a child process.

> **OPSEC NOTE:** By default, execute-assembly will spawn a `notepad.exe` process when run with any .NET binary. Obviously not ideal. `--process` resolves this.

Tool Usage
```bash
<tool> --help

# Download
download [flags] remote-path [local-path]

  #-F, --file-type string    force a specific file type (binary/text) if looting
  #-h, --help                display help
  #-X, --loot                save output as loot
  #-n, --name      string    name to assign the download if looting
  #-r, --recurse             recursively download all files in a directory
  #-t, --timeout   int       command timeout in seconds (default: 60)
  #-T, --type      string    force a specific loot type (file/cred) if looting

# Upload
upload [flags] local-path [remote-path]

# Execute-assembly
execute-assembly [flags] filepath [arguments...]

Flags
======
  -M, --amsi-bypass                 Bypass AMSI on Windows (only supported when used with --in-process)
  -d, --app-domain        string    AppDomain name to create for .NET assembly. Generated randomly if not set.
  -a, --arch              string    Assembly target architecture: x86, x64, x84 (x86+x64) (default: x84)
  -c, --class             string    Optional class name (required for .NET DLL)
  -E, --etw-bypass                  Bypass ETW on Windows (only supported when used with --in-process)
  -h, --help                        display help
  -i, --in-process                  Run in the current sliver process
  -X, --loot                        save output as loot
  -m, --method            string    Optional method (a method is required for a .NET DLL)
  -n, --name              string    name to assign loot (optional)
  -P, --ppid              uint      parent process id (optional) (default: 0)
  -p, --process           string    hosting process to inject into (default: notepad.exe)
  -A, --process-arguments string    arguments to pass to the hosting process
  -r, --runtime           string    Runtime to use for running the assembly (only supported when used with --in-process)
  -s, --save                        save output to file
  -t, --timeout           int       command timeout in seconds (default: 60)
```



