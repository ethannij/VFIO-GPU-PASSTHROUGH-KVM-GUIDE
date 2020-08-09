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

# Windows
