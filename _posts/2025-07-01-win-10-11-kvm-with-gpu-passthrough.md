---
title: Create a Win10/11 KVM virtual machine with GPU passthrough
date: 2025-07-01 00:04:00 +/-0800
categories: [writing, linux]
tags: [win10, qemu, kvm, libvirt, pcie, tutorial]
author: mwertman
---

## Introduction
In this post, I will be giving detailed instructions on how to setup a Windows 10/11 KVM-based VM (Virtual Machine) with GPU passthrough.

This guide includes how to setup your GPU with VFIO, creating the Win10 or Win11 Virtual machine within VMM (Virtual Machine Manager), configuring the VM for PCIe/GPU passthrough, and setting up the VM with the [Looking Glass](https://looking-glass.io/) client.

## Why?
This guide is for people wanting to achieve a high-performing local Windows VM on a Linux host without the need of dual-booting.

### Quick aside: Gaming on Linux
Gaming on Linux has been a real pain point in the past. But withing the last few years, it has gotten way better with the help of Valve's [Proton](https://github.com/ValveSoftware/Proton) compatability layer for Steam. However, even still, some games do require windows as the only way to play on PC. For example, online games that utilize Anti-Cheat software or DRM.

This is the main reason I wanted a solution like this without the need to dual-boot or paying for a cloud provider subscription. Having something like this allows me to seemlessly switch between my Linux host and Windows guest and have very-comparable performance to running native Windows.

This comes with a cost of it's own, unfortually. This setup is quite lengthy and tedious and may have to adjust some steps to get it working for you. To get a setup like this working for my system took a lot of trail and error, but hopefully this guide can aid to avoiding some of the pitfalls that occur in this process.

## Resources
Here is some further reading material that I have used to make this guide.

 - Arch Wiki [PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
 - bryansteiner's guide [gpu-passthrough-tutorial](https://github.com/bryansteiner/gpu-passthrough-tutorial/blob/master/README.md)
 - [Looking Glass Docs](https://looking-glass.io/docs/B7/)

## Hardware Requirements
 - Two GPUs (one guest GPU dedicated to the VM)
 - CPU that supports hardware virtualization (Intel VT-x/VT-d or AMD-Vi) See the [Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Prerequisites) for more detailed list of CPUs
 - Motherboard that supports [IOMMU](https://en.wikipedia.org/wiki/List_of_IOMMU-supporting_hardware)
 - Guest GPU ROM must support UEFI. Can find your GPU make/model [here](https://www.techpowerup.com/vgabios/) and verify
 - Monitor with multiple inputs or multiple monitors

## Looking Glass peripherals
 - Secondary keyboard and mouse
 - HMDI cable or optionally HDMI dummy adapter

## Hardware Setup
My current configuration is as follows:
 - CPU
   - AMD Ryzen 7 5700X
 - Motherboard
   - MSI MPG B550
 - GPUs
   - NVIDIA RTX 3070 (Guest Card)
   - AMD Radeon RX 6800 (Host Card)
 - RAM
   - 64GB DDR4 (4x16)

OS: Pop!_OS 22.04

Kernel: 6.12.10-76061203-generic

## Getting started
Firstly, lets get some needed dependencies that we will need.
```bash
sudo apt install libvirt-daemon-system libvirt-clients qemu-kvm qemu-utils qemu ovmf bridge-utils virt-manager virt-viewer libosinfo-bin -y
```
After installing the packages, reboot your system into the BIOS to enable 'VT-d' or "Virtualization Technology" for Intel CPUs or 'AMD-Vi' for AMD if you haven't already.

To ensure that CPU virtualization is enabled:
```bash
dmesg | grep VT-d    # for Intel
dmesg | grep AMD-Vi  # for AMD
```

Secondly verify that IOMMU is enabled:
```bash
dmesg | grep -i IOMMU
```
Should look something like this if enabled
```
[    0.000000] ACPI: DMAR 0x00000000BDCB1CB0 0000B8 (v01 INTEL  BDW      00000001 INTL 00000001)
[    0.000000] Intel-IOMMU: enabled
[    0.028879] dmar: IOMMU 0: reg_base_addr fed90000 ver 1:0 cap c0000020660462 ecap f0101a
[    0.028883] dmar: IOMMU 1: reg_base_addr fed91000 ver 1:0 cap d2008c20660462 ecap f010da
[    0.028950] IOAPIC id 8 under DRHD base  0xfed91000 IOMMU 1
[    0.536212] DMAR: No ATSR found
[    0.536229] IOMMU 0 0xfed90000: using Queued invalidation
[    0.536230] IOMMU 1 0xfed91000: using Queued invalidation
[    0.536231] IOMMU: Setting RMRR:
[    0.536241] IOMMU: Setting identity map for device 0000:00:02.0 [0xbf000000 - 0xcf1fffff]
[    0.537490] IOMMU: Setting identity map for device 0000:00:14.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537512] IOMMU: Setting identity map for device 0000:00:1a.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537530] IOMMU: Setting identity map for device 0000:00:1d.0 [0xbdea8000 - 0xbdeb6fff]
[    0.537543] IOMMU: Prepare 0-16MiB unity mapping for LPC
[    0.537549] IOMMU: Setting identity map for device 0000:00:1f.0 [0x0 - 0xffffff]
[    2.182790] [drm] DMAR active, disabling use of stolen memory
```

### Force setting IOMMU in kernel parameters
To ensure that IOMMU is enabled *everytime*, you can set in your kernel parameters.
For AMD users, this is not necessary as it is automatically enabled if your hardware supports it.

In my case, I am using systemd as my boot loader:
```bash
sudo kernelstub --add-optinos "intel_iommu=on"  # for Intel
sudo kernelstub --add-options "amd_iommu=on"    # for AMD
```

If you are on GRUB, you can set it by editing `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on"  # for Intel
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on"    # for AMD
```

### Finding IOMMU Groups
An IOMMU group is the smallest set of devices that can be passed into the VM. Meaning all devices within the group have to be passed into the VM.
This can be a little bit tricky, but you want your Guest GPU that you are gonna pass in to be into its own isolated group.

To see how your IOMMU groups are mapped out currently you can run the following bash script:
```bash
#!/bin/bash
##
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```
{: file="iommu_groups"}

Example output:
```
IOMMU Group 15 04:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Upstream Port of PCI Express Switch [1002:1478] (rev c3)
IOMMU Group 15 05:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Downstream Port of PCI Express Switch [1002:1479]
IOMMU Group 15 06:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 [Radeon RX 6800/6800 XT / 6900 XT] [1002:73bf] (rev c3)
IOMMU Group 15 06:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
IOMMU Group 15 2a:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
IOMMU Group 16 2b:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3070] [10de:2484] (rev a1)
IOMMU Group 16 2b:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
```

If your Guest GPU isnt isolated in its own group. You will need to figure out a way to separate it out.
 1. If your Guest GPU is not in the top slot of your motherboard, you can try swapping it with your Host GPU. This may vary the grouping so that your Guest GPU is in it's own group.
 2. If you have any other PCIe devices installed that are attached to that same slot of your Guest GPU (i.e an NVME drive). you can try moving that device to another slot.
 3. An alternative solution is [ACS override patch](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)). Can also check out [this post](https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html) by Alex Williamson. BEWARE as this comes with some [risks](https://www.reddit.com/r/VFIO/comments/bvif8d/official_reason_why_acs_override_patch_is_not_in/).

For my motherboard, I had to swap my GPUs around to get the Guest GPU into its own group.

If you are able to isolate your GPU, like in the example above. You can continue to the next step!

## Binding VFIO drivers to the Guest GPU
There are many ways to achieve this. You can dynamically bind and unbind the VFIO drivers via [libvirt hooks](https://www.libvirt.org/hooks.html) if you still want to utilize your Guest GPU when the VM is not running. See [bryansteiner's guide](https://github.com/bryansteiner/gpu-passthrough-tutorial?tab=readme-ov-file#----part-2-vm-logistics) for an example setup.
I havent had much luck with this however, but your mileage may vary. For this guide, I am simply going to disable/uninstall the nvidia drivers and early-load the VFIO drivers as my Guest GPU will be dedicated to the VM.

### Blacklisting Nvidia drivers
```bash
sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
```

### Uninstalling the Nvidia drivers
Again, this setup is not necessary. But, in my case, I have no need for them.
```bash
sudo apt-get remove --purge  '^nvidia-.*'
```

### Binding the VFIO drivers
To bind the guest GPU to the VFIO drivers, you will need the PCI vendor ID for *everything* within the IOMMU group.
```bash
lspci -nn
```

Example output:
```
05:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 XL Downstream Port of PCI Express Switch [1002:1479]
06:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 [Radeon RX 6800/6800 XT / 6900 XT] [1002:73bf] (rev c3)
06:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 21 HDMI Audio [Radeon RX 6800/6800 XT / 6900 XT] [1002:ab28]
2a:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
2b:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3070] [10de:2484] (rev a1)
2b:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
2c:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Function [1022:148a]
2d:00.0 Non-Essential Instrumentation [1300]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Reserved SPP [1022:1485]
2d:00.1 Encryption controller [1080]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse Cryptographic Coprocessor PSPCPP [1022:1486]
2d:00.3 USB controller [0c03]: Advanced Micro Devices, Inc. [AMD] Matisse USB 3.0 Host Controller [1022:149c]
2d:00.4 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse HD Audio Controller [1022:1487]
```

My guest GPU is busid `2b:00.0` and `2b:00.1` which in the table above shows the vendor ids in the square brackets.
So `10de:2484` and `10de:228b`

After you have the vendor IDs, you can set the kernel parameter to ensure that they bind after every boot:
```bash
sudo kernelstub --add-option vfio-pci.ids=10de:2484,10de:228b
```

Finally, reboot your system to start creating the VM!
