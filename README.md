# VFIO-GPU-PASSTHROUGH-KVM-GUIDE

## Introduction
**What is VFIO?**
When virtualizing an operating system like linux or windows, the main drawback is the lack of a proper graphics driver to bridge the gpu in your computer, to what the virtual machine guest can interact with. VFIO solves the problem by hijacking your graphics card, and reserving it for the virtual machine to take hold of, providing near native gpu performance.

**What is KVM?**
KVM is a hypervisor built into the linux kernel, similar to Oracle Virtualbox, or VMware. The difference being, the hypervisor being built into the linux kernel allows for higher performance, and better compatibility.

## Prerequisites
