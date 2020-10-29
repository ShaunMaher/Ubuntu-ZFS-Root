# Ubuntu Server
zfs set mountpoint=/boot bpool
## Installation
```
rsync -avPX /target/. /alexandria/ROOT/ubuntu-1/.
```

### Partitioning
```
sudo parted /dev/sda
```
```
mklabel gpt
mkpart bios_grub ext2 2048s 8M
set 1 bios_grub on
mkpart ESP fat32 8M 1G
set 2 ESP on
mkpart BOOT ext2 1G 3G
mkpart SSD1 ext2 3G 100G
mkpart TMP ext2 100G 120G
```
```
mkfs.fat -F 32 /dev/sda2
```

### Software RAID (optional)
```
mdadm --create md0 --level=1 --raid-devices=2 --metadata=0.90 --name=BOOT /dev/sda3 /dev/sab3
mdadm --create md1 --level=1 --raid-devices=2 --name=ROOT /dev/sda4 /dev/sdb4
```

### Install Ubuntu Server
At this point, install Ubuntu Server, using the normal installation media, onto
/dev/sda5.  When prompted, reboot but boot off the installation media again.

### boot zpool
```
sudo apt install zfsutils-linux
sudo zpool create -d \
                -o feature@async_destroy=enabled \
                -o feature@empty_bpobj=enabled \
                -o feature@spacemap_histogram=enabled \
                -o feature@enabled_txg=enabled \
                -o feature@hole_birth=enabled \
                -o feature@bookmarks=enabled \
                -o feature@embedded_data=enabled \
                -o feature@large_blocks=enabled \
                -O mountpoint=/mnt/zfs/bpool \
                bpool /dev/disk/by-partlabel/BOOT
sudo zfs create bpool/BOOT
sudo zfs create bpool/BOOT/ROOT    <-- I think this should match the name of the OS ROOT zfs dataset
sudo zfs set mountpoint=/boot bpool/BOOT/ROOT
```

### Root zpool
```
sudo zpool create -o ashift=12  -o feature@async_destroy=enabled \
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
                                -O acltype=posixacl \
                                -O compression=lz4 \
                                -O devices=off \
                                -O normalization=formD \
                                -O relatime=on \
                                -O xattr=sa \
                                -O mountpoint=/mnt/zfs/SSD1 SSD1 /dev/disk/by-partlabel/SSD1
sudo zfs create SSD1/OS
sudo zfs create -o encryption=on -o keyformat=passphrase -o compression=on SSD1/OS/UbuntuServerFocal
sudo zfs load-key SSD1/OS/UbuntuServerFocal
sudo zfs create SSD1/OS/UbuntuServerFocal/ROOT

```

### Copy the installed OS
```
zpool export SSD1
zpool export bpool
mkdir /target
zpool import -R /target SSD1
zpool import -R /target bpool
zfs set overlay=on SSD1/OS/UbuntuServerFocal/ROOT
zfs set mountpoint=/ SSD1/OS/UbuntuServerFocal/ROOT
zpool export bpool
zpool export SSD1
zpool import -R /target SSD1
zfs load-key SSD1/OS/UbuntuServerFocal
zfs mount SSD1/OS/UbuntuServerFocal/ROOT
zpool import -R /target bpool

mkdir /source
mount /dev/sda5 /source
rsync -avPX /source/. /target/.
```

### Make the new OS bootable
```
sudo mount /dev/sda2 /target/boot/efi
sudo mount --rbind /dev /target/dev
sudo mount --rbind /proc /target/proc
sudo mount --rbind /sys /target/sys
sudo mount --rbind /dev/pts /target/dev/pts
sudo chroot /target
```
```
echo "nameserver 8.8.8.8" >/etc/resolv.conf
sudo apt install zfsutils-linux zfs-initramfs
sudo grub-install /dev/sda
sudo update-grub
exit
```
```
sudo umount /target/boot/efi
zfs snapshot SSD1/OS/UbuntuServerFocal/ROOT@fresh-install
zfs snapshot bpool@fresh-install
zpool export bpool
zpool export SSD1
sudo reboot
```

## ZFS Encrypted Root, remote unlock on start up
```
sudo apt install dropbear-initramfs
```

Generate a public/private key pair
```
ssh-keygen -b 2048 -t rsa -C "initramfs" -N "" -f ~/.ssh/ph_initramfs
```

Edit `/usr/share/initramfs-tools/scripts/unlock-zfs-root`, adding the following
content:
```
# Source the ZFS functions
. /scripts/zfs

root=$(ps | grep -v grep | grep 'zfs load-key' | awk '{print $NF}')
load_key_pid=$(ps | grep -v grep | grep 'zfs load-key' | awk '{print $1}')

if [ ! "x${root}" = "x" ] && [ ! "x${load_key_pid}" = "x" ]; then
  decrypt_fs $root
  kill $load_key_pid
  exit 1
else
  echo "Unable to find the \"zfs load-key\" process."
  exit 1
fi
```

Edit `/etc/dropbear-initramfs/authorized_keys`
```
no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/scripts/unlock-zfs-root" ssh-rsa AAAAB3Nz...cedT initramfs
```

Edit `/etc/dropbear-initramfs/config` and set `DROPBEAR_OPTIONS` to:
```
DROPBEAR_OPTIONS="-p 4748 -s -j -k -I 60"
```

If you need to assign a static IP, In `/etc/default/grub`, set
`GRUB_CMDLINE_LINUX_DEFAULT` to:
```
GRUB_CMDLINE_LINUX_DEFAULT="ip=172.30.0.205::172.30.0.1:255.255.0.0:ph2:enp34s0"
```

## ZFS Mount Encrypted Volumes
Out of the box, ZFS encrupted volumes other than the root volume are not mounted
on boot, even if they have a file path as the source of the encryption key.

We can modify the systemd zfs-mount.service unit file to load the keys before
issuing the mount command.

```
cp /lib/systemd/system/zfs-mount.service /etc/systemd/system/
```
Now edit `/etc/systemd/system/zfs-mount.service`.  For each dataset that needs
to be unlocked before being mounted and for which the unlock key is stored as a
file, add a line like the following in the `[Service]` stanza:
```
ExecStartPre=-/sbin/zfs load-key SSD1/VMs
ExecStartPost=-/usr/bin/mount /boot/efi
```

## Network config
Remove NetPlan:
```
sudo apt install network-manager
printf "[keyfile]\nunmanaged-devices=none\n" | sudo tee /etc/NetworkManager/conf.d/10-globally-managed-devices.conf
sudo apt remove netplan.io
```

To remove the virbr0 interface created by libvirt:
```
sudo rm /etc/libvirt/qemu/networks/autostart/default.xml
```

TODO: things done in /etc/NetworkManager/NetworkManager.conf

```
sudo nmcli con down 'Wired connection 1'
sudo nmcli con delete 'Wired connection 1'
sudo nmcli con add type bridge ifname brLAN con-name brLAN
sudo nmcli con add type bridge-slave ifname enp34s0 con-name enp34s0 master brLAN
sudo nmcli con modify brLAN ipv4.addresses 172.30.0.205/16 ipv4.gateway 172.30.0.1 ipv4.method manual
sudo nmcli con modify brLAN ipv4.dns 172.30.0.1,172.30.0.2
sudo nmcli connection modify brLAN connection.autoconnect-slaves 1
sudo nmcli connection modify brLAN connection.autoconnect-retries 0
sudo nmcli connection modify brLAN bridge.stp no

sudo nmcli con add type bridge ifname brWAN con-name brWAN
sudo nmcli con add type vlan ifname enp0s25.vlWAN con-name enp0s25.vlWAN dev enp0s25 id 2 master brWAN slave-type bridge
sudo nmcli connection modify brWAN ipv4.method disabled
sudo nmcli connection modify brWAN ipv6.method ignore
sudo nmcli connection modify brWAN connection.autoconnect-slaves 1
sudo nmcli connection modify brWAN connection.autoconnect-retries 0
sudo nmcli connection modify brWAN bridge.stp no

sudo nmcli con add type bridge ifname brWWAN con-name brWWAN
sudo nmcli con add type vlan ifname enp0s25.vlWWAN con-name enp0s25.vlWWAN dev enp0s25 id 3 master brWWAN slave-type bridge
sudo nmcli connection modify brWWAN ipv4.method disabled
sudo nmcli connection modify brWWAN ipv6.method ignore
sudo nmcli connection modify brWWAN connection.autoconnect-slaves 1
sudo nmcli connection modify brWWAN connection.autoconnect-retries 0
sudo nmcli connection modify brWWAN bridge.stp no

sudo nmcli con add type bridge ifname brIOC con-name brIOC
sudo nmcli con add type vlan ifname enp0s25.vlIOC con-name enp0s25.vlIOC dev enp0s25 id 4 master brIOC slave-type bridge
sudo nmcli connection modify brIOC ipv4.method disabled
sudo nmcli connection modify brIOC ipv6.method ignore
sudo nmcli connection modify brIOC connection.autoconnect-slaves 1
sudo nmcli connection modify brIOC connection.autoconnect-retries 0
sudo nmcli connection modify brIOC bridge.stp no

sudo nmcli con add type bridge ifname brGUEST con-name brGUEST
sudo nmcli con add type vlan ifname enp0s25.vlGUEST con-name enp0s25.vlGUEST dev enp0s25 id 5 master brGUEST slave-type bridge
sudo nmcli connection modify brGUEST ipv4.method disabled
sudo nmcli connection modify brGUEST ipv6.method ignore
sudo nmcli connection modify brGUEST bridge.stp no
sudo nmcli connection modify brGUEST connection.autoconnect-slaves 1
sudo nmcli connection modify brGUEST connection.autoconnect-retries 0
```

## Additional packages
```
sudo apt install libvirt-daemon sgabios ovmf nfs-kernel-server qemu-kvm
sudo snap install lxd
```

## Dataset for VMs
```
sudo mv /etc/libvirt /etc/libvirt.orig
head -c 32 /dev/urandom | sudo tee /etc/zfs/keys/SSD1_VMs
sudo zfs create -o sync=disabled -o encryption=on -o keylocation=file:///etc/zfs/keys/SSD1_VMs -o keyformat=raw SSD1/VMs
sudo zfs create -o mountpoint=/etc/libvirt SSD1/VMs/config
sudo zfs set overlay=on SSD1/VMs/config
sudo zfs create -o mountpoint=/var/lib/libvirt/images -o overlay=on SSD1/VMs/shared
sudo setfacl -m g:libvirt:rwX /var/lib/libvirt/images
sudo zfs create -o mountpoint=/var/lib/libvirt/machines -o overlay=on SSD1/VMs/machines
cd /etc/libvirt.orig
sudo tar -czf /tmp/libvirt.tgz .
cd /etc/libvirt
sudo tar -xzf /tmp/libvirt.tgz
cd ..
rm -fr /etc/libvirt.orig
```

## Provision a new KVM VM
### ZFS dataset passthrough with 9p on root (experimental)
```
VMNAME="neon2.internal.ghanima.net"
sudo zfs create SSD1/VMs/machines/${VMNAME}/boot
sudo zfs create SSD1/VMs/machines/${VMNAME}/rootfs
sudo zfs set sync=disabled SSD1/VMs/machines/${VMNAME}/rootfs
sudo zfs create SSD1/VMs/machines/${VMNAME}/temp
sudo zfs create SSD1/VMs/machines/${VMNAME}/swap
virsh -c qemu:///system pool-define-as ${VMNAME}-boot dir - - - - "/tank/kvm/images/${VMNAME}/boot"
virsh -c qemu:///system pool-define-as ${VMNAME}-temp dir - - - - "/tank/kvm/images/${VMNAME}/temp"
virsh -c qemu:///system pool-define-as ${VMNAME}-swap dir - - - - "/tank/kvm/images/${VMNAME}/swap"
virsh -c qemu:///system pool-build ${VMNAME}-boot; virsh -c qemu:///system pool-start ${VMNAME}-boot; virsh -c qemu:///system pool-autostart ${VMNAME}-boot
virsh -c qemu:///system pool-build ${VMNAME}-swap; virsh -c qemu:///system pool-start ${VMNAME}-swap; virsh -c qemu:///system pool-autostart ${VMNAME}-swap
virsh -c qemu:///system pool-build ${VMNAME}-temp; virsh -c qemu:///system pool-start ${VMNAME}-temp; virsh -c qemu:///system pool-autostart ${VMNAME}-temp
```

Inside the VM:
```
echo -e "9p\n9pnet_virtio\n9pnet" | sudo tee -a /etc/initramfs-tools/modules
sudo update-initramfs -u
sudo mkdir /target
rsync TODO
```
Remove swap from /etc/fstab until later.
Make /tmp tmpfs inside the VM?

```
root=root rw rootfstype=9p rootflags=trans=virtio console=ttyS0 ro quiet splash vt.handoff=7
root=root rw rootfstype=9p rootflags=trans=virtio console=ttyS0 verbose=5 init=/bin/sh
```

## IKFS
Create a suitable ZFS dataset with the desired encryption, etc. then
```
sudo apt install pv qemu-user-static
head -c 32 /dev/urandom | sudo tee /etc/zfs/keys/SSD1_ikfs
sudo zfs create -o mountpoint=/var/lib/ikfs -o sync=disabled -o encryption=on -o keylocation=file:///etc/zfs/keys/SSD1_ikfs -o keyformat=raw SSD1/ikfs
sudo zfs create -o mountpoint=/tmp/build-temp SSD1/ikfs/build-temp
sudo zfs create SSD1/ikfs/chroot-templates
sudo useradd -m -d /var/lib/ikfs -r -s /bin/bash ikfs
sudo usermod -a -G sudo ikfs
sudo chown -R ikfs:ikfs /var/lib/ikfs
sudo chmod -R g+w /var/lib/ikfs
sudo usermod -a -G ikfs ubuntu
sudo visudo /etc/sudoers.d/ikfs
```
```
ikfs    ALL=(ALL) NOPASSWD: /var/lib/ikfs/bin/ikfs-cron
```
```
cd /var/lib/ikfs
```

## LXD
```
sudo zfs create -o sync=disabled -o encryption=on -o keylocation=file:///etc/zfs/keys/SSD1_ikfs -o keyformat=raw SSD1/containers
lxc profile device add default eth0 nic nictype=bridged parent=brLAN
```

```
config: {}
networks: []
storage_pools:
- config:
    source: SSD1/Containers
  description: ""
  name: SSD1
  driver: zfs
profiles:
- config: {}
  description: ""
  devices:
    eth0:
      name: eth0
      nictype: bridged
      parent: brLAN
      type: nic
    root:
      path: /
      pool: SSD1
      type: disk
  name: default
cluster: null
```

### LXC in a partitioned off network on your PC
172.31.3.192/26 = LXC Container subnet
172.31.3.193 = LXC Host IP in LXC container subnet
172.30.0.0/16 = The LAN that the container should also have no access to
```
sudo ufw insert 1 deny from 172.31.3.192/26 to 172.31.3.193/32
sudo ufw insert 1 allow from 172.31.3.192/26 to 172.31.3.193/32 port 53 proto udp
sudo ufw insert 1 allow from 172.31.3.192/26 to 172.31.3.193/32 port 53 proto tcp
sudo ufw insert 1 allow from 0.0.0.0/0 to 0.0.0.0/0 port 67 proto udp
```
To also deny access to your LAN so the container basically has internet access
and nothing else, add the following to /etc/ufw/before.rules
```
-I ufw-before-forward -d 172.31.3.193/32 -j DROP
-A ufw-before-forward -j ACCEPT
```

If, ont he other hand, you don't mind if the containers can access your LAN, you
can simply:
```
sudo ufw default allow FORWARD
```

### LXC Container hack to NOT have a link-local IPv6 address (just a global scope IPv6 Address)
```
sudo apt install ifupdown
sudo apt purge netplan.io
sudo vim /etc/network/interfaces
```
```
auto eth0
iface eth0 inet6 auto
        pre-up  /bin/bash -c "IP=\$(ip address show eth0 | grep 'scope link' | awk '{print \$2}'); if [ ! \"\$IP\" == \"\" ]; then ip address del \$IP dev eth0; fi"
        post-up /bin/bash -c "IP=\$(ip address show eth0 | grep 'scope link' | awk '{print \$2}'); if [ ! \"\$IP\" == \"\" ]; then ip address del \$IP dev eth0; fi"
iface eth0 inet dhcp
        post-up /bin/bash -c "IP=\$(ip address show eth0 | grep 'scope link' | awk '{print \$2}'); if [ ! \"\$IP\" == \"\" ]; then ip address del \$IP dev eth0; fi"
```
