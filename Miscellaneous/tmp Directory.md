/tmp is often mounted with the `noexec` flag, which prevents scripts from running

Example:
```bash
mount | grep tmp

/dev/mapper/vg00-TmpVol on /tmp type ext4 (rw,noexec,nosuid,nodev)
```

So if you are trying to privesc, /tmp MIGHT get in your way