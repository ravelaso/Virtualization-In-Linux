# Setting up virtual machine

This guide provides the steps for setting up a virtual machine on your system. 
Please follow the instructions carefully to ensure a successful setup.

## Package Installation

Begin by installing the necessary packages using the following command:


```Bash
sudo pacman -S qemu-desktop dnsmasq virt-manager ebtables libvirt edk2-ovmf swtpm
```

## Network Configuration

Next, activate the default NAT network with the following commands:

```Bash
virsh net-autostart default
virsh net-start default     
```

## Virtual Machine Setup

Setting up a virtual machine using `virt-manager` is generally straightforward due to comprehensive on-screen instructions. However, you need to pay close attention to the following steps:

(Note: If you plan to use virtio for your main disk, configure it at this point, [Virtio](https://wiki.archlinux.org/title/QEMU#Installing_virtio_drivers))

1. During the virtual machine creation process, you will be prompted to give a name to your virtual machine. At this stage (**the final step before clicking "Finish"**), ensure to tick the **"Customize before install" checkbox**

2. Next, select the BIOS and search for the UEFI with secure-boot for our x64 platform (Should be a .fd file)

3. In the "CPUs" section, carefully adjust your socket, cores and threads per core. Ensure the settings are correct before proceeding

## Windows 11 Installation

Since we are installing Windows 11, it is important to note that TPM2 is required. Thankfully, we have already installed the \`swtpm\` package for this purpose. In the "Add Hardware" section, you should find a TPM2 Emulator.

Now you're all set to install the operating system as usual.