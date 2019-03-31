# Partitioning
Number  Start   End     Size    File system  Name       Flags
 1      1049kB  48.2MB  47.2MB  fat32        esp        boot, esp
 2      48.2MB  96.5MB  48.2MB               bios_grub  bios_grub
 3      96.5MB  1096MB  999MB   ext2         boot
 4      1096MB  7000MB  5905MB  ext2         rescue

# Post Install
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

 the zfs contributed initramfs additions want to go into /usr/local/share instead of /usr/share
cd /usr/local/share
ln -s /usr/share/initramfs-tools

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

sudo update-initramfs -c -k all

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

# Using Debootstrap
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

dpkg-query -l |awk '{print $2}' >/mnt/target/tmp/package-list
vim /mnt/target/tmp/package-list
  Remove the first few lines which aren't package names

sudo chroot /mnt/target
  Crap! I didn't document the step that goes here

sudo adduser shaun
sudo usermod -a -G sudo shaun
sudo apt purge netplan.io
sudo rm /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf

# Using the rescue installation
sudo zfs mount -O SSD1/OS/kdeneon/ROOT


cd /mnt/zfs/SSD1
sudo tar -v --one-file-system --acls --xattrs -czf ./rescue-root.tgz /

# Personalisation
sudo apt install zsh git
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
vim ~/.zshrc
ZSH_THEME="agnoster"

# Libvirt
sudo apt install libvirt-bin libvirt-daemon-driver-storage-zfs sgabios ovmf

 For local VM management
sudo apt install virt-manager

sudo vim /etc/ssh/sshd_config
 Set "MaxSessions" to 100
sudo zfs create SSD1/VMs/libvirt

## Add a zpool as a location that libvirt can store disks as zvols
virsh -c qemu:///system pool-define-as --name SSD1 --type zfs --source-dev SSD1
virsh -c qemu:///system pool-autostart SSD1
virsh -c qemu:///system pool-start SSD1

## Add a shared location for .isos, etc.
sudo zfs create SSD1/VMs/Shared
virsh -c qemu:///system pool-define-as --name Shared --type dir --target /mnt/zfs/SSD1/VMs/Shared

## Setup zfs for a new VM
VMNAME="neon1.ghanima.net"
sudo zfs create SSD1/VMs/libvirt/${VMNAME}
sudo zfs create -V 10G SSD1/VMs/libvirt/${VMNAME}/ROOT
sudo zfs create -V 5G -o checksum=off -o compression=off -o dedup=off -o sync=disabled -o primarycache=none SSD1/VMs/libvirt/${VMNAME}/SWAP
sudo zfs create -V 2G -o checksum=off -o compression=off -o dedup=off -o sync=disabled -o primarycache=none SSD1/VMs/libvirt/${VMNAME}/TEMP

## Setup a bridge with NetworkManager
nmcli con down 'Wired connection 1'
nmcli con delete 'Wired connection 1'
nmcli con add type bridge ifname brLAN con-name brLAN
nmcli con add type bridge-slave ifname eno1 con-name eno1 master brLAN
nmcli con up brLAN

## Setup the bridge to be available in libvirt
vim /tmp/brLAN.xml
```
<network>
  <name>brLAN</name>
  <forward mode="bridge"/>
  <bridge name="brLAN" />
</network>
```
virsh -c qemu:///system net-define /tmp/brLAN.xml
virsh -c qemu:///system net-start brLAN
virsh -c qemu:///system net-autostart brLAN
