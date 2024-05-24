# Introduction

This guide is meant to make you able to successfully passthrough the dedicated GPU found on MSI GS65 laptops. It is aimed to regular Linux users who can understand basic stuff of this beautiful OS.

I am using Fedora for this setup, but this may work with other distros. An especially stable result I did obtain was in fact with Debian, but with a customized kernel where both `nouveau` and `vga_switcheroo` modules were removed, so there was no way for the OS to even thinking about interacting with the video card if not with the VFIO drivers.

This guide does indeed aim to "disable" the Nvidia GPU from the whole Linux OS, as it will only be used inside the VM when needed.

Other guides may suggest that this is not needed and you can "switch" between different drivers for the video card, even in these shi\*ty optimus setups. This was never the case for me but may of course work well with another setup.
 
For the normal operativity of the Linux OS, you will use the integrated Intel graphics.

# Model

My laptop model is a **GS65-8RF** Stealth Thin. (I guess only the 8RF part is the needed one to describe it).

This guide may or may not work for other models.

# Drawbacks

After following the full procedure you will not be able to use the Nvidia card on Linux along with its directly connected ports, which are both HDMI and DisplayPort. The only way for you to connect an external monitor for Linux, will be through a USB-C to HDMI adapter, as the connection to it will go through the CPU.

The HDMI/DP ports will be used for displaying the content of the Virtual Machine. G-Sync will be fully supported.

# Getting started

The first thing to do is, on a Fedora distro, to disable the `nouveau` driver and enable the `vfio` driver, along with enabling IOMMU (VT-d) on the BIOS.

For achieving the first part you will need to modify the kernel parameters and to edit the initramfs, so grab your favorite editor and edit `/etc/default/grub` to add:

`vfio-pci.ids=10de:1ba1,1462:1227,10de:10f0 intel_iommu=on iommu=pt  rd.driver.blacklist=nouveau modprobe.blacklist=nouveau`

This will tell the kernel to stop loading `nouveau` (even if we could still have some known issues that will be fixed later), to bind the said devices id to the `vfio` driver, to enable IOMMU and to "tag" only the devices marked for passthrough. This last setting should serve the purpose of not using IOMMU somehow on other peripherals.

The `1462:1227` id is probably not needed, but I had no time to test it.

Once done, create the file `/etc/dracut.conf.d/10-vfio.conf` and add:

```
force_drivers+=" vfio_pci vfio vfio_iommu_type1 "
```

After doing this issue the following commands:

```
dracut --regenerate-all --force
grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

Note: On different distros this WILL change. Please if you aren't on Fedora, follow the Archlinux documentation at the bottom.

Regarding the `vfio-pci-ids` if your model is different, you will need to search them through the `lspci -nnk` command.

For example my output is:

```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104M [GeForce GTX 1070 Mobile] [10de:1ba1] (rev a1)
	Subsystem: Micro-Star International Co., Ltd. [MSI] Device [1462:1227]

01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
	Kernel driver in use: snd_hda_intel
```

Your video card must also be "all" under the same IOMMU group. I say all because its components may live in different IOMMU groups, even if this should not be the case on this machine.

"An IOMMU group is the smallest set of physical devices that can be passed to a virtual machine."

For showing all the IOMMU groups with their peripherals simply run this script from the Archlinux Wiki:

```
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

This is my output:

```
IOMMU Group 2:
	00:01.0 PCI bridge [0604]: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104M [GeForce GTX 1070 Mobile] [10de:1ba1] (rev a1)
	01:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
```

As we can see, all the video card components are under the same IOMMU group.
On top of that we can also see a "PCI bridge" is present on my IOMMU group. This falls under the "*Plugging your guest GPU in an unisolated CPU-based PCIe slot*" section on the Archlinux Wiki in the references of the document.

This is fine and no action is needed.

Now please `reboot` in order to make the changes work.
Note: If you have full disk encryption you may or may not experience some issues while inputting the password. The Archlinux Wiki reports to add the gpu drivers, the intel one in our case, to the initramfs. I personally did not experience any issue at any time during boot.

If you experience any other issues, remember that you can edit the kernel parameters on boot with the `e` key. 

**If you experience freezes go to the Known Issues section down below.**

# Creating the VM

Ok now comes the fun part! jk.

Install `virt-manager` and `libvirt-daemon`, enable the daemon at the system startup, then start `virt-manager` and create a Windows 10 VM (I only tested W10):

Ensure to be using a `qemu://system` connection. If not setup it throug `File` -> `Add Connection` and select `QEMU/KVM` and press `Connect`.

Go to `File` -> `New Virtual Machine` -> `Local install media (ISO etc..)` -> Select the media, specify Windows 10 and setup all the resources. At the end flag `Customize configuration before install` and do the following things:

- Under `Overview`, set `Firmware` to `UEFI`. I was able to make it working also with BIOS, but UEFI is what other guides suggest.
- Go on `Add Hardware` and, under `PCI Host Device`, select ALL the GPU-related devices under the same IOMMU group. I, for example, added the GPU itself and its audio controller. If you can't see the PCI bridge don't worry, it's normal.
- Add at least a USB mouse and keyboard through the `Add Hardware` menu. Note: If you pass them you will not be able to use them on the Linux OS for the whole uptime of the VM.
- Bonus: Setup the VirtIO Disk and Network card for optimal performance. You can find the windows drivers online. This is another guide. I now have to sleep.
- Click back on `Overview` and, under the `XML` tab, add the following at the end of the XML right before `</domain>`:

```
<qemu:override>
    <qemu:device alias="hostdev0">
      <qemu:frontend>
        <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="5218"/>
        <qemu:property name="x-pci-sub-device-id" type="unsigned" value="4647"/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
```

Note the `x-pci-sub-vendor-id` and `x-pci-sub-device-id` values. They are the decimal representation of my PCI subvendor and subdevice id. In hexadecimal, like they are shown on the Windows Hardware app, they are respectively `1462` and `1227`, like the ones passed to VFIO.

They might change in different models. If you don't specify them you will not be able to install the Nvidia drivers later on, as they will be unable to recognize the card. You will in fact find the video card under the control panel, and you will become crazy and search online for Nvidia drivers bugs for days, like I did.

Once done, just click on `Begin installation` and do the classic Windows installation. After that install the Nvidia drivers and you will find yourself  with two video cards inside the VM, one is emulated and the other is your true video card.

Many guides tell you to disable the SPICE server, channel, the emulated card, audio and other stuff. I personally like to have my audio routed through the VM and not through another "passed" USB peripherals and also like to have a debug "channel" (the emulated screen and card) if things don't go as expected. Performances may vary but I honestly did not notice any drop in frame rate.

Now connect the DisplayPort/HDMI cable on the external screen and enjoy your not-so-virtual machine!

# Known issues

## OS Freeze:

There is a little bug with PCI D power states, where the GPU D state gets modified by some power saving mechanisms. When the `nouveau` driver is loaded this problem is never experienced, but when blacklisting the driver it may happen.

You can notice it by watching the dmesg messages speaking about the fact that the peripheral can't go in a D3Cold state or something like that.

For this purpose, you must go in the "hidden" MSI bios, so go under the `Advanced` tab and press, at the same time, `Right ctrl` + `Right shift` + `Left alt` + `F2`, scroll down, enter the `ACPI D3Cold Settings` and enable it.

After doing that you must disable the power management on the specific device via:
`echo on | tee /sys/bus/pci/devices/0000\:01\:00.0/power/control /sys/bus/pci/devices/0000\:01\:00.1/power/control > /dev/null`

This will consume more battery, but I created a simple systemd unit that disables the power management only when the `nouveau` driver is blacklisted. You can find it in the current repository.

You must add it to the `/etc/systemd/system` directory and after that issue:

`systemctl daemon-reload` and
`systemctl enable d0_gpu.service`

and then reboot.

Note: Also here, if you have a different model this is not guaranteed to work.

# References

1. The exceptional guide from the Arch Linux Wiki: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF

# Donations

I know that I (and you) suffered while setting up this, but there are people who suffer more, especially during the last times. Donate to them.
