Scaleway OpenWRT Image
======================

This tutorial is intended to demonstrate how to build a custom image for Scaleway from scratch using the new export/import feature.

The procedure presented is specific to OpenWRT, an open-source project for embedded operating systems based on Linux, primarily used to route network traffic.

It will present the basic needs and actions to create a custom image, but each operating systems may have specifics which will need to be troubleshooted.

Overview
--------

To create a custom image, you will need to build a QCOW2 image and create a snapshot from it.

The steps needed will be:

- Download an OS disk image
- Convert it to QCOW2 (if not provided in this format)
- Edit the image to fit Scaleway ecosystem
- Upload the image to Scaleway Object Storage
- Convert to Snapshot through import
- Test the image and troubleshoot via Console

The following commands are done on an Ubuntu 22.04.

Download the image
------------------

The needed image must be a full disk image, not an ISO image or only a rootfs.

In order to work on Scaleway Instances, the image must be using EFI (not just BIOS) to boot.

OpenWRT images are available [here](https://openwrt.org/downloads), we are going to use a stable release, for x86_64 architecture, with EFI.

To download the image we are going to use in this tutorial:

```
curl -sSLO https://downloads.openwrt.org/releases/22.03.2/targets/x86/64/openwrt-22.03.2-x86-64-generic-ext4-combined-efi.img.gz
gunzip openwrt-22.03.2-x86-64-generic-ext4-combined-efi.img.gz
```

Prepare the QCOW2 image
-----------------------

QCOW2 images are container images used in QEMU as virtual disks.

Install the needed QEMU tools:

```
apt-get install qemu-utils -y
```

Convert the image and resize the target disk size (minimum disk size at Scaleway = 1GB):

```
qemu-img convert -f raw -O qcow2 openwrt-22.03.2-x86-64-generic-ext4-combined-efi.img openwrt-22.03.2-x86-64-generic-ext4-combined-efi.qcow2
qemu-img resize openwrt-22.03.2-x86-64-generic-ext4-combined-efi.qcow2 1G
```

Mount the QCOW2 image as a device

```
modprobe nbd
qemu-nbd -c /dev/nbd0 openwrt-22.03.2-x86-64-generic-ext4-combined-efi.qcow2
```

Conversion and resizing will create errors in the partition table which are fixed when re-writing it using fdisk. Plus, to resize the main partition, the partition number must be found out.

To print and fix partition table, do:

```
echo 'p\nw\n' | fdisk /dev/nbd0
```

To resize the main partition to fit the available space (partition number here is 2):

```
growpart /dev/nbd0 2
resize2fs /dev/nbd0p2
```

Edit image content
------------------

Network configuration and specifics actions may need to be done in order to access the image when run in Scaleway Instances.

For exemple, in Scaleway Instances, the first NIC (eth0 for exemple) will be associated to the public network and need to be configured (Cloud-init? DHCP?)

If the image provide a GUI, it may need to be configured to allow access from public interface (specific port? firewall rules?)

Idem for the SSH access (user password? SSH keys?)

To mount the image:

```
mkdir -p /mount/temp
mount /dev/nbd0p2 /mount/temp
```

OpenWRT need two interfaces to work (wan and lan). But a default Instance only provide one interface. We are going to use the dummy module to add an interface.

Download the needed package

```
curl -sSL -o /mount/temp/kmod-dummy_5.10.146-1_x86_64.ipk https://downloads.openwrt.org/releases/22.03.2/targets/x86/64/packages/kmod-dummy_5.10.146-1_x86_64.ipk
```

Chroot in the image

```
chroot /mount/temp/ /bin/ash
```

Install the package

```
mkdir -p /tmp/lock
opkg install kmod-dummy_5.10.146-1_x86_64.ipk
rm -rf /tmp/* kmod-dummy_5.10.146-1_x86_64.ipk
```

While in the chroot, let's set the root password. In OpenWRT, it is used by the web interface for login.

```
passwd
```

We'll also use the `uci` CLI of OpenWRT to set some configuration

Configure the web server ports and redirect http to https

```
uci del uhttpd.main.listen_http
uci add_list uhttpd.main.listen_http='0.0.0.0:8080'
uci del uhttpd.main.listen_https
uci add_list uhttpd.main.listen_https='0.0.0.0:8443'
uci set uhttpd.main.redirect_https=1
uci commit uhttpd
```

Disable password auth in SSH

```
uci set dropbear.@dropbear[0].PasswordAuth=off
uci set dropbear.@dropbear[0].RootPasswordAuth=off
uci commit dropbear
```

Since we have disabled password auth in SSH, we need a way to load SSH keys when running.

In this tutorial, we will not setup Cloud-init for this but use the same magic ip mechanism to get the keys.

Create the authorized_keys file:

```
touch /etc/dropbear/authorized_keys
chmod 600 /etc/dropbear/authorized_keys
```

Create the fetch script:

```
cat <<EOF>/etc/init.d/fetch_ssh_keys
#!/bin/sh /etc/rc.common

START=97

start() {
     echo -e "\nFetching SSH keys from Scaleway Metadata" > /dev/console
     wget -qO- http://169.254.42.42/conf | egrep 'SSH_PUBLIC_KEYS_._KEY' | cut -d'=' -f2- | sed "s/'//g" > /etc/dropbear/authorized_keys
}

reload() {
     start
}
EOF
chmod +x /etc/init.d/fetch_ssh_keys
```

Add the script to rc.d:

```
ln -s /etc/init.d/fetch_ssh_keys /etc/rc.d/S97fetch_ssh_keys
```

Exit the chroot

```
exit
```

Let's configure the network. eth0 is the first interface associated by Scaleway Instances to the public network (wan) and need to be left to our DHCP server for configuration.

```
cat <<EOF>/mount/temp/etc/config/network
config interface 'loopback'
   option device 'lo'
   option proto 'static'
   option ipaddr '127.0.0.1'
   option netmask '255.0.0.0'

config interface 'wan'
   option device 'eth0'
   option proto 'dhcp'

config interface 'lan'
   option device 'dummy0'
   option proto 'static'
   option ipaddr '192.168.0.1'
   option netmask '255.255.255.0'
EOF
```

Accessing the UI and SSH in OpenWRT is normally only possible through the lan interface.

We will add firewall rules to redirect wan ports to lan ports:

```
cat <<EOF>>/mount/temp/etc/config/firewall
# port redirect UI ACCESS and SSH
config redirect
   option name             HTTP-UI
   option src              wan
   option src_dport        8080
   option desti            lan
   option dest_ip          192.168.0.1
   option dest_port        8080
   option proto            tcp

config redirect
   option name             HTTPS-UI
   option src              wan
   option src_dport        8443
   option dest             lan
   option dest_ip          192.168.0.1
   option dest_port        8443
   option proto            tcp

config redirect
   option name             SSH
   option src              wan
   option src_dport        22
   option dest             lan
   option dest_ip          192.168.0.1
   option dest_port        22
   option proto            tcp
EOF
```

Once all actions are done, let's "close" the image container:

```
umount /mount/temp
qemu-nbd -d /dev/nbd0
```

Importing the image
-------------------

The first step to import the image is to upload it to Scaleway Object Storage.

You can use the web console or your favorite S3 CLI to upload in a bucket.

In this example, the [AWS CLI](https://www.scaleway.com/en/docs/storage/object/api-cli/object-storage-aws-cli/) is used:

```
aws s3 cp openwrt-22.03.2-x86-64-generic-ext4-combined-efi.qcow2 s3://demo-arno/openwrt.qcow2
```

Then, you can trigger the import of the image as a snapshot in one of the region where the bucket is, using [SCW CLI](https://github.com/scaleway/scaleway-cli):

```
scw instance snapshot create zone=fr-par-1 name=openwrt volume-type=unified bucket=demo-arno key=openwrt.qcow2
```

Create the corresponding Instance image:

```
scw instance image create zone=fr-par-1 name=openwrt arch=x86_64 snapshot-id=$(scw instance snapshot list | grep -m1 openwrt | awk '{print $1}')
```


Testing and troubleshooting the image
-------------------------------------

Let's create an Instance using the snapshot image:

```
scw instance server create type=DEV1-S zone=fr-par-1 name=scw-openwrt ip=new image=$(scw instance image list | grep -m1 openwrt | awk '{print $1}')
```

You can then troubleshoot the boot and configuration using the console:

```
scw instance server console zone=fr-par-1 $(scw instance server list | grep -m1 scw-openwrt | awk '{print $1}')
```

You can exit the console using `Ctrl+q`

You can access the web console of OpenWRT using `https://<public ip of instance>:8443`


Note on support
---------------

Scaleway can't provide support for these kind of images and building/running those is the responsability of the user.
