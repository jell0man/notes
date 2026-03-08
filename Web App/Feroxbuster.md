Usage:
```bash
# syntax
feroxbuster --url http://<ip_address>/<dir> --wordlist=<path_to_wordlist>

# wordlists
/usr/share/dirb/wordlists/common.txt
/usr/share/dirb/wordlists/big.txt
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
	DirBuster-2007_directory-list-2.3-big.txt

# extensions
-x php,txt,bak,html,zip,conf,py,aspx,sl,sh # sh files indicates shellshock exploit

# additional useful options
--filter-status 404
-H 'Authorization: Basic <base64_encoded_user:pass>'
```

Authenticated scan
	Some sites, you will have to authenticate before ANY pages load
	So we must pass the auth token to the scan
```bash
## EXAMPLE
# Intercept authentication request with Burp
...SNIP...
Authorization: Basic c2t5bGFyazpVc2VyK2RjR3Zmd1RialZbXQ==

# Add HTTP Header option to feroxbuster
-H 'Authorization: Basic c2t5bGFyazpVc2VyK2RjR3Zmd1RialZbXQ=='
```

