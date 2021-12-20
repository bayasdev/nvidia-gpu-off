# 👀 Checkout the new [EnvyControl](https://github.com/geminis3/EnvyControl) tool 🚀🚀🚀
It provides a much simpler way to switch between GPU modes on Optimus Laptops under Linux, as well as turning off the Nvidia dGPU using an alternative approach that consists of removing the card from the PCI bus with Udev rules.

_________________

# nvidia-gpu-off
The definitive guide to completely turn off your Nvidia dedicated GPU and thus doubling your laptop's battery life on Linux.

## Preface
Nvidia GPUs on Linux have always been a headache due to their propietary drivers ([obligatory reference?](https://www.youtube.com/watch?v=_36yNWw_07g))  but this problem is exacerbated on laptops due to **inexistent dynamic power management** technologies on pre-Turing cards paired with Intel processors older than Coffee Lake ([official Nvidia documentation on dynamic PM for Turing and newer cards](http://us.download.nvidia.com/XFree86/Linux-x86_64/465.31/README/dynamicpowermanagement.html)).

## Why `nouveau` or `bbswitch` are NOT an alternative?

- The open-source `nouveau` driver built into the Linux kernel provides adequate dynamic PM and PRIME offloading functionality however due to restrictions imposed by Nvidia **it can't control the graphics card clock speeds** providing mediocre performance and it **may cause software incompatibility issues** (in my laptop opening GNOME settings app when running `nouveau` causes the entire desktop to hang for almost 5 seconds)

- `bbswitch` hasn't been updated since 2013 and is completely **broken on some systems**.

## Requirements

 - Having an Optimus laptop with a pre-Turing Nvidia graphics card.
 - This guide was designed for Linux distributions using `systemd` as their init system.

### Limitations
**Your HDMI ports won't work** if they're wired to the Nvidia GPU so please keep this in mind, however you can always reverse the procedures described in this guide.

## Let's go!

### 1) Get rid of propietary Nvidia drivers

Please refer to your distribution's documentation to remove the Nvidia drivers however on Ubuntu you can do:

    sudo apt-get autoremove --purge *nvidia*

Then reboot your system.

### 2) Get to know your PCI busses
Before proceeding with this guide we need to know which are the PCI busses employed by the Nvidia card, please run: `lspci | grep NVIDIA`

In my laptop this was the output:

    01:00.0 VGA compatible controller: NVIDIA Corporation GP106M [GeForce GTX 1060 Mobile]
    01:00.1 Audio device: NVIDIA Corporation GP106 High Definition Audio Controller

As you can see `01:00.0` belongs to the GPU itself and `01:00.1` to the GPU's audio chipset, let's take note of those PCI busses.

### 3.1) Install `acpi-call-dkms`

On Ubuntu, Debian and friends do:

    sudo apt install acpi-call-dkms

On Arch Linux and friends this package is called `acpi_call-dkms` so do:

    sudo pacman -S acpi_call-dkms

If your distro is not listed please refer to their respective documentation nor repositories.

### 3.2) Find an ACPI call that works for you

Before proceeding we need to:

    sudo modprobe acpi_call

On Ubuntu, Debian and friends please run `sudo /usr/share/doc/acpi-call-dkms/examples/turn_off_gpu.sh`

For Arch Linux and friends run `sudo /usr/share/acpi_call/examples/turn_off_gpu.sh`

This was my output:

    Trying \_SB.PCI0.P0P1.VGA._OFF: failed
    Trying \_SB.PCI0.P0P2.VGA._OFF: failed
    Trying \_SB_.PCI0.OVGA.ATPX: failed
    Trying \_SB_.PCI0.OVGA.XTPX: failed
    Trying \_SB.PCI0.P0P3.PEGP._OFF: failed
    Trying \_SB.PCI0.P0P2.PEGP._OFF: failed
    Trying \_SB.PCI0.P0P1.PEGP._OFF: failed
    Trying \_SB.PCI0.MXR0.MXM0._OFF: failed
    Trying \_SB.PCI0.PEG1.GFX0._OFF: failed
    Trying \_SB.PCI0.PEG0.GFX0.DOFF: failed
    Trying \_SB.PCI0.PEG1.GFX0.DOFF: failed
    Trying \_SB.PCI0.PEG0.PEGP._OFF: works!
    Trying \_SB.PCI0.XVR0.Z01I.DGOF: failed
    Trying \_SB.PCI0.PEGR.GFX0._OFF: failed
    Trying \_SB.PCI0.PEG.VID._OFF: failed
    Trying \_SB.PCI0.PEG0.VID._OFF: failed
    Trying \_SB.PCI0.P0P2.DGPU._OFF: failed
    Trying \_SB.PCI0.P0P4.DGPU.DOFF: failed
    Trying \_SB.PCI0.IXVE.IGPU.DGOF: failed
    Trying \_SB.PCI0.RP00.VGA._PS3: failed
    Trying \_SB.PCI0.RP00.VGA.P3MO: failed
    Trying \_SB.PCI0.GFX0.DSM._T_0: failed
    Trying \_SB.PCI0.LPC.EC.PUBS._OFF: failed
    Trying \_SB.PCI0.P0P2.NVID._OFF: failed
    Trying \_SB.PCI0.P0P2.VGA.PX02: failed
    Trying \_SB_.PCI0.PEGP.DGFX._OFF: failed
    Trying \_SB_.PCI0.VGA.PX02: failed

As you can see `\_SB.PCI0.PEG0.PEGP._OFF` was the right ACPI call for my laptop so please take note of yours.

### 4) Blacklist `nouveau` and load `acpi-call-dkms`

Edit the `/etc/modprobe.d/blacklist.conf` and append the following at the end of the file:

    blacklist nouveau

Edit `/etc/modules` and append `acpi_call` at the end of the file.

### 5) Turn off your GPU at startup

For autostart we'll be using [systemd-tmpfiles](https://www.freedesktop.org/software/systemd/man/tmpfiles.d.html).

Edit `/etc/tmpfiles.d/acpi_call.conf` and append:

    w /proc/acpi/call - - - - \\_SB.PCI0.PEG0.PEGP._OFF

Please replace `\_SB.PCI0.PEG0.PEGP._OFF` with the ACPI call you got on step 3.2 but don't forget to escape the backslash.

Now rebuild your initrams with `sudo update-initramfs -u -k all` and reboot.

### 6.1) Try to remove those PCI busses

After rebooting your Nvidia GPU should have been turned off by the ACPI call but the failure to remove its PCI busses may lead to stability issues so do the following for each one the busses you got on step 2.

    echo 1 > /sys/bus/pci/devices/[PCI bus here]/remove

For my laptop I had to do:

    sudo su
    echo 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
    echo 1 > /sys/bus/pci/devices/0000\:01\:00.1/remove

**Be careful with backslashes.**

Now you can re-run `lspci` to check that NVIDIA is no longer present on our system.

### 6.2) Remove PCI busses at startup

Edit `/etc/tmpfiles.d/remove_gpu_from_lspci.conf` and append the following for each one of your Nvidia busses:

    w /sys/bus/pci/devices/[PCI bus here]/remove - - - - 1

For my laptop it was:

    w /sys/bus/pci/devices/0000\:01\:00.0/remove - - - - 1
    w /sys/bus/pci/devices/0000\:01\:00.1/remove - - - - 1

As a precautionary step I rebuilt my initramfs before rebooting again.

## Results

My laptop's power consumption went down from 30W watching a Youtube video with propietary drivers on-demand mode to just 10W after following this guide according to `powertop`.

I hope this guide may help you :D
