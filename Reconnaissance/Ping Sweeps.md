#### Step 1
When exposed to an internal network, ping sweeps may be used to identify hosts

Attempt twice!!!

```bash
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
```

```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```

```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

#### Step 2
Blocked ICMP?
```bash
proxychains nmap <whole_subnet> -Pn -n --disable-arp-ping -sT -sV -T5  # Remove -T5 if errors occur
```