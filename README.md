## Overview
This documents the rather lengthy process I used to install KDE Neon on my PC
with the OS root on ZFS.  To complicate matters, I insisted on using the latest
available of ZFS-on-Linux so that I could make use of native encryption.

The usual approach for encryption with ZFS-on-Linux has been to use LUKS as an
encryption layer between ZFS and the raw partition.  This has served me very
well for several years.  The awesome team over at ZFS-on-Linux and OpenZFS
implemented native ZFS encryption though and I wanted to give it a try.

Because the process requires building the ZFS tools and kernel modules from
source, doing this repeatedly in the Live CD environment became a real pain.  To
work around needing a live environment to fall back to if something went wrong,
I have a full KDE Neon install in an ext4 partition that I can boot to any time
something goes wrong.  As I hone the process, I can probably do away with this
extra partition in future builds.

What is working:
* OS boots and from that point everything operates as normal

What is not working:
* Grub integration: The normal update-grub script doesn't find the Linux
installations on ZFS.  At the time of writing, I have been manually modifying my
grub.cfg
* Plymouth: The nice loading splash screen while waiting for the OS to boot
doesn't work properly with encryption enabled.  Instead of something asking for
the ZFS volume's key, you are dropped to a busybox prompt and asked to mount
your OS volume yourself.

### A future feature: individual encrypted user home datasets
As well as having my OS root on ZFS, I also want to have each (non-system)
user's home directory it's own ZFS dataset with the user's password used as the
encryption key.  Logging in would use the login password to unlock and mount the
dataset.  This is nowhere near operational yet.  Watch this space.

## Partitioning
My target machine has a 120GiB SSD as it's primary disk  This is how I
partitioned it:
```
Number  Start   End     Size    File system  Name       Flags
 1      1049kB  48.2MB  47.2MB  fat32        esp        boot, esp
 2      48.2MB  96.5MB  48.2MB               bios_grub  bios_grub
 3      96.5MB  1096MB  999MB   ext4         boot
 4      1096MB  7000MB  5905MB  ext4         rescue
 5      7000MB  120GB   113GB   zfs          SSD1
```

## Install Rescue OS
I now just install KDE Neon into partition 4 (labelled "rescue") and restart
when it's done.

## Post Rescue OS Install
### Building ZFS tools and kernel module from source
```
sudo apt install zfsutils-linux dkms zfs-initramfs dh-make
sudo apt install python3-distutils dh-autoreconf tzdata uuid-dev libblkid-dev libssl-dev zlib1g-dev
cd /usr/src
ZFSVERSION="0.8.0"
KERNELVERSION=$(uname -r)
sudo wget https://github.com/zfsonlinux/zfs/releases/download/zfs-0.8.0-rc3/zfs-0.8.0-rc3.tar.gz
sudo tar -xzf zfs-0.8.0-rc3.tar.gz
printf '#!/bin/sh\ncp \"$@\"' | sudo tee zfs-${ZFSVERSION}/cp
sudo chmod +x zfs-${ZFSVERSION}/cp
cd zfs-${ZFSVERSION}/scripts/
sudo ./dkms.mkconf -n zfs -v ${ZFSVERSION} -c kernel -f ../dkms.conf

sudo dkms build -m zfs -v ${ZFSVERSION} -k ${KERNELVERSION}
sudo dkms mkbmdeb -m zfs -v ${ZFSVERSION} -k ${KERNELVERSION}
sudo reboot
```
The zfs contributed initramfs additions want to go into /usr/local/share instead of /usr/share
```
cd /usr/local/share
ln -s /usr/share/initramfs-tools
```

```
cd /usr/src/zfs-0.8.0
sudo ./configure --bindir=/usr/bin --sbindir=/sbin --libdir=/lib --with-udevdir=/lib/udev --with-config=user --with-systemdunitdir=/lib/systemd/system --with-systemdpresetdir=/lib/systemd/system-preset
cd lib
sudo make
sudo make install
cd ../cmd
sudo make; sudo make install
cd ../contrib/initramfs
sudo make
sudo make install
```

TODO: Grub settings

```
sudo update-initramfs -c -k all
```

### Create the zpool that the normal OS will be installed to
```
sudo zpool create -o ashift=12 -d \
      -o feature@async_destroy=enabled \
      -o feature@encryption=enabled \
      -o feature@bookmarks=enabled \
      -o feature@embedded_data=enabled \
      -o feature@empty_bpobj=enabled \
      -o feature@enabled_txg=enabled \
      -o feature@extensible_dataset=enabled \
      -o feature@filesystem_limits=enabled \
      -o feature@hole_birth=enabled \
      -o feature@large_blocks=enabled \
      -o feature@lz4_compress=enabled \
      -o feature@spacemap_histogram=enabled \
      -o feature@userobj_accounting=enabled \
      -O acltype=posixacl -O canmount=off -O compression=lz4 -O devices=off \
      -O normalization=formD -O relatime=on -O xattr=sa \
      -O mountpoint=/ -R /mnt/zfs/SSD1 \
      SSD1 /dev/sda5
sudo zfs create SSD1/OS
sudo zfs create -o canmount=off -o mountpoint=none -o encryption=on -o keyformat=passphrase SSD1/kdeneon
sudo zfs create -o canmount=noauto -o mountpoint=/ SSD1/kdeneon/ROOT
```

## Install the normal (non-rescue) OS using Debootstrap
```
mkdir /mnt/target
sudo apt install debootstrap
sudo zfs set mountpoint=legacy SSD1/OS/kdeneon/ROOT
sudo mount -t zfs SSD1/OS/kdeneon/ROOT /mnt/target
sudo debootstrap --variant=buildd --arch amd64 bionic /mnt/target http://archive.ubuntu.com/ubuntu/
sudo mount --rbind /dev /mnt/target/dev
sudo mount --rbind /proc /mnt/target/proc
sudo mount --rbind /sys /mnt/target/sys
sudo mount --rbind /dev/pts /mnt/target/dev/pts
sudo cp -rav /etc/apt/* /mnt/target/etc/apt/
sudo rm /mnt/target/etc/apt/apt.conf.d/20snapd.conf
```

```
dpkg-query -l |awk '{print $2}' >/mnt/target/tmp/package-list
vim /mnt/target/tmp/package-list
```
Remove the first few lines which aren't package names

```
sudo chroot /mnt/target
```
Crap! I didn't document the step that goes here.  It was something like:
```
sudo apt install $(cat /tmp/package-list)
```

Create a user in the new OS (use your own preferred username obviously)
```
sudo adduser shaun
sudo usermod -a -G sudo shaun
```

Configure NetworkManager operate as expected on first boot.
```
sudo apt purge netplan.io
sudo rm /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf
```

## First Boot
It's time to reboot into the OS that's on ZFS.  When you restart you should end
up at a Grub menu with Neon, Ubuntu or whichever OS you installed in your rescue
partition.  Press the "e" key toy edit this entry.  Use the arrow keys to move
the cursor to the line that starts with "linux".  There will be a section that
looks like this:
```
root=UUID=15623082-53a1-11e9-9c87-5f721789ea04
```
Change it to point to the ZFS root volume:
```
root=ZFS=SSD1/OS/kdeneon/ROOT
```
Press F10 to boot.

With luck, you should soon be prompted for the password to unlock the ZFS
dataset and, after you have provided the passphrase, the OS should boot to the
graphical login screen.

Once you're logged in, you need to repeat the ***"Building ZFS tools and kernel
module from source"*** section from above on this OS instance.

## Permanently add an entry to the Grub menu
TODO

## Troubleshooting
TODO

## Personalisation
Everything below this point is just what I do to setup my system the way I like
it.  Use anything you like but everything is optional.

### ZSH as default shell with oh-my-zsh
```
sudo apt install zsh git
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
vim ~/.zshrc
ZSH_THEME="agnoster"
```

### Libvirt for running local VMs
```
sudo apt install libvirt-bin libvirt-daemon-driver-storage-zfs sgabios ovmf
```

```
sudo vim /etc/ssh/sshd_config
```
Set "MaxSessions" to 100

For local VM management
```
sudo apt install virt-manager
```

Create a ZFS dataset for VMs (even though the VMs themselves will use ZVOLs)
```
sudo zfs create SSD1/VMs/libvirt
```

### Add a zpool as a location that libvirt can store disks as zvols
```
virsh -c qemu:///system pool-define-as --name SSD1 --type zfs --source-dev SSD1
virsh -c qemu:///system pool-autostart SSD1
virsh -c qemu:///system pool-start SSD1
```

### Add a shared location (dataset) for .isos, etc.
```
sudo zfs create SSD1/VMs/Shared
virsh -c qemu:///system pool-define-as --name Shared --type dir --target /mnt/zfs/SSD1/VMs/Shared
```

### Setup zfs for a new VM
```
VMNAME="neon1.ghanima.net"
sudo zfs create SSD1/VMs/libvirt/${VMNAME}
sudo zfs create -V 10G SSD1/VMs/libvirt/${VMNAME}/ROOT
sudo zfs create -V 5G -o checksum=off -o compression=off -o dedup=off -o sync=disabled -o primarycache=none SSD1/VMs/libvirt/${VMNAME}/SWAP
sudo zfs create -V 2G -o checksum=off -o compression=off -o dedup=off -o sync=disabled -o primarycache=none SSD1/VMs/libvirt/${VMNAME}/TEMP
```

### Setup a bridge with NetworkManager
```
nmcli con down 'Wired connection 1'
nmcli con delete 'Wired connection 1'
nmcli con add type bridge ifname brLAN con-name brLAN
nmcli con add type bridge-slave ifname eno1 con-name eno1 master brLAN
nmcli con up brLAN
```

### Setup the bridge to be available in libvirt
```
vim /tmp/brLAN.xml
```

```
<network>
  <name>brLAN</name>
  <forward mode="bridge"/>
  <bridge name="brLAN" />
</network>
```

```
virsh -c qemu:///system net-define /tmp/brLAN.xml
virsh -c qemu:///system net-start brLAN
virsh -c qemu:///system net-autostart brLAN
```
