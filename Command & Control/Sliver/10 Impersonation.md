Impersonation is the process of changing our current access token of the user we run as to another user. `make-token`, `impersonate` and `runas` can be used to get an authentication session.
## make-token
Use `make-token` to create a new token with harvested credentials.

```bash
sliver > help make-token

Usage:
======
  make-token [flags]

Flags:
======
  -d, --domain     string    domain of the user to impersonate
  -h, --help                 display help
  -T, --logon-type string    logon type to use (default: LOGON_NEW_CREDENTIALS)
  -p, --password   string    password of the user to impersonate
  -t, --timeout    int       command timeout in seconds (default: 60)
  -u, --username   string    username of the user to impersonate
  
# Revert
rev2self
```

| Type                    | Description                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| LOGON_INTERACTIVE       | Type of logon mostly used by every user on the computer, be it using a console, RDP session or else.                            |
| LOGON_NETWORK           | Type of logon intended for high performance servers that accept authenticators such as a Password, NT Hash, Kerberos Ticket     |
| LOGON_BATCH             | Type of logon equivalent to a Scheduled Task, Startup or else that could execute a process on behalf of the user                |
| LOGON_SERVICE           | Type of logon where the account must have service privileges enabled                                                            |
| LOGON_UNLOCK            | Type of logon that uses [DLLs/GINA](https://learn.microsoft.com/en-us/windows/win32/secgloss/g-gly) that are loaded by Winlogon |
| LOGON_NETWORK_CLEARTEXT | Type of logon that utilizes plaintext credentials                                                                               |
| LOGON_NEW_CREDENTIALS   | Type of logon allow the current user to clone its token while specifying new set of credentials for outbound connections        |
Example
```bash
sliver > make-token -u svc_sql -d child.htb.local -p jkhnrjk123! [-T LOGON_INTERACTIVE]
# [*] Successfully impersonated child.htb.local\svc_sql. Use `rev2self` to revert to your previous token.
```
> -T is useful as this will modify what you are able to do. Default for moving laterally, but interactive if you want to stay in place...

