Note: x86_64/x86/amd64 are same names for 64-bit AMD/Intel CPUs. Similarly AArch64/arm64/ARMv8/ARMv9 are same names for 64-bit ARM CPUs

Ubuntu 24.04.1 LTS

`sudo docker ps`
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
28910d1a2545   ubuntu    "bash"    10 minutes ago   Up 10 minutes             objective_hawking

On mac
Install:
	Open terminal
	`docker pull --platform linux/x86_64 ubuntu

Run container
	`docker run -it --platform linux/x86_64 ubuntu bash
	`cat /etc/os-release`
		verify the os

Install nc
	`apt-get update && apt-get install -y netcat-traditional`
	to transfer this over, we are transfering over /usr/bin/nc.traditional, NOT /usr/bin/nc
		/nc is a symbolic link (think a windows shortcut), not the actual binary

Install gcc
	`apt-get install gcc

How to transfer files
	`docker cp <containerId>:/file/path/within/container /host/path/target
	example
	`docker cp 28910d1a2545:/usr/bin/nc /Users/howell/Mount
		transfers x86 nc binary to my shared mount with kali ARM
		on kali, go to /mnt/utm and should see it


___
Debian
`docker pull --platform linux/amd64 debian
`docker run -it --platform linux/amd64 debian bash`
then repeat updates above

___
KALI
`docker pull --platform linux/amd64 kalilinux/kali-rolling`
`docker run -it --platform linux/amd64 kalilinux/kali-rolling bash
repeat above
