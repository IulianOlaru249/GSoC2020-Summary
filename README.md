## Overview

[Click me](http://www.google.com){: .btn}	<button name="button" onclick="http://www.google.com">Click me</button>

The goal is to enhance the Linux kernel debugging support on imx8 and imx8m platforms by capturing traces when the DSP panics.

Before working on thisWorking on this, I had no prior experience with open source projects and kernel development. It has been a great learning experience for me, and I would like to thank [Daniel Baluta](https://github.com/dbaluta) for his awesome guidance.

### The patches that make up the project

- [1. This allows imx platforms to use functions from the common core to acces information about the stack and registers.](https://github.com/thesofproject/linux/pull/2322)
- [2. This adds a common memory region in the mailbox so that the firmware can pass panic messages to the kernel.](https://github.com/thesofproject/linux/pull/2341)
- [3. This tells the firmware to send the panic message to the kernel and generate an interrupt.](https://github.com/thesofproject/sof/pull/3282)
- [4. This adds the functions that will gather the information about registers, stack , file name and line number and then will print it to the console.](https://github.com/thesofproject/linux/pull/2348)

## How was it done?

The first step was to get the hardware ready and set up the development envirnment.
I was working on an [I.MX8qm board](https://www.nxp.com/design/development-boards/i-mx-evaluation-and-development-boards/i-mx-8quadmax-multisensory-enablement-kit-mek:MCIMX8QM-CPU). The dev environment was set-up via u-boot, with the rootfs on the host machine and parted via an NFS server. The kernel image is cross compiled on the host machine and passed via a TFTP server.

### Setting up U-boot with NFS an TFTP

#### TFTP
Find which device is your board
```markdown
ls /dev//<tty device>
```

Connect to the board via serial communication. I personally prefer Picocom.
```markdown
sudo picocom -b 115200 <device>
```

Your board and physical machine must be the same network. They can be connected to the same network via a Switch. The board will get it’s IP address via DHCP. Run the **dhcp** command in the u-boot terminal

Set up tftp server on physical machine. [This is a useful tutorial](https://linuxhint.com/install_tftp_server_ubuntu/).
```markdown
sudo apt update
sudo apt install tftpd-hpa
```
Make sure that no other process is taking up port 69, which is default for your tftp server. If so, find the process pid, kill it and restart the tftp server
```markdown
sudo netstat -nlp | grep ":69"
kill -9 <pid>
sudo systemctl restart tftpd-hpa
```

Find the server ip. To test if all works, ping the IP from the u-boot terminal.

Finally, place the compiled kernel Image in the tftp folder.

#### NFS
[This is a useful tutorial](https://wiki.emacinc.com/wiki/Setting_up_an_NFS_File_Server)
```markdown
sudo apt update
sudo apt install tftpd-hpa
```
You will need to create a directory that will act as the access point to the sever. I recommend creating it in your home, instead of root, so you don’t have permission issues with the files you want to share. Inside this directory, you might want to make another one where you will mount the rootfs. You can also mount it right here.

Set the config file: **sudo vim etc/exports**
```markdown
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/<path_to_nfs>/rootfs 192.168.*.*(rw,sync,no_root_squash,no_subtree_check)
```

#### Setting up SOF

First step is to [cross compile](https://gist.github.com/lategoodbye/c7317a42bf7f9c07f5a91baed8c68f75) the kernel for arm64.

Decompress the archive in the /opt (is not required but keeps things tidy)
```markdown
sudo tar xf gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz -C /opt
```

Set the architecture to arm 64
```markdown
export ARCH=arm64
```

Compile the kernel
```markdown
make defconfig
make
```

The resulted image is in **arch/arm64/boot/**
You will need to get the Image file and put in in your tftp directory.
From u-boot terminal, update the image environmental variable to match the name of this image
```markdown
setenv image Image
```

The resultet kernel tree file is in **arch/arm64/boot/dts/freescale/**
You will need the **imx8qm-mek-sof-wm8960.dtb** file
Copy it in your tftp directory.
From u-boot terminal, update the fdt_file environmental variable to match the name of this kernel tree file.
```markdown
setenv fdt_file imx8qm-mek-sof-wm8960.dtb 
```

[Get the yocto project rootfs.](https://eur01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fwww.nxp.com%2Fwebapp%2FDownload%3FcolCode%3DL5.4.24_2.1.0_MX8QXPB0%26appType%3Dlicense&data=02%7C01%7Cdaniel.baluta%40nxp.com%7C034a2337763b4dd2fb8e08d81cd2c292%7C686ea1d3bc2b4c6fa92cd99c5c301635%7C0%7C0%7C637291038830412906&sdata=zCxfrNQMs%2BxTPoW6ikFb2ofFjhAoFkn9QCjW%2FBjt3Vk%3D&reserved=0)

Decompress the archive.
Look for **imx-image-multimedia-imx8qxpmek.tar.bz2**. This will be your rootfs.
Decompress the **imx-image-multimedia-imx8qxpmek.tar.bz2** in your NFS directory (or a 	subdirectory of it).

Update the **/etc/exports** file so the path points to where you decompressed the archive then run
```markdown
exports -ra
```

From the u-boot terminal, update the nfsroot environmental variable to point to where you 	decompressed the archive.
setenv nfsroot <path_to_rootfs>

#### Booting the board

You will need to set some environmental variable from the u-boot terminal.
```markdown
setenv nfsroot <path>/nfs (path to the rootfs mounted un the nfs directory)
setenv image < zImage name > (the kernel image file name from the tftp directory)
setenv fdt_file <dtb file name on host> (the .dtb file from the tftp directory)
setenv netargs 'setenv bootargs console=${console},${baudrate} ${smp} root=/dev/nfs ip=dhcp nfsroot=${serverip}:${nfsroot},v3,tcp'
	
To boot: run netboot
To save the config: saveenv
To auto boot: setenv bootcmd run netboot
```

Play a song.

Get a .wav file on the rootfs (.mp3 not supported yet).
```markdown
aplay -Dhw:0,0 -f S32_LE -c 2 -r 4800 file.wav
```
