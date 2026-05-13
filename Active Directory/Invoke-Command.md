## Why It Matters

evil-winrm and psexec can't forward creds (network logon / SYSTEM). Invoke-Command with explicit `-Credential` creates a real logon session — solves double-hop.

Requirements
	Requires WinRM (port 5985) open on target
	 `-Credential` is mandatory to avoid double-hop — without it you get `A specified logon session does not exist`
	Hash-only auth doesn't work — need plaintext password (reset with bloodyAD if needed)
	ScriptBlocks are isolated — define variables inside each block

## Basic Usage

```powershell
$pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('DOMAIN\User', $pass)

Invoke-Command -ComputerName TARGET -Credential $cred -ScriptBlock { whoami; hostname }
```

## Nested (Double-Hop Killer)

When you're on A, need to reach C, but only B can talk to C:

```powershell
# A → B → C
Invoke-Command -ComputerName B -Credential $cred -ScriptBlock {
    $pass = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
    $cred2 = New-Object System.Management.Automation.PSCredential('DOMAIN\User', $pass)
    Invoke-Command -ComputerName C -Credential $cred2 -ScriptBlock {
        type C:\Users\Administrator\Desktop\flag.txt
    }
}
```

Creds must be rebuilt inside each ScriptBlock — outer variables don't pass through.

## Persistent Session

```powershell
$s = New-PSSession -ComputerName TARGET -Credential $cred
Invoke-Command -Session $s -ScriptBlock { ... }  # reuse session, no re-auth
Remove-PSSession $s
```
