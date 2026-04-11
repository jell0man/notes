Modify GET to a POST request
	need to add these:
	`Content-Type: application/x-www-form-urlencoded
	`Content-Length = <should auto-update>`
		make sure this is updating from 0 or you may get errors
		![[Pasted image 20250512185446.png]]
		make sure this setting is enabled so Content-Length auto updates
	`<api_field>=<our_revshell_code>