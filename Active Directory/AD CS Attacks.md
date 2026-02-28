See https://academy.hackthebox.com/module/details/236 ADCS Module (still need to do...)

See https://0xdf.gitlab.io/2023/06/17/htb-escape.html#smb---tcp-445 for an example

One thing that always needs enumeration on a Windows domain is to look for Active Directory Certificate Services (ADCS). A quick way to check for this is using `netexec`

Initial Check
```bash
nxc ldap <victim_ip> -u 'user' -p 'password' -M adcs
```

## Identify Vulnerable Template

#### Certify.exe
We can use `Certify.exe` to identify vulnerable services. The README has a walkthrough to enumerate and abuse certificate services

Usage
```powershell
.\Certify.exe find /vulnerable /currentuser
	# /currentuser checks for groups of the current user
```
Based on the results, the README will guide on the actions to take

#### Certipy
`Certipy` is an alternative tool and can be run remotely

Usage
```bash
certipy find -u 'user' -p 'pass' [-hashes <NTLM>] -target <target> -text -stdout -vulnerable

# Look Here
[!] Vulnerabilities
```
