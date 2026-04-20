I am hoping to god this fixed it
https://www.reddit.com/r/vmware/comments/umwdyc/vmware_tools_issue_with_kali/

```
sudo apt autoremove open-vm-tools
sudo apt install libfuse-dev open-vm-tools-desktop fuse
reboot
```

___
BELOW DOES NOT WORK

if it acts up, restart vmtools
	`sudo systemctl restart vmtoolsd.service
	`sudo systemctl restart open-vm-tools
	check their status with `sudo systemctl status <service>`


If we need to install a new VM, follow these steps

open .vmx file for VM

add these entries
```
isolation.tools.copy.disable = "FALSE"
isolation.tools.paste.disable = "FALSE"
```
done


___
Configure kali AND HOST so it never sleeps



Kali:
Settings Manager > Power Manager
![[Pasted image 20250607195529.png]]

![[Pasted image 20250607195701.png]]![[Pasted image 20250607195718.png]]


Settings Manager > Xfce Screensaver:
![[Pasted image 20250607202440.png]]
![[Pasted image 20250607202501.png]]



Settings > Session and Startup
![[Pasted image 20250607211125.png]]
![[Pasted image 20250607211149.png]]




Enable Presentation Mode
![[Pasted image 20250607204132.png]]
This is the way but we may have to toggle this after reboots


Also edit VMX File ("C:\Users\joshu\VMs\kali-linux-2024.4-vmware-amd64.vmwarevm\kali-linux-2024.4-vmware-amd64.vmx")
	add `battery.present = "FALSE"`
	



___
Host:
![[Pasted image 20250607200400.png]]
![[Pasted image 20250607200437.png]]