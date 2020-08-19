# VFIO-GPU-PASSTHROUGH-KVM-GUIDE

## Introduction
**What is VFIO?**
When virtualizing an operating system like linux or windows, the main drawback is the lack of a proper graphics driver to bridge the gpu in your computer, to what the virtual machine guest can interact with. VFIO solves the problem by hijacking your graphics card, and reserving it for the virtual machine to take hold of, providing near native gpu performance.

**What is KVM?**
KVM is a hypervisor built into the linux kernel, similar to Oracle Virtualbox, or VMware. The difference being, the hypervisor being built into the linux kernel allows for higher performance, and better compatibility.

## Prerequisites
Before you begin, you will need the following:

* Two graphics cards: If you want to take advantage of VFIO and give a gpu to your virtual machine, you will need two graphics cards. The gpu you give to your vm will be unusable to your host.
* Linux: You will need some distro of linux, whether it is ubuntu or Arch. This guide will point towards as many resources as possible, but do not be afraid to branch off of this guide to find methods for your specific distro, though I will try my best to give general descriptions of what we are trying to accomplish, and resources you can look at.
* Pulseaudio: I have found ways to make pulseaudio delay free, and high quality for your VM needs

*IMPORTANT*: Please make a backup before following the rest of this guide, there is a chance that your system will be messed up, and it is a lot easier to restore a backup, then fix it.

## What Are We Doing
Before we begin, you should understand the steps, and order of things, so you will not get confused:
* Enable KVM within your kernel: Depending on your distro this may be necessary
* Enable Virtualization: Before we can begin, we must make sure your bios allows virtualization, usually you can find this as (Intel VT-x Intel VT-d) or (AMD-Vi)
* Enable Iommu: We need to enable IOMMU so we can seperate your many devices into groups, in order to isolate them, specifically your graphics card
* Isolating your gpu: Using VFIO we will isolate your gpu so the host system does not take it over, we will do this by making VFIO load before the gpu driver
* Installing neccessary packages: Including qemu, libvirt, edk2-ovmf, and virt-manager
* Creating a bridged network for your host and guest to interact with

We will not be configuring QEMU or libvirt in this part of the guide, I will make subsections for each guest OS (Windows, Mac, Linux).

## Enable KVM
This is enabled by default in most linux distros, but if you are using a custom kernel (gentoo users), you should pay attention. In your kernel, you must enable Kernel-based Virtual Machine (KVM) support, as well as KVM for (Intel or AMD) processsors support.
For a better explaination, please refer to the Kernel section of: https://wiki.gentoo.org/wiki/QEMU#Installation 
If that applies to you, be sure to follow the vhost-net portion as well, since we will be using that for a bridged connection later.

## Enabling IOMMU
This section is best explained by the arch wiki: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_IOMMU
but I will summarize it here.
We need to add a few kernel parameters, how you do this is dependant of your bootloader.
###### If you are using GRUB: https://wiki.archlinux.org/index.php/Kernel_parameters#GRUB
Edit `/etc/default/grub` and navigate to `GRUB_CMDDLINE_LINUX_DEFAULT=` and add `intel_iommu=on iommu=pt` (replace intel_iommu=on with amd_iommu=on if you are using an amd cpu)
Regenerate grub with `grub-mkconfig -o /boot/grub/grub.cfg`

###### To append arguments to the kernel itself
Navigate to `/usr/src/linux`
`make menuconfig`
Navigate to `processor type and features`
Scroll down until you see `Built-in kernel command line` and enable it
add `iommu=pt` and `intel_iommu=on` (or `amd_iommu=on`) save your .config
exit the config
run `make` and `make modules_install` and `make install` and copy your new kernel to your bootloader
This is specific to the bootloader you have chosen
In the case of efibootmgr I copy `/boot/vmlinux-5.4.48-gentoo` to `/boot/efi/boot/bootx64.efi` but this may not apply to you.

For more information: https://wiki.archlinux.org/index.php/Kernel_parameters

Now reboot

If your computer boots without error
Before we celebrate, let's make sure IOMMU is really enabled
`dmesg | grep -i -e DMAR -e IOMMU`
do you see the line `Intel-IOMMU: enabled` or something along those lines for amd?
Hopefully, if not, make sure your cpu supports IOMMU and you correctly followed the prevoius steps
```
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
will print out your IOMMU groups
we are looking for something along the lines of

```
IOMMU Group 12:
        03:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 3GB] [10de:1c02] (rev a1)
        03:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
```
The group containg your gpu (the one you want to use in your VM) should only contain the VGA controller, and audio device. If you have more than that, refer to https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)
Though that will not be covered in this guide

Congratulations! You have successfully enabled IOMMU and your groups are valid, time to move on

## Isolating the GPU with VFIO
In this step we will be using VFIO to isolate your gpu. When this step is complete, your video card should no longer be able to output video to your computer, so be sure you have two GPUs.

With a custom kernel you must enable VFIO manually https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm#VFIO

At this point you should still have your IOMMU groups displayed. Find the IOMMU group, of the video card you want to passthrough, and take not of each devices id 
*Example*: `[10de:1c02] [10de:10f1]`
Be sure to take note of every device in your target IOMMU group, because you must pass all of them to the VM

I recommend using modprobe to interact with VFIO
Create the file `/etc/modprobe.d/vfio.conf`
Now add the following to the file:
`options vfio-pci ids=10de:1c02,10de:10f1` the order is part of it.
Notice how I listed both devices for my gpu, seperated by a comma.

###### For grub:
edit `/etc/mkinitcpio.conf`
Add `MODULES=(... vfio_pci vfio vfio_iommu_type1 vfio_virqfd ...)`
and `HOOKS=(... modconf ...)`
Now to regenerate your initramfs `mkinitcpio -p linux`, instead of typing linux, press tab to see what it corrects to. You want it to be your kernel.

If you are on gentoo (like me), this can be accomplished by correctly enabling kernel modules. See: https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm#VFIO

Now reboot once again. If all went well, you should no longer see input from the graphics card you passed through.
To verify `dmesg | grep -i vfio`, if you see your devices, perfect
If not, also check `lspci -nnk` and find your graphics card
Make sure `Kernel driver in use: vfio-pci`
That means vfio has successfully captured your gpu

If not, and you are on nvidia:
edit `/etc/modprobe.d/nvidia.conf` and add the following lines
```
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
softdep nvidia* pre: vfio-pci
```
Reboot, it should work now

Since you have passed your graphics card to vfio, your vm will be outputted from the passed gpu itself
Be sure to connect it to a monitor, because that is the only way you will be able to see your vm

# Installing Necessary Packages
It's time to configure libvirt
Install qemu, libvirt, edk2-ovmf, and virt-manager

Enable libvirtd.service and virtlogd.socket
###### systemd
`systemctl enable libvirtd.service`
`systemctl enable virtlogd.socket`
`systemctl start libvirtd.service`
`systemctl start virtlogd.socket`

###### openrc
`rc-update add libvirtd default`
`rc-update add virtlogd default`
`rc-service libvirtd start`
`rc-service virtlogd start`

# Creating a Bridged Network
Some distros may not come with a preconfigured Bridged Network

Check:
start virt-manager
select new virtual machine
select an iso
continue selecting forward until you are on Step 5 of 5
is there an error or warning next to Nework Selection?
If yes, refer to https://wiki.gentoo.org/wiki/Network_bridge
If not, you most likely already have one, and you can continue

# Creating Your VM
At this point, you should be ready to create a virtual machine, but there are some important steps to follow in order to get the most out of your vm experience. I will now be splitting the guide by which operating system it pertains to.

# Evdev
I highly recommend using evdev to interact with your vm, instead of using usb passthrough. If for some reason you don't want to use evdev, skip this section.

Add yourself to the `input` group with:
`usermod -a -G input $USER`

`ll /dev/input/by-id` you should see a list of usb devices
*Example*:
```
❯ ll /dev/input/by-id
total 0
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Corsair_Corsair_Gaming_K95_RGB_PLATINUM_Keyboard_0E02F02BAF0798275886B4DBF5001BC5-event-if00 -> ../event23
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Corsair_Corsair_Gaming_K95_RGB_PLATINUM_Keyboard_0E02F02BAF0798275886B4DBF5001BC5-event-kbd -> ../event20
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Corsair_Corsair_Gaming_K95_RGB_PLATINUM_Keyboard_0E02F02BAF0798275886B4DBF5001BC5-event-mouse -> ../event24
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Razer_Razer_Mamba_Wireless_000000000000-event-if01 -> ../event15
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Razer_Razer_Mamba_Wireless_000000000000-event-mouse -> ../event12
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Razer_Razer_Mamba_Wireless_000000000000-if01-event-kbd -> ../event14
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-Razer_Razer_Mamba_Wireless_000000000000-if02-event-kbd -> ../event13
lrwxrwxrwx 1 root root 10 Aug  8 17:08 usb-SteelSeries_SteelSeries_Arctis_7-event-if05 -> ../event19
```
we are looking for your keyboard and mouse specifically
if you are are not sure which device is your keyboard:
`cat /dev/input/by-id/usb-Corsair_Corsair_Gaming_K95_RGB_PLATINUM_Keyboard_0E02F02BAF0798275886B4DBF5001BC5-event-if00` and press buttons on your keyboard, if characters of any sort appear, that is the proper device
the same applies to your mouse

If for any reason you prefer to use event# you can see which event your device is symlinked to, and use `cat /dev/input/even20` your results should be the same

edit `/etc/libvirt/qemu.conf` and append the following:
```
...
user = "<your_user>"
...
cgroup_device_acl = [
    "/dev/kvm",
    "/dev/input/by-id/KEYBOARD_NAME",
    "/dev/input/by-id/MOUSE_NAME",
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet", "/dev/sev"
]
...
```

We will be adding arguments to the xml of your vm later, so for now, you should be fine.

Make sure to download ebtables and dnsmasq

# Windows 10
Setting up a windows vm is quite simple, although making it nearly perfect requires a little bit of configuration.
Start by creating a new virtual machine, and select the iso you would like to use, in this case, you should choose a windows iso. Otherwise, refer to one of the other sections.

Allocate as much memory, cpu, and storage as you would like to your virtual machine
Proceed until you reach the final step, be sure to customize before install
In overview, select Q35 as your chipset, and `/usr/share/qemu/edk2-x86_64-code.fd` as your firmware
In CPUs select `copy host CPU configuration` and set your topology manually. With my 8 core processor, I set `1 socket, 6 cores, 1 thread`.
In Boot options add SATA CDROM 1 above SATA Disk 1, but enable both
In SATA Disk 1 select `advanced options` and change Disk bus: to `VirtIO`
In NIC select `Bridge br0` if it is available, if not, choose one without a warning
In Sound select AC97, this is important for high quality sound

Remove the following:
* Tablet: we are going to replace this
* Display Spice: Not needed with GPU passthrough
* Console: Not needed with GPU passthrough
* Channel spice: Related to Display Spice, and not needed with GPU passthrough
* Video QXL: Not needed with GPU passthrough

Add the following hardware with `+Add Hardware`:
* Storage: select custom storage: `virtio-win=0.1.171.iso` (https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/) and set device type to `CDROM device` and Bus Type: `sata`
* Input `Virtio Keyboard`
* Input `Virtio Tablet`
* PCI Host Device: All devices pertaining to the gpu you passed through

Before we finish, we must make the following adjustments to the XML of our vm
If you would like to edit the XML with virt-manager, be sure to enable `Enable XML Editing` under Edit/Preference/General
Otherwise:
`EDITOR=<preffered editor ex. vim> virsh edit <domain name> Windows10` <- whatever you named your VM
replace `<domain type="kvm">` with `<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">`
In the `<features>` subsection, append:
```
<hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vendor_id state="on" value="whatever"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
```
This pertians to nvidia gpus, for more info about amd see: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF

Scroll to the bottom of the XML, and append the following between `</devices>` and `</domain>`:
```
<qemu:commandline>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on"/>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=unix:/run/user/1000/pulse/native"/>
  </qemu:commandline>
  ```
  
*NOTE*: `/run/user/1000/pulse/native` change 1000 to your user id, which can be found with the `id` command

It is important that you are using pulseaudio for this to work

Now you can begin installation:
If you are using evdev, the vm will grab your keyboard and mouse, press both ctrl keys to switch between host and vm

The gpu you passed through will now be displaying the vm
Progress through the windows installation until you reach the disk screen
You should see the message `We couldn't find any drives. To get storage driver, click Load driver.`
Select load driver and choose `ok`
be sure to select `Red Hat VirtIO SCSI controller (E:\amd64\w10\viostor.inf)` as your driver and click next
Your disk will now appear, and you can continue installation like normal

To set up your audio:
Download realtek ac97 audio driver from this repo

Navigate to settings/Update & Security/Recovery/ and restart now
Choose Troubleshoot/Advanced Options/Startup Settings/ and press restart
Once booted, you should see a list of startup settings
press 7) Disable driver signature enforcement

Now you can navigate to device manager
Select Other devices/Multimedia audio controller and Update Driver
Browse your computer, and select the Vist/Win7 realtek ac97 driver you downloaded (https://www.realtek.com/en/component/zoo/category/pc-audio-codecs-ac-97-audio-codecs-software)
Select Vista64 and when prompted, press install
You will now have high quality audio passed through from your windows vm

To finish up:
navigate back to device manger
under Other devices will be a few PCI devices
right click on them, update driver, and navigate to, and select the virtio cd
It will automatically pick your drivers
repeat for all of the devices with an error

To bypass battle eye anti cheat follow https://github.com/WCharacter/RDTSC-KVM-Handler
Your windows vm is all done. Congratulations!

# Mac OS X
Getting a mac os kvm up and running is not particularly difficult, but making everything work can be a bit of a challenge. For the most part, I will be summarizing: https://passthroughpo.st/new-and-improved-mac-os-tutorial-part-1-the-basics/
If you run into any issues, please refer to their guide.

First of all, If you are using an Nvidia GPU, mac os high sierra is your best bet. Nvidia drivers will not work afterwards. If you are on an amd gpu, use whatever you like.
Make sure you have the following: `qemu python python-pip git`
And then run this command `pip install click request`
This will install python click request, which is recommended

Navigate to the location you would like to store your virtual machine. This will be the permanant spot for its data. Though it will be added to virt-manager

## Basic Installation
`git clone https://github.com/foxlet/macOS-Simple-KVM.git`
`cd macOS-Simple-KVM`
Once inside, you have a choice to make:
Nvidia: `./jumpstart.sh --high-sierra`
Else: `./jumpstart.sh`

Now we are going to create your virtual harddisk. I recommend leaving the name as `MyDisk`, this can be renamed within MacOS
`qemu-img create -f qcow2 MyDisk.qcow2 64G` <- Put whatever size you want

We need to edit the `basic.sh`
add these two lines to the bottom, to add your disk to the vm
```
-drive id=SystemDisk,if=none,file=MyDisk.qcow2 \
-device ide-hd,bus=sata.4,drive=SystemDisk \
```

If you want your Apple ID to work within the VM, you must edit the `-device e1000-82545em` line and replace the `mac=` address
`openssl rand -hex 6 | sed 's/\(..\)/\1:/g; s/:$//'` will generate a new one, you can replace the existing one with

Now you can run `./basic.sh` to start your VM

Once booted, select `Disk Utility`
Select the disk you created, and `Erase`
Now you can exit that menu, and select `Reinstall OS X`
Follow the instructions as usual and you should be booted into MacOSX

## Fixing Resolution
Open the terminal, and type `sudo diskutil list`
Find the efi partition and `sudo diskutil mount diskXsY` in my case `sudo diskutil mount disk2s1`

Open finder, and navigate to the EFI device you just mounted
Navigate EFI/CLOVER/ and edit `config.plist`
Scroll down until you see `<string>1280x720</string>`
Change that to your screen resolution
Now restart the virtual machine and press esc while it's booting
if you are met with a cursor in a command line, type `exit`
You should now be taken to the "bios" of your virtual machine
Navigat to Device Manager
Select OVMF Platform Congfiguration

Change your preferred resolution, to the SAME one you set in your config.plist

## Switching to virt-manager
Download the example XML from: https://github.com/PassthroughPOST/Example-OSX-Virt-Manager

Place OSX.xml in the folder with the rest of your files for OSX
Edit OSX.xml, and replace all instances of `YOURPATH` with the path to your current directory
Now you can `virsh define OSX.xml`
If that doesn't work, try using sudo

Open virt-manager
You should now see OSX as a VM
It is important to not change your CPU topology from the virt-manager gui
Clover will fail to start if you do
Instead, you must edit the XML, either from virt-manger, or with virsh edit

Remove the following:
* Display Spice: Not needed with GPU passthrough
* Video QXL: Not needed with GPU passthrough
* Sound ICH9: This will allow high quality audio once we get the drivers

Change `Controller USB 0` to USB 2
Change NIC to your preffered Network Source

Add the following:
* Input: Generic USB keyboard if it's not already there
* PCI Host device: Add your graphics card's devices

Now we must edit the XML:
Change your cpu topology however you please
Add this to the `qemu:commandline` at the bottom of the xml:
```
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=unix:/run/user/1000/pulse/native"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=mouse1,evdev=/dev/input/by-id/YOUR_MOUSE"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=kbd1,evdev=/dev/input/by-id/YOUR_KEYBOARD,grab_all=on,repeat=on"/>
```

*NOTE*: `/run/user/1000/pulse/native` change 1000 to your user id, which can be found with the `id` command

Start your VM, your keyboard and mouse will be taken over by the vm
press both ctrl keys to switch between host and guest input

## Configuration
We need to get a few drivers, and make a few adjustments to make our vm work well. We are going to start with the graphics driver, and then get the audio driver.

For the graphics driver we are going to navigate to https://github.com/Benjamin-Dobell/nvidia-update

`curl -O https://raw.githubusercontent.com/Benjamin-Dobell/nvidia-update/master/nvidia-update.sh`
`chmod 755 nvidia-update.sh`

Run these commands to download, and set permissions
`./nvidia-update.sh` to install and patch the driver

Now reboot your vm
Upon login, you should notice a difference in performance
You can manually set your refresh rate in your display settings now

Now let's fix our audio
navigate to https://sourceforge.net/projects/voodoohda/

Download the driver, and save it to your desktop
Place the file in System/Library/Extensions
Now reboot, you should have working, and high quality sound
Congrats, you finished! Enjoy your hackintosh kvm

# Linux
Setting up a linux guest in virt-manager is arguably the easiest type to set up

All we need to do, is create a vm, pass through a gpu, and install your graphics driver

In this guide, I will be installing manjaro, but you can subsitute manjaro with any linux distribution you wish to virtualize

Open virt-manager
Select new virtual machine
If you are using an arch based distro, select Arch as the architecture
If you are unsure which type to select, you can choose generic
Be sure to select `Customize configuration before install`

In Overview: Change chipset to Q35, and Firmware to `/usr/share/qemu/edk2-x86_64-code.fd`
In Boot Options: Select SATA CDROM 1, and move it to the top of the boot order
In NIC: Select your bridged network

Remove the following:
* Tablet: we will replace this with virtio
* Display spice: Not needed with GPU Passthrough
* Console: Related to spice
* Channel qemu-ga: Not needed
* Channel spice: Not needed with gpu passthrough
* Video QXL: Not needed with GPU passthrough
* USB Redirector 1: Related to Spice, Not needed
* USB Redirector 2: Related to Spice, Not needed
* RNG /dev/urandom: Not needed

Add the following:
* Input: VirtIO Keyboard
* Input: VirtIO Tablet
* PCI Host Device: Select all of your GPU's devices

Now we must edit the XML:
Replace `<domain type="kvm">` with `<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">`
Between `</devices>` and `</domain>` add:
```
<qemu:commandline>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=unix:/run/user/1000/pulse/native"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=mouse1,evdev=/dev/input/by-id/YOUR_MOUSE"/>
    <qemu:arg value="-object"/>
    <qemu:arg value="input-linux,id=kbd1,evdev=/dev/input/by-id/YOUR_KEYBOARD,grab_all=on,repeat=on"/>
</qemu:commandline>
```

*NOTE*: `/run/user/1000/pulse/native` change 1000 to your user id, which can be found with the `id` command

At this point, you can begin installation
Your keyboard and mouse will be grabbed by the vm, press both ctrl keys to switch input from vm to host

Install your distro how you normally would
After it is installed, you need your graphics driver

This can be done the same way you do it on your host. The package is usually distro specific.
Once you have your graphics driver you will need virtio drivers
Once again, this is distro specific, on Arch linux, the packages is `qemu-guest-agent`
Reboot your linux guest, and your installation is now complete

If you have any issues with your linux guest, please refer to the wiki of your distro
