# Setting up virtual machine

We need to install some packages:

```Bash
sudo pacman -S qemu-desktop dnsmasq virt-manager ebtables libvirt edk2-ovmf swtpm
```

We will need to activate the default NAT network:

```Bash
virsh net-autostart default
virsh net-start default
```

The process of setting up a virtual machine using virt-manager is mostly self-explanatory, as most of the process comes with fairly comprehensive on-screen instructions.

However, you should pay special attention to the following steps:

- When the virtual machine creation wizard asks you to name your virtual machine (**final step before clicking "Finish"**), check the **"Customize before install" checkbox**.
- Select the BIOS and search for the UEFI with secureboot for our x64 platform.
- In the "CPUs" section, adjust your socket, cores and threads per core, make sure to verify you are sending the correct setup.

Since we are installing Windows 11, we need also a TPM2.
Since we installed _swtpm_, we can "Add Hardware" and we should see a TPM2 Emulator.

(Check also if you want to use virtio for your main disk now)

Then we are ready to install the OS as normal.