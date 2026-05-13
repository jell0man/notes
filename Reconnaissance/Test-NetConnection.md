ICMP might be blocked ;)

```powershell
Test-NetConnection -ComputerName <ip> -Port <port>

# Wanna test connection from a box you dont have direct auth on? Might have to if envorntment firewall is WEIRD
Invoke-Command -ComputerName <hostname> -Credential $cred -ScriptBlock { powershell Test-NetConnection -ComputerName <target> -Port 5985; Test-NetConnection -ComputerName <target> -Port 445 }
```