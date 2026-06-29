Plink, short for PuTTY Link, is a Windows command-line SSH tool that comes as a part of the PuTTY package when installed.

Usage:
```cmd
# set up proxifier -- proxychains windows alternative essentially
127.0.0.1 9050 SOCKS4

# use like ssh
plink -ssh -D 9050 <user>@<ip>
```