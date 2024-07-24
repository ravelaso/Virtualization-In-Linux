# VM Configuration

Now we are going to configure the VM.

First of all, shut down your VM!

We need to set up Virt-Manager under Settings or Preferences, to allow us to edit XML.


## Mouse & Keyboard (evdev)

We are going to tackle the mouse and keyboard.
Right now we have Spice and a virtual monitor to us, which captures the mouse and keyboard, 
however is best to send them using evdev.

Firs we need to get the devices ID of our mouse and keyboard:

```Bash
ls -l /dev/input/by-id/
```

You will get something like this:

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

Im going to use my MonsGeek keyboard and my Razer Viper mouse. For that we need to look at the event name of each one.
In my case they are:

```Bash
usb-Razer_Razer_Viper_Ultimate_000000000000-event-mouse
usb-monsgeek_MonsGeek_Keyboard-event-kbd
```

**IMPORTANT**

Some devices will have other event like my keyboard has: **usb-monsgeek_MonsGeek_Keyboard-if02-event-kbd**

Sometimes when adding these devices, and we want to get back the mouse and keyboard to the host, it will only pass the keyboard and not the mouse.

For that, we need to experiment, usually it is the if02 event of the keyboard.
In my case, I need it, so I will use it from now on.

#### Configure the VM XML file.

Now that we have our devices, we will edit the VM using xml. (We can use a command or the virt-manager editor)

```Bash
sudo EDITOR=nano virsh edit nameofyourvm
```

We will add this configuration:

```xml
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

You can see my if02-event-kbd has the options grab grabToggle and repeat, this is the device which recognized better in my machine.
Instead of the regular -event-kbd. For you it might work with the regular one or you might have to use one of the if02.

In either case, you need to also pass the default -event-kbd if you needed the if02 as you see in my example.


#### Check

If we boot the machine now, the VM will take instantly control over our mouse and keyboard, if you press ``CTRL + CTRL``, both CTRL buttons on your keyboard, the mouse and keyboard should respond back in the host. If that is not the case, please check the step before.


## GPU pass-through

This is actually an easy one, we just isolated the graphics card, so we can just go over the GUI (virt-manager), click on 
'Add Hardware' and select "PCI Host Device", we should find our NVIDIA VGA and its Audio Device, we need to add both.

Now the VM will have direct access to it.


## Disable Virtual Monitor

In my setup I use a DisplayPort cable connected to my monitor second input, 
so I do not need to see or have a screen sharing of the machine.

Therefore, I will disable spice, and the monitor (virtual one).
For that, we need to remove these entries from the XML all at once.

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

Save the file and start your machine again, 
nothing will display but your external monitor should show you still the DisplayPort signal, with only that monitor. (Physical)

## Include Virtio Disks

If you have installed Virtio drivers from redhat / fedora binaries to your window's machine. Check [Wiki Virtio](https://wiki.archlinux.org/title/QEMU#Installing_virtio_drivers)
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

If you get an error about io threads add also this line:

```xml
<iothreads>1</iothreads>
```


## Get Audio from Guest to your Host

If we want to hear the audio from the guest to the host, we can use PipeWire.

We can add this to our XML config:

```xml
<audio id="1" type="pipewire" runtimeDir="/run/user/1000">
    <input name="qemuinput"/>
    <output name="qemuoutput"/>
</audio>
```

Of course make sure you are user 1000 or set the user yourself.








