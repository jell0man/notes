Use `proxychains4 <commands...>` to utilize proxy specified in /etc/proxychains.conf file

Firefox over proxychains
```bash
# Must be ONLY browser running :(
proxychains4 firefox
```

What conf file to use?
```
proxychains looks for config file in following order:

file listed in environment variable PROXYCHAINS_CONF_FILE or provided as a -f argument to proxychains script or
binary.

/proxychains.conf

$(HOME)/.proxychains/proxychains.conf

/etc/proxychains.conf

/etc/proxychains4.conf
```
so just edit /etc/proxychains.conf :)

Modifying the conf file
```/etc/proxychains.conf
...SNIP...
SOCKS5  127.0.0.1 9050
```

http proxy instead of socks (such as [[Squid Proxy]])
	specify user and pass if required
	(s) in http is dependent on if proxy is over https or not
```/etc/proxychains.conf
...SNIP...
#SOCKS5  127.0.0.1 9050
http(s) <ip> <port> <user> <pass>
```