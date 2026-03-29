[Sliver](https://github.com/BishopFox/sliver) is a command and control software developed by [BishopFox](https://bishopfox.com/). Used by penetration testers and red teamers, its client, server, and beacons (known as implants) are written in [Golang](https://go.dev/) - making it easy to cross-compile for different platforms.

## Initial Setup
Use this releases page to download servers / clients https://github.com/BishopFox/sliver/releases

Note: Servers and client versions must MATCH

Getting Started
```bash
# Server Setup
wget -q <link>
chmod +x ./sliver-server_linux
./sliver-server_linux

# Help
[server] sliver > help              # List of available commands
[server] sliver > help <command>    # Detailed information of each command

# Setup Operator profile 
[server] sliver > new-operator -n <name> -l <lhost>
[*] Saved new client config to: /path/to/<name>_<lhost>.cfg 

# Client setup
chmod +x ./sliver-client_linux
./sliver-client_linux import <name>_<lhost>.cfg # client requires operator config
./sliver-client_linux                           # run
```

Multiplayer Mode
```bash
# Start
[server] sliver > multiplayer 

# Commands related to multiplayer
kick-operator  # Kick an operator from the server
multiplayer    # Enable multiplayer mode
new-operator   # Create a new operator config file
operators      # Manage operators
```

Armory - pre-installed .NET binaries
```bash
sliver > armory --help              # help menu

Sub Commands:
=============
  install  # Install an alias or extension
  search   # Search for aliases and extensions by name (regex)
  update   # Update installed an aliases and extensions


sliver > armory install <package>
sliver > armory install all         # install all packages
```

