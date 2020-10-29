# Headless Ubuntu Server with GPU passed through to a VM
## References:
* https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF - I don't use Arch
but their Wiki is AWESOME!
* https://gist.github.com/TomFaulkner/389e8e2e9525e11afe2e775355954cdf
* https://www.kernel.org/doc/html/latest/admin-guide/serial-console.html
* https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm
* https://www.reddit.com/r/VFIO/comments/762lt3/problems_when_booting_host_with_efi/
* https://www.youtube.com/watch?v=1IP-h9IKof0&feature=youtu.be

## Cavaets
This process takes over a GPU in your server.  If it only has one GPU, that
makes the host OS completely headless.  99% of the time this won't be an issue
but consider how you will deal with any issues with the host OS booting.

In this guide I configure Grub and the Linux Kernel to output to the host's only
serial port.  I can then connect an old school null-modem cable between this
serial port and the one on my desktop PC (or laptop, USB to serial adapter may
be required) if ever I need "console" access to diagnose boot issues.  When the
server is booted, I can simply connect with SSH as I usually would.

## PCI device IDs of the GPU to pass through
I'm not going to rehash this here.  Look to the Arch wiki: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Isolating_the_GPU

In my examples, I have two devices that are in the same IOMMU group that I need
to pass through.  These are my Nvidia GPU and it's associated HDMI audio device:
```
10de:1c81
10de:0fb9
```

## Configure kernel and GRUB
For me, all of the stumbling blocks I had were solved by adding more options to
either Grub itself or the Linux kernel command line.

My `GRUB_CMDLINE_LINUX_DEFAULT` looks like this:
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt vfio-pci.ids=10de:1c81,10de:0fb9 console=ttyS0,115200 video=efifb:off"
```
* `intel_iommu=on iommu=pt`: enable the IOMMU
* `vfio-pci.ids=10de:1c81,10de:0fb9`: Tell VFIO that it should take control of
these PCI devices
* `console=ttyS0,115200`: Make the primary console for the Linux kernel the
serial port
* `video=efifb:off`:  Dunno but you need it.

Next, tell grub that it should also use the serial port for it's console:
```
GRUB_TERMINAL=serial
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
GRUB_GFXPAYLOAD_LINUX=text
```

`GRUB_GFXPAYLOAD_LINUX=text` turned out to be the solution when I was receiving this error in the guest:
```
nouveau 0000:05:00.0: fifo: fault 00 [READ] at 000000000105a000 engine 05 [BAR2] client 08 [GPC0/PE_2] reason 02 [PTE] on channel -1 [007febf000 unknown]
```
and this error on the host:
```
vfio-pci 0000:00:03.0: BAR 3: cannot reserve [mem 0xf0000000-0xf1ffffff 64bit pref]
```

## VGA Rom (and AppArmor)
During the setup process I attempted to provide the VGA Rom image to the VM.
I'm ultimately not sure if this is necessary.  I was trying to store the rom
file with the VM's disk images but AppArmor kept denying Qemu access to the
file.
```
kernel: audit: type=1400 audit(1568205545.212:40): apparmor="DENIED" operation="open" profile="libvirt-28693a09-dc34-4aba-9b47-1c5c4bbbec45" name="/var/lib/libvirt/machines/neon1.ad.ghanima.net/images/vbios.rom" pid=5916 comm="qemu-system-x86" requested_mask="r" denied_mask="r" fsuid=64055 ouid=1000
```
Ultimately the solution was to create the directory /usr/share/vgabios and put
the rom file in there.  This directory is already configured to allow reads in
the default libvirt AppArmor profile.

## VGA Rom Nvidia Header
With the AppArmor issue resolved, the VM still won't start if you have a VGA Rom
file specified in the configuration.  Now, instead of AppArmor issues being
logged, the Qemu process is segfaulting on start.

It turns out that sometimes the Nvidia VGA Rom files have a header attached that
is required if you're actually flashing the Rom onto the card but stops the Rom
loading properly from Qemu.  We need to remove this header.

This video gives some details and step by step instructions: https://www.youtube.com/watch?v=1IP-h9IKof0&feature=youtu.be

In short though, open the .rom file in a hex editor (e.g.  Bless), find the
sequence that starts with "55 AA" that is shortly before the text "IBM VGA
Compatible" and delete everything before it.  The "55 AA" sequence should be the
first 2 bytes in the new file.
