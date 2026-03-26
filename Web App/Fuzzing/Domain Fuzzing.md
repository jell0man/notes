NOTE: if a subdomain/vhost is configured for localhost only, this will not reveal it. `/etc/apache2/sites-enabled` will reveal it though
 
How to check if there are sub-domains/vhosts? -- ffuf!

## Sub-Domain Fuzzing
If there are public sub-domains present, we can fuzz with ffuf very simply like so...

Sub-Domain Fuzzing
```bash
# NOTE: http vs https MATTERS. If DNS is mentioned in nmap and specifies *.domain in http or https, fuzz with the SPECIFIED protocol. 
ffuf -w <wordlist>:FUZZ -u https://FUZZ.<server>/
```

## Vhost Fuzzing
If we are fuzzing sub-domains that do not have public DNS records, we must fuzz HTTP headers instead. We use the -H flag to do this.

Vhost Fuzzing with ffuf
```bash
# NOTE: http vs https MATTERS. If DNS is mentioned in nmap and specifies *.domain in http or https, fuzz with the SPECIFIED protocol. 

1. # Determine word length
ffuf -u http://<domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.<domain>"

# ctrl-c, look at word length

2. # Exclude most common word-length
ffuf -u http://<domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.<domain>" -fw <word-length>

3. Add subdomain/v-host to /etc/hosts
```

wfuzz alternative (fuff sometimes fails)
```bash
# wfuzz
1. wfuzz -u <url> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.domain"

# pipe to invert-match grep
1. | grep -v '<n.> W'
```

## Options
Options
```bash
# throlle requests /second
-rate <n.>
```

Wordlists
```bash
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt

-w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt

# use when throttling
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```