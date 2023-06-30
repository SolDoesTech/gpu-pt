# gpu-pt
Scripts to assist with GPU passthrough


### Credits:
This is based on code and guides located at https://gitlab.com/risingprismtv/single-gpu-passthrough

**NOTE** - All commands listed below are for Arch linux as seen in my video, check the gitlab link above for guides and commands for other distros

---
## Preparations:
Check if your motherboard BIOS version is up to date. (This can help you have better IOMMU groups for the next steps)
Check if your system is installed in UEFI mode, do not use CSM, as it will install your system as Legacy, which doesn't work with GPU passthrough. (Force UEFI in your BIOS when installing your distro)

---
### Bios settings
Depending on your machine CPU, you need to enable certain settings in your BIOS for your passthrough to succeed. Enable the settings listed in this table:

| AMD CPU | Intel CPU |
|---------|-----------|
| IOMMU   | VT-D      |
| NX mode | VT-X      |

Note for Intel : You may not have both options, in that case, just enable the one available to you.
If you do not have any virtualization settings, like said before make sure your BIOS is up to date, and that your CPU and motherboard support virtualization.

---
## Editing GRUB
### Enable IOMMU
Set the parameter respective to your system in the grub config:
| AMD CPU      | Intel CPU     |
|--------------|---------------|
| amd_iommu=on | intel_iommu=on|

Set the parameter ```iommu=pt``` in grub config for safety reasons, regardless of CPU type

Mostly for AMD users, the parameter ```video=efifb:off``` can fix issues when returning back to the host, it is recommended that you add it.

Run ```sudo nvim /etc/default/grub```
Edit the line that starts with ```GRUB_CMDLINE_LINUX_DEFAULT``` so it resembles something like this, **keeping any previous parameters if there are any:**
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"
```
On my clean install with Intel CPU the line looked like this:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet intel_iommu=on iommu=pt"
```

After you edit the file you will nee dto update grub with this command ```sudo grub-mkconfig -o /boot/grub/grub.cfg```

---
## Configuration of libvirt
### Installing the virtualization packages
Now that we prepared both the bios and grub, we need to install the necessary virtualization packages on our system.
```
sudo pacman -S virt-manager qemu vde2 ebtables iptables-nft nftables dnsmasq bridge-utils ovmf
```
**Please note:** Conflicts may happen when installing these programs.
A warning like the below example may appear in your terminal:
```:: iptables and iptables-nft are in conflict. Remove iptables? [y/N]```
If you do encounter this kind of message, press ```y``` and ```enter``` to continue the installation.

**Note** - When installing you may see the following:
```
:: There are 3 providers available for qemu:
:: Repository extra
   1) qemu-base  2) qemu-desktop  3) qemu-full
```
Select option 2 by typing ```2``` and then hit ```enter``` to continue.

### Editing the libvirt config files
We now need to edit some of the config files. We will start with libvirt.conf
```
sudo nvim /etc/libvirt/libvrtd.conf
```
Search the file for: ```unix_sock_group = "libvirt"``` (should be around line 85)
Search the file for: ```unix_sock_rw_perms = "0770"``` (should be around line 108).
make sure the both lines are uncommented (remove the ```#```)

Head to the bottom of the file and add the following lines to enable logging:
```
log_filters="3:qemu 1:libvirt"
log_outputs="2:file:/var/log/libvirt/libvirtd.log"
```

### Adding user to the libvirt group
Next you need to add your user to the libvirt group this would give you the correct permissions .
```
sudo usermod -a -G kvm,libvirt $(whoami)
```
To verify the command indeed worked execute the following to see all the groups your user is in:
```
sudo groups $(whoami)
```

### Starting the libvirt service
Execute the 2 commands below to enable and start the service
```
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

### Editing the qemu config file
Next we need to edit the qemu.conf
```
sudo nvim /etc/libvirt/qemu.conf
```

Search for and uncomment (remove the ```#```) the following 2 lines (should be around line 519 and 523):
```
user = "root"
group = "root"
```
Now that the lines are active we need to update both to use your username, my login is "sol" so my config would look like this:
```
user = "sol"
group = "sol"
```
Your config file would have your login, the above is just an example.

Next to apply all the changes we need to restart libvirt service
```
sudo systemctl restart libvirtd
```

### Enable the VM default network
This part is optional but it is highly recommended as it prevents a manual step when starting the VM, anything to make life easier :)
Execute the following command:
```
sudo virsh net-autostart default
```

If you opted out from doing the above you will need to run the below command every time before you can start your VM
```
sudo virsh net-start default
```

## Configure the Virtual Machine
Next we need to download ISO for the OS. In this tutorial I am going to cover an Arch VM which should be fairly simple if you are already running Arch to follow this document. I will also cover installing of Windows 11 but you can choose to install windows 10 instead which is a similar process (heck you can install both if you want, the power of virtualizing!)

### Download The virtio tools
head on over to [virtio-win-pkg-scripts](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)
Download the ```Stable virtio-win ISO``` 

Next download the [Arch Linux ISO](https://archlinux.org/download/)

And last, grab the [Windows 11 ISO](https://www.microsoft.com/software-download/windows11)


### Getting the GPU ROM
For the VM to work correctly with the installed GPU it would need to have access to the GPU ROM (or BIOS).
This step is needed for Nvidia cards if you have an AMD GPU then congratulations, should be able to skip this step. 

**Warning!!** Do not download "pre-made" ROMS from the internet, you have been warned.

head on over to to this link and download the [Nvidia NVFlash](https://www.techpowerup.com/download/nvidia-nvflash/) tool. You would want to download the latest **"Linux"** version from the left side.

Once you have the file, extract it and note the location (~/Downloads/ by default).

### Dumping the ROM
Switch to a TTY with ```CTRL + ALT + F3``` (or F4)

Stop your display manager, here are some examples
SDDM -  ```sudo systemctl stop sddm```
GDM - ```sudo systemctl stop gdm3```

Now we need to unload the drivers so the NVFlash tool can have access to the GPU, execute the following:
```
sudo rmmod nvidia_uvm
sudo rmmod nvidia_drm
sudo rmmod nvidia_modeset
sudo rmmod nvidia
```
next navigate into the NVFlash extracted location and make the utility executable with:
```sudo chmod +x nvflash```

Now run the utility with this command:
```sudo ./nvflash --save vbios.rom```

If everything worked, you should have a new file named **vbios.rom**

Now we need to reload the drivers and get back to the desktop, the easy way is to ```reboot```, but you can also do this:
```
sudo modprobe nvidia
sudo modprobe nvidia_uvm
sudo modprobe nvidia_drm
sudo modprobe nvidia_modeset
```

Now we are ready to start the display manager, so depending which one you stopped earlier you can use:
SDDM - ```sudo systemctl restart sddm```
GDM - ```sudo systemctl restart gdm3```

### Patching the ROM
In order to use the ROM it must be first edited with a HEX editor. Download Okteta with:
```
sudo pacman -S okteta
```
