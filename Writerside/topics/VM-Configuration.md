# VM Configuration

Now that your GPU is ready for pass-through, it's time to configure your virtual machine.

Before starting, make sure your VM is shut down!

You'll need to set up Virt-Manager under Settings or Preferences to enable XML editing.

## Mouse & Keyboard Setup (evdev)

Let's begin with the mouse and keyboard configuration.

Currently, we are using Spice and a virtual monitor (QXL), which captures the mouse and keyboard inputs. However, we'll switch to using evdev, a more consistent method for tracking peripherals.

First, we'll identify the device IDs:

```Bash
ls -l /dev/input/by-id/
```

Your output might look something like this:

```Bash
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Corsair_CORSAIR_HS80_MAX_WIRELESS_Gaming_Receiver_592466F35C4EC86F-event-if03 -> ../event24
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Corsair_LCD_Cap_for_Elite_Capellix_coolers_1416221080004-event-if00 -> ../event22
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Microsoft_Controller_3032363330313135383532323033-event-joystick -> ../event26
lrwxrwxrwx 1 root root  6 Jul 24 19:57 usb-Microsoft_Controller_3032363330313135383532323033-joystick -> ../js0
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-monsgeek_MonsGeek_Keyboard-event-if02 -> ../event15
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-monsgeek_MonsGeek_Keyboard-event-kbd -> ../event14
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-monsgeek_MonsGeek_Keyboard-if02-event-kbd -> ../event17
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Razer_Razer_Viper_Ultimate_000000000000-event-if01 -> ../event20
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Razer_Razer_Viper_Ultimate_000000000000-event-mouse -> ../event18
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Razer_Razer_Viper_Ultimate_000000000000-if01-event-kbd -> ../event19
lrwxrwxrwx 1 root root 10 Jul 24 19:57 usb-Razer_Razer_Viper_Ultimate_000000000000-if02-event-kbd -> ../event21
lrwxrwxrwx 1 root root  9 Jul 24 19:57 usb-Razer_Razer_Viper_Ultimate_000000000000-mouse -> ../mouse0
```

For this guide, we'll use the MonsGeek keyboard and Razer Viper mouse. To do this, we need to identify each device's event name. In our case, they are:

```Bash
usb-Razer_Razer_Viper_Ultimate_000000000000-event-mouse
usb-monsgeek_MonsGeek_Keyboard-event-kbd
```

>**IMPORTANT:**
>
>Some devices might have additional events. For instance, my keyboard has an additional event:
>
>**usb-monsgeek_MonsGeek_Keyboard-if02-event-kbd**
>
>When adding these devices, you may find that only the keyboard is passed back to the host, not the mouse. If this happens, try using the if02 event of the keyboard. In our case, we need it, so we'll use it from now on.


### Modifying the VM XML File

With our devices identified, we can now modify the XML configuration of the VM. You can do this either with a command or using the Virt-Manager editor.

```Bash
sudo EDITOR=nano virsh edit nameofyourvm
```

We're going to add the following configuration:

```XML
...
  <devices>
    ...
      <input type="evdev">
          <source dev="/dev/input/by-id/usb-monsgeek_MonsGeek_Keyboard-if02-event-kbd" grab="all" grabToggle="ctrl-ctrl" repeat="on"/>
      </input>
      <input type="evdev">
          <source dev="/dev/input/by-id/usb-monsgeek_MonsGeek_Keyboard-event-kbd"/>
      </input>
      <input type="evdev">
          <source dev="/dev/input/by-id/usb-Razer_Razer_Viper_Ultimate_000000000000-event-mouse"/>
      </input>
    ...
  </devices>
```

You'll notice that my `if02-event-kbd` device has the `grab`, `grabToggle`, and `repeat` options. 
This is because this device is better recognized on my machine compared to the standard `-event-kbd`. 
You might find that the standard device works for you, or you might need to use one of the `if02` devices.

Regardless, if you needed to use the `if02` device, you also need to pass the default `-event-kbd`, as shown in my example.

### Verification

If you boot the machine now, the VM will immediately take control of your mouse and keyboard.
By pressing `CTRL + CTRL` (both CTRL keys on your keyboard at the same time), the mouse and keyboard should revert back to the host. If this doesn't happen, please revisit the previous step.

## GPU pass-through

Passing through the GPU is a straightforward process, thanks to the previous isolation steps. 
Simply open the GUI (Virt-Manager), click on 'Add Hardware', and select "PCI Host Device". 
Here, you should find your NVIDIA VGA and its Audio Device. Make sure to add both.

With these steps, the VM will now have direct access to the GPU.


## Disable Virtual Monitor

If your setup includes a second input to your monitor via a DisplayPort cable, 
you might not need a virtual monitor or screen sharing for your machine. 
In this case, you can disable Spice and the virtual monitor.

To do this, you'll need to remove the following entries from the XML file:

```xml
...
<audio id="1" type="spice"/>
...
<graphics type="spice" autoport="yes">
    <listen type="address"/>
    <image compression="off"/>
</graphics>
...
<channel type="spicevmc">
    <target type="virtio" name="com.redhat.spice.0"/>
    <address type="virtio-serial" controller="0" bus="0" port="1"/>
</channel>
...
<redirdev bus="usb" type="spicevmc">
    <address type="usb" bus="0" port="1"/>
</redirdev>
...

```

Once these entries are removed, save the file and start your machine again. 
While nothing will display on the virtual monitor, your external monitor should continue to show the DisplayPort signal on the physical monitor.

## Include Virtio Disks

If you have installed Virtio drivers from redhat / fedora binaries to your window's machine. ( Check [Wiki Virtio](https://wiki.archlinux.org/title/QEMU#Installing_virtio_drivers) )
You can hook up any other internal drivers you have to that machine like this:

```xml
<disk type='block' device='disk'>
    <driver name='qemu' type='raw' cache='writeback' io='threads' discard='unmap'/>
    <source dev='/dev/disk/by-id/ata-Samsung_SSD_870_EVO_4TB_S758NX0W502467Z'/>
    <target dev='sda' bus='scsi'/>
</disk>
<controller type='scsi' index='0' model='virtio-scsi'>
    <driver iothread='1' queues='8'/>
</controller>
```

Make sure you identify your correct disk by id:

````Bash
ls -l /dev/disk/by-id/
````

>If you get an error about ``io threads`` after saving the file, add also this line:
>
>```xml
><iothreads>1</iothreads>
>```


## Get Audio from Guest to your Host

If we want to hear the audio from the guest to the host, we can use PipeWire.

We can add this to your XML config:

```xml
<audio id="1" type="pipewire" runtimeDir="/run/user/1000">
    <input name="qemuinput"/>
    <output name="qemuoutput"/>
</audio>
```

Of course make sure you are user 1000 or set the user yourself.








