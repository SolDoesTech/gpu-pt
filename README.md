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

**NOTE** If you are having a hard time with switching TTY as requested later on in this document, you will need to add the "```nomodeset```" parameter as well and rebuild GRUB - example:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet nomodeset intel_iommu=on iommu=pt"
```

After you edit the file you will need to rebuild GRUB with this command ```sudo grub-mkconfig -o /boot/grub/grub.cfg```

**Dont forget to reboot**

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
- SDDM: ```sudo systemctl stop sddm```
- GDM: ```sudo systemctl stop gdm3```

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
- SDDM: ```sudo systemctl restart sddm```
- GDM: ```sudo systemctl restart gdm3```

### Patching the ROM
In order to use the ROM it must be first edited with a HEX editor. Download Okteta with:
```
sudo pacman -S okteta
```

Make a copy of the vbios.rom file **keep the original safe** so you have a backup. For the sake of this document we will call the copy "vbios_copy.rom"

Open vbios_copy.rom with Okteta, use ```CTRL + F``` to search the file and change the settings to ```Char``` and search for the value ```VIDEO```.

Now, place your cursor **before the first U** before the ```VIDEO``` value you found and select everything before it. Hit the **INS** (Insert) key to switch Okteta to Edit mode and with everything selected before the **U** hit the "Delete" key on your keyboard.

Save the file as another copy naming it "vbios_patched.rom".

### Moving the ROM
We now need to move the patched ROM to the correct location. First we need to create a directory:
```
sudo mkdir /usr/share/vgabios
```
Now we need to copy the ROM and set the permissions:
```
sudo cp ./vbios_patched.rom /usr/share/vgabios/
cd /usr/share/vgabios
sudo chmod -R 644 vbios_patched.rom
sudo chown yourusername:yourusername vbios_patched.rom
```

## Clone this repo
If you haven't done so already, clone a copy of this repo.
```
git clone https://github.com/SolDoesTech/gpu-pt.git
```
### Execute script
Now cd into the downloaded repo and execute the Install script:
```
cd gpu-pt
sudo ./install_hooks.sh
```

The above script install all the hooks needed to disconnect the video card and allows the GPU to be attached to the VM.

Next execute the "get-group" script with ```./get-group```. Pay attention to the devices in your GPU group as you will later need to attach all of these devices to your VM with the exception of any devices labeled as ```PCI bridge [0604]```

### Validate installed files
You would want to confirm that the following files were created:
```
/etc/systemd/system/libvirt-nosleep@.service
/bin/vfio-startup.sh
/bin/vfio-teardown.sh
/etc/libvirt/hooks/qemu
```

### Update the hook script
We now need to update the hook script so id doesn't execute by mistake as we are building vms:
```
sudo nvim /etc/libvirt/hooks/qemu
```
Edit the line that reads ```if [[ $OBJECT == "win10" ]]; then``` replacing "win10" with "somevm". This is a place holder, we will come back and edit this file again once our VM is ready for the GPU.

## Create a VM
Use the following process to create your VM, this process is similar for both Windows or Linux guest machines. Since you can only run one VM with GPU passthrough then you can follow this step as many times as needed for each type of VM you are going to use with the GPU passthrough feature. 

- Launch the "Virtual Machine Manager". 
- Click on "Create new virtual machine" button. 
- Step 1: Select "Local install media (ISO image or CDROM).
- Step 2: Click on "Browse" and then on "Browse Local" and navigate to your OS install ISO.
- Step 3: Assign resources, you can use the default here as we would update this value later.
- Step 4: Set the size of your VM disk storage.
- Step 5: Name your VM, select anything you like here as long as it is **NOT** "somevm". **IMPORTANT** place a check mark in "Customize configuration before install" and then hit "Finish"

### Customize the VM
We now need to customize the VM, do the following in each of the tabs described below and hit "Apply" before moving to the next tab:
- Overview tab: switch "Firmware" from "BIOS" to "UEFI".
- CPUs tab: Adjust your CPU config to use all cores you can check your CPU with: ```lscpu | egrep 'Model name|Socket|Thread|NUMA|CPU\(s\)'```, the total number in "vCPU allocation" should match "Logical host CPUs" value.
- Memory tab: Adjust your VMs memory here, a good value is half of your system's total RAM. So if you have 32GB then assign 16GB to your VM.
- Boot Options tab: Place a check mark near "SATA CDROM" and move it up so its the first device.
- VirtIO Disk1 tab: In "Advanced options" change "Cache mode" to "writeback".
- NIC tab: Make sure that "Device model" is "virtio"
- **Windows VM only**: click on the "Add Hardware" button (down below). Select "Storage" from the list and switch "Device type" to "CDROM device". Click on "Manage" then click on "Browser Local" and navigate to the "virtio-win..." ISO we downloaded earlier. This would be needed during the Windows Install process to help discover the hard drive.

**NOTE** if installing Windows 11 you will need to add a TPM module. Before you can so that you will need to install the ```swtpm``` package with 
```
sudo pacman -S swtpm
```
You can then add a TPM from the menu with settings CRB and version 2.0

With all the options set you can now click on "Begin Installation" on the top. This would kick off the OS install process.

### OS Install
On Linux the OS install should be able to complete with out any additional steps. You would just go through the process as you normally would. 

On Windows there are a few steps needed to install the virtio drivers.
- During the Windows Setup you will be prompted with "Where do you want to install Windows?". Typically a drive would be present here but with VIRTIO the list would be empty.
- Click "Load driver"
- Select the drive with the virtio-win ISO
- navigate to the "amd64" folder and ten to your Windows version folder (w11 for example)
- You would now see the virtio driver in the list, select it and click "Next".
- You should now see your VM drive so you can use it as an install target.

Once Windows had installed and booted to the desktop you would also want to install the full virtio driver package.
- Open File Explorer, your virtio-win ISO should be present.
- Open the drive and scroll to "virtio-win-gt-x64" and execute it.
- Follow the prompt accepting the default values.
This step would make all the devices appear and work as they should.

Confirm your VM can reach the internet
We can now power down our OS so we can modify the VM config to include the GPU

## VM Config - Post OS Install
With your VM powered off remove the following items from the config:
- Any "USB Redirector"
- Display Spice
- Video QXL

If you are unable to remove all of the above via the GUI then you will need to edit the XML and remove the following:
```
  <graphics type="spice" autoport="yes">
    <listen type="address"/>
    <gl enable="no"/>
  </graphics>
```
```
  <audio id="1" type="none"/>
```
```
  <video>
    <model type="bochs" vram="16384" heads="1" primary="yes"/>
    <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
  </video>
```
```
  <channel type="spicevmc">
    <target type="virtio" name="com.redhat.spice.0"/>
    <address type="virtio-serial" controller="0" bus="0" port="1"/>
  </channel>
```

### Add the GPU
Now we are ready to add the GPU and to the VM
- click on "Add Hardware".
- select "PCI Host Device".
- Add every item in your GPU group as we discovered earlier.

If Nvidia GPU, we need to add the ROM file to teh XML:
- select your primary GPU PCI device
- click on XML
- Below the "```</source>```" add a new line and enter ```<rom file='/usr/share/vgabios/vbios_patched.rom'/>```
- click apply.


### Add USB Devices
Do the following for any devices you would like to pass to your VM. This can be USB Keyboard, USB Mice, USB Headphones, USB game controllers etc.
- Click on "Add Hardware".
- Select "USB Host Device".
- Select your USB wired device from the list.

**DO NOT Boot the VM just yet**

## Setup VM hook
Now we need to go back and setup the hook so it will kick off the scripts for the GPU disconnect from the host to the VM.
```
sudo nvim /etc/libvirt/hooks/qemu
```
Edit the line that reads ```if [[ $OBJECT == "somevm" ]]; then``` replacing "somevm" with the name of your new VM. If you have multiples you can add them in with the ```||``` (or operator), for example:
```
if [[ $OBJECT == "vm1" || $OBJECT == "vm2" ]]; then
```

## boot the VM
You can now boot your VM. On Linux things would just work out of the box. On Windows the VM would boot and would give you a black screen at first but after some time Windows would reach out to the internet and auto download and install the Nvidia driver which at that point you would get the desktop.

Congratulations!!