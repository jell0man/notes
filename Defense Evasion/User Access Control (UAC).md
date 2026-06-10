UAC Enumeration
```powershell
# Query UAC
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
	# 0x0 = Disabled
	# 0x1 = Enabled

# Query ConsentPromptBehaviorAdmin
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
	# 0x0 = Elevate without prompting (the best!)
	# 0x1 — Prompt for credentials on the secure desktop (must re-type password)
	# 0x2 — Prompt for consent on the secure desktop (dimmed-screen "Yes/No")
	# 0x3 — Prompt for credentials
	# 0x4 — Prompt for consent
	# 0x5 — Prompt for consent for non-Windows binaries (the Windows default)
```