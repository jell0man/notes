If you ssh and recieve an error like:
	`Load key "id_rsa": error in libcrypto

this is due to End of Line endings being messed up when you copy the file (usually from git or windows, etc)

#### Fix:
Open file in notepad++

Show liine endings 
![[Pasted image 20250531171355.png]]

If CRLF, it is messed up

to fix, Edit > EOL Conversion > Unix (LF)
![[Pasted image 20250531171501.png]]

Ensure LF is on last line as well

Save and paste back into file


If this doesnt work, try `dos2unix` on your file and retry as well


If this also does not work, verify the formatting is perfect
	Look for dashed at beginning of every line (INCLUDING 1ST and LAST lines)

