A monitoring tool, usually for DevOps. 

RCE
```bash
Administraton > Scripts
# Check what other scripts are executing on

# Create script
	# Name : Whatever
	# Scope : Manual host action
	# Type : Script
	# Execute on : whatever other scripts are firing on
	# Commands
	rev shell, or whatever you want
	
	# Save

# Fire script
Monitoring > Hosts > Zabbix Server > click > CLick the script you made
```