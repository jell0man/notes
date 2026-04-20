```
#tcpdump can be used to test connectivity

sudo tcpdump -i tun0 "icmp"

ping -c 192.168.45.183

#we should recieve icmp packets if conenction is being made
```


