# Getting Started

This guide will take you through my experiences with virtualization on Arch Linux, and by the end, you will have a fully functioning Windows 11 Virtual Machine (VM) with GPU NVIDIA pass-through, seamless mouse and keyboard capture.

The tools we'll be using for this setup are **KVM / QEMU** as the virtualization platform, **libvirt** as the management API, and **virt-manager** for the user interface.

For audio, my Arch Linux utilizes PipeWire, which allows us to send guest audio back to the host without needing a virtual monitor attachment.

Please note that we won't be using **Spice** or **Looking Glass**, so you will need a monitor connected to your PCI GPU.

## Prerequisites

Before starting, please ensure the following requirements are met:

- Your CPU and Motherboard (MB) must support Virtualization / IOMMU.
- You should have at least two GPUs. An internal GPU from an Intel/AMD CPU chip (APU) can work as one of them.

This guide also references the [Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF), so you might find additional helpful information there.

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

Add Intel / AMD iommu set to on:

```Bash
 GRUB_CMDLINE_LINUX_DEFAULT=" ... intel_iommu=on ..."
```

Then update grub:

```Bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


## 4. Load Missing Modules

If any modules didn't load correctly during the IOMMU check, you'll need to load those missing modules manually.

To do this, create a new file ``/etc/modules-load.d/vfio.conf``.

In this file, you should list the names of the missing modules. This will ensure that these modules are loaded at boot time. The exact names of the modules will depend on your specific situation.

```Bash
sudo nano /etc/modules-load.d/vfio.conf


# Write this to file:

vfio
vfio_iommu_type1
vfio_pci

# In case of virtio, also:

virtio
virtio_blk
virtio_pci
virtio_net

```


## 5. GPU Isolation

The next step in setting up your virtual machine is to isolate the GPU, as we'll be using it for pass-through.

### Identifying GPU IDs

To identify your GPU IDs, use the following command:

```Bash
lspci -nn
```

The output will contain a wealth of information, but we're only interested in the NVIDIA Graphics and its associated Audio device. For example:

```Bash
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD103 [GeForce RTX 4080] [10de:2704] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)
```

### Configuring VFIO

Once you've identified the IDs, create a new file `/etc/modprobe.d/vfio.conf` and add the following lines:

```Bash
options vfio-pci ids=10de:2704,10de:22bb
softdep nvidia pre: vfio-pci
```

Replace `10de:2704` and `10de:22bb` with the IDs of your VGA and Audio device, respectively, separated by a comma.

The `softdep` line ensures that we isolate the GPU before the driver loads.



## 6. Post-Reboot Validation

After rebooting your system, it's crucial to verify that the changes you've made are correctly implemented.

## IOMMU Group Check

We'll start with a script to check the IOMMU groups:

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

Running this script will help you determine if your GPU is in an isolated group. In my case, it's in group 13.

```Bash
IOMMU Group 13:
      01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD103 [GeForce RTX 4080] [10de:2704] (rev a1)
      01:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:22bb] (rev a1)
```

## Driver Check

Next, let's verify which driver is handling our NVIDIA device:

```Bash
lspci -nnk
```

The output should show that the `vfio-pci` driver is in use:

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

With these validations complete, you can be confident that your GPU is ready for pass-through!

You're now ready to proceed with setting up your virtual machine.









