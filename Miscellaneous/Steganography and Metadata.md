```bash
# Identify weird stuff
exiftool <file>
exiftool -a -u -g1 <file>   # More stuff

# Extract
steghide extract -sf image.jpg

# Checking for strings
strings image.png | grep -iE 'pass|key'

# Check for hidden archives
binwalk -e <image.png>

# Check for steg, zlib-compressed data, weird palettes
zsteg [-a] image.png  -a : all
```



#### Crack steghide files

Old way `stegcracker`
	`stegcracker <file> <wordlist> -t <threads>` default is 16
	retired
	replaced by stegseek

New Way `stegseek`
https://github.com/RickdeJager/stegseek/blob/master/BUILD.md
i have installed already
	Bruteforce
		`stegseek <file> <wordlist>
	Passwordless extraction
		`steegseek --seed <file>`