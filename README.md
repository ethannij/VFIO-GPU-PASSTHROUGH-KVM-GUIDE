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
Edit /etc/default/grub and navigate to "GRUB_CMDDLINE_LINUX_DEFAULT=" and add "intel_iommu=on iommu=pt" (replace intel_iommu=on with amd_iommu=on if you are using an amd cpu)
Regenerate grub with "grub-mkconfig -o /boot/grub/grub.cfg"
