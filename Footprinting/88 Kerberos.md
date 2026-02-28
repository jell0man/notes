## User Enumeration

Nmap
```bash
nmap -p 88 --script=krb5-enum-users --script-args krb5-enum-users.realm=<Domain>,userdb=<Wordlist> <IP>

# Wordlists
/usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt 
/usr/share/wordlists/seclists/Usernames/Names/names.txt 
/usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```

Kerbrute
```bash
Download: https://github.com/ropnop/kerbrute

./kerbrute userenum <UserList> --dc <IP> --domain <Domain>


# Cheetsheet

Available Commands:
  bruteforce    Bruteforce username:password combos, from a file or stdin
  bruteuser     Bruteforce a single user's password from a wordlist
  help          Help about any command
  passwordspray Test a single password against a list of users
  userenum      Enumerate valid domain usernames via Kerberos
  version       Display version info and quit
```

Rubeus
```powershell
# with a list of users
.\Rubeus.exe brute /users:<UserList> /passwords:<Wordlist> /domain:<Domain>

# Check all domain users again password list
.\Rubeus.exe brute /passwords:<Wordlist>
```
