Anonymous Auth — Always Check First
	Try both `anonymous` with a blank password AND `anonymous:anonymous`.
	Password spray with `auxiliary/scanner/ftp/ftp_login` or Netexec (use a USER_PASS file).

Tips
	**Active mode** (`-A`) — sometimes required to auth or transfer files
	**Binary mode** (`binary`) — always use when transferring sensitive files (e.g. `.kdbx`) to prevent corruption
		The alternative is ASCII mode, which translates line endings (`\r\n` ↔ `\n`) during transfer. Breaks binaries 
	**Write access?** Upload reverse shells to web root or FTP root. IIS → target the `iisstart.htm` directory
	**Extra perms needed?** Try `admin:admin`, `Admin:Admin`, `<User>:<User>`, or any creds already found
	**Metadata** — run `exiftool`, `strings` on every file that seems out of place, check author fields and hidden properties

Usage
```bash
# Authentication
ftp <ip> [-A]                         # connect
	# anonymous :                     # blank password
	# anonymous : anonymous           # try both
	# -A : Active mode

# Binary mode
binary   # For senstive transfers (db, etc...)

# Download all
mget *   # download all files (skips directories)
wget -m --user=<user> --password=<pass> ftp://<ip>  # recursive download

# Some quick static analysis post-exfil
exiftool <file>   # check metadata / author fields
strings <file>
```