# gpu-pt
Scripts to assist with GPU passthrough


### Credits:
This is based on code and guides located at https://gitlab.com/risingprismtv/single-gpu-passthrough

**NOTE** - All commnads listed below are for Arch linux as seen in my video, check the gitlab link above for gides and commands for other distros

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
If you do not have any virtualisation settings, like said before make sure your BIOS is up to date, and that your CPU and motherboard support virtualisation.

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
Edit the line that starts with ```GRUB_CMDLINE_LINUX_DEFAULT``` so it ressembles something like this, **keeping any previous parameters if there are any:**
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
A warning like the below example may apear in your terminal:
```:: iptables and iptables-nft are in conflict. Remove iptables? [y/N]```
If you do encounter this kind of message, press ```y``` and ```enter``` to continue the installation.

**Note** - When installing you may see the following:
```
:: There are 3 providers available for qemu:
:: Repository extra
   1) qemu-base  2) qemu-desktop  3) qemu-full
```
Select option 2 by typing ```2``` and then hit ```enter``` to continue.

