A collection of oneliners to fire to rapidly loot.

Also if you downloaded a bunch of files, you can loot for info like this
```bash
grep -r -l 'password' <directory> 2>/dev/null
```
## Windows

Search for passwords
```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

Search for specific file
	`dir /a /s *<file>
	or
	`Get-ChildItem -Path C:\ -Filter <file> -Recurse -ErrorAction SilentlyContinue -Force 
```powershell
Edit as needed

# Search for password manager databases on C:\ Drive
Get-ChildItem -Path C:\ -Include *.kdbx,*.db,*.sql -File -Recurse -ErrorAction SilentlyContinue

# Search for sensitive info, text files and password manager databases in home directory of dave
Get-ChildItem -Path C:\Users\sasha -Include *.txt,*.pdf,*.xls,*.xlsx,*.doc,*.docx,*ini -File -Recurse -ErrorAction SilentlyContinue

# Config Files
Get-ChildItem -Path C:\ -Include *.conf,*conf*,*.cfg,*cfg*,*.config,*config* -File -Recurse -ErrorAction SilentlyContinue

# .ini Files
Get-ChildItem -Path C:\ -Include *ini* -File -Recurse -ErrorAction SilentlyContinue

# Look for zip files (may be backups)
Get-ChildItem -Path C:\ -Include *.zip -File -Recurse -ErrorAction SilentlyContinue

# Search for SAM/SYSTEM/SECURITY accessible by current user
Get-ChildItem C:\ -Recurse -Force -ErrorAction SilentlyContinue | Where-Object { $_.Name -match '^(SAM|SYSTEM|SECURITY)$' -and -not $_.PSIsContainer } | Format-List FullName,Length
```

LaZagne
```
.\LaZagne.exe all
```

Browser Creds
```
If you see any of these in winpeas output, I reccomend a meterpreter module, like post/windows/gather/enum_browsers
```

## Linux
```bash
# conf files
for l in $(echo ".conf .config .cnf .xml");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done

# user, pass, password in cnf files
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done

# databases
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done

# commonly encrypted files
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*");do echo -e "\nFile extension: " $ext; find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core" ;done

# ssh keys
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

# notes
find /home/* -type f -name "*.txt" -o ! -name "*.*"

# scripts
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done

# enumerating history files
tail -n5 /home/*/.bash*

# log files -- still manually enum but this might help
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done


# memory
sudo python3 mimipenguin.py
sudo python2.7 laZagne.py all

# browser creds
ls -l .mozilla/firefox/ | grep default 
cat .mozilla/firefox/1bplpd86.default-release/logins.json | jq .
python3.9 firefox_decrypt.py

or

python3 laZagne.py browsers
```