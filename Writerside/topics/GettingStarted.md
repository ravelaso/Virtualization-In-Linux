# Getting Started

Here is what I've learned regarding Virtualization in Arch Linux (btw)

At the end of this guide we will have a nice Windows 11 VM with pass-through GPU NVIDIA, 
and seamless capture of mouse and keyboard.

We will use **KVM / QEMU** with **libvirt** and our GUI will be **virt-manager**

Arch Linux uses PipeWire for audio 
(this we'll use for sending guest audio back to host, without the needs of a virtual monitor attached)

I won't use **Spice** or **Looking Glass**, therefore you will need to use a monitor hooked into your PCI GPU.

## Before you start

List of the prerequisites that are required or recommended.

Make sure that:

- First: Your CPU and your MB Must be compatible with Virtualization, IOMMU.
- Second: You have at least 2 GPUS, internal GPU from an intel/amd CPU chip (APU) works.

We are going to also follow the reference from this wiki [Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

## 1. Setting up BIOS

First of all you need to go to your BIOS settings and activate: 

   1. # Virtualization Extensions

      ### Intel VT-x (Intel Virtualization Technology for x86)
   
      - What: Allows a CPU to act as if you have multiple independent computers, called virtual machines (VMs).
   
      - How: In BIOS/UEFI, look for options like "Intel Virtualization Technology," "VT-x," or "Intel VT-x," and enable them.
   
      ### AMD-V (AMD Virtualization)
   
      - What: Similar to Intel VT-x, it allows hardware-assisted virtualization on AMD processors.
   
      - How: In BIOS/UEFI, find settings like "SVM Mode" or "Secure Virtual Machine" and enable them.

   2. # IOMMU (Input-Output Memory Management Unit)

      - What: IOMMU maps physical memory to virtual memory, which is critical for device passthrough (e.g., PCIe devices) in virtualized environments. It ensures that VMs can securely and efficiently use hardware devices.
   
      ### Intel: Known as VT-d (Intel Virtualization Technology for Directed I/O).
   
      - How: Enable "VT-d" or "Intel VT for Directed I/O" in the BIOS/UEFI settings.
   
      ### AMD: Often referred to simply as "IOMMU."
   
      - How: Enable "IOMMU" or "AMD-Vi" in the BIOS/UEFI.




## 2. Check IOMMU

Ensure Your Kernel Supports IOMMU and VFIO:

```Bash
LC_ALL=C.UTF-8 lscpu | grep Virtualization
```


See kernel support:

```Bash
zgrep CONFIG_KVM= /proc/config.gz
lsmod | grep kvm
lsmod | grep vfio
lsmod | grep virtio
```

## 3. Configure Grub

We are going to configure GRUB first.

```Bash
sudo nano /etc/default/grub
```

Add intel or amd iommu to on:

```Bash
 GRUB_CMDLINE_LINUX_DEFAULT=" ... intel_iommu=on ..."
```

Then update grub:

```Bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


## 4. Load Missing Modules

In case something was not correctly loaded when checking IOMMU, we are going to load the missing modules.

Create a file /etc/modules-load.d/vfio.conf

And write the modules

```Bash
sudo nano /etc/modules-load.d/vfio.conf

vfio
vfio_iommu_type1
vfio_pci

```


## 5. Isolate the GPU

We need to find the IDs for our GPU, since we are going to pass-through.

```Bash
lspci -nn
```

We will find a lot of information, but we only need to pick both the NVIDIA Graphics and it's Audio.

```Bash
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD103 [GeForce RTX 4080] [10de:2704] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)

```

Now we create a file /etc/modprobe.d/vfio.conf 

```Bash
options vfio-pci ids=10de:2704,10de:22bb
softdep nvidia pre: vfio-pci
```

I use my ids, from both the VGA and the Audio device, split by comma.

The _softdep_ line is to make sure we isolate the GPU before the driver.



## 6. After Reboot.

We can check finally if everything we did is correct.


Here is a nice script to check the IOMMU groups:

```Bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

If we execute this we can find if our GPU is in an isolated group. Mine's in group 13.

```Bash
IOMMU Group 13:
	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD103 [GeForce RTX 4080] [10de:2704] (rev a1)
	01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)

```

Now we can check which driver is taking our NVIDIA.

```Bash
lspci -nnk
```

We should see that the driver in use is vfio-pci:

```Bash
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD103 [GeForce RTX 4080] [10de:2704] (rev a1)
	Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:5110]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)
	Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:5110]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel

```

This is cool, we can pass this GPU without issues!

You can continue next to setting up the virtual machine! 







