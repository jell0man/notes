Installation - https://exegol.com/install

This if for me working from Ubuntu WSL. Note I only have access to the free image.

Exegol Usage
```bash
# HTB One-Liner Setup
exegol start inquisitor --desktop --vpn </mnt/c/path/htb.ovpn> -V /path/on/host/:/path/in/container
	# -V : mount point
	# Once setup, you can just run this to start it in others panes/tabs
	exegol start inquisitor

# Exit exegol shell
exit

# Basic Syntax
exegol [options...] <argument> [options...]

# Required arguments:
    start               Automatically create, start / resume and enter an Exegol container
    stop                Stop an Exegol container
    restart             Restart an Exegol container
    install             Install Exegol image
    build               Build a local Exegol image
    update              Update an Exegol image
    upgrade             Upgrade an Exegol container
    uninstall           Remove Exegol image(s)
    remove              Remove Exegol container(s)
    exec                Execute a command on an Exegol container
    info                Show info on containers and images (local & remote)
    activate            Activate an exegol license
    version             Print current Exegol version

# Optional arguments:
  -h, --help            
  -v, --verbose         
  -q, --quiet           
  --offline             # Run exegol in offline mode, no request will be made on internet (default: Disable)
  --arch {arm64,amd64}  # Overwrite default image architecture (default, hosts arch: amd64)

# Starting containers
exegol start [container] [image] [options...]  
	# my first container :)
	container : inquisitor
	
	# options
	--disable-X11  # Disable X11 sharing to run GUI-based applications
	--desktop      # Enable desktop feature
	--vpn VPN      # setup .ovpn or .conf (wireguard) connection at creation
	--log          # enable shell logging

# Executing one-off commands
exegol exec <container OR image> <command> [command...]

# Get container info
exegol info [container_name]
```

[[Windows Terminal]] (for quick reference :))
```bash
Ctrl + Shift + [num]   # new tab... 6 = Ubuntu for me

Alt + Shift + +        # Split pane vertically (to the right)
Alt + Shift + -        # Split pane horizontally (below)
Alt + Shift + <Arrow>  # Resize pane in direction of arrow
```

Desktop Troubleshooting
```bash
exegol info inquisitor   # make note of desktop port

# Inside exegol
ss -tulnp | grep -i vnc  # identify port vnc is listening on
websockify -D --web=/usr/share/novnc 6080 127.0.0.1:<vnc port>

# Access Web GUI
http://localhost:<desktop port>/
```

Custom Setup
```bash
# User Setup
useradd -m -s /bin/zsh jell0man
passwd jell0man  # set pass

usermod -aG sudo jell0man
cp /root/.zshrc /home/jell0man/
chown jell0man:jell0man /home/jell0man/.zshrc
cp -r /root/.oh-my-zsh /home/jell0man/
chown -R jell0man:jell0man /home/jell0man/.oh-my-zsh
sed -i 's|/root/.oh-my-zsh|/home/jell0man/.oh-my-zsh|g' /home/jell0man/.zshrc
chmod -R 777 /opt/my-resources/setup
mkdir -p /home/jell0man/.cargo
touch /home/jell0man/.cargo/env
chown -R jell0man:jell0man /home/jell0man/.cargo
cp -r /root/.asdf /home/jell0man/
cp -r /root/.pyenv /home/jell0man/
chown -R jell0man:jell0man /home/jell0man/.asdf /home/jell0man/.pyenv

su - jell0man
asdf set -u golang 1.24.1
```