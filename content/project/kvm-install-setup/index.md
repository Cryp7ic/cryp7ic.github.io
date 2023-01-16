---
title: KVM Installation and Setup on Arch
summary: KVM Installation, network cofiguration and disk resizing.
tags:
  - linux
date: 2023-01-16
external_link: /project/kvm-install-setup
---

Kernel-based Virtual Machine (KVM) is a visualisation module in the Linux kernel that allows the kernel to function as a hypervisor.

## Table of Contents

- [Installing KVM](#installing-kvm)
- [Enable KVM](#enable-kvm)
- [KVM Configuration](#kvm-configuration)
- [Enable Nested Virtualisation](#enable-nested-virtualisation)
- [Bridged Networking](#bridged-networking)
  - [Method 1: Creating Bridge Network using Virtual Machine Manager (NATed)](#method-2:-Creating-Bridge-Network-using-Virtual-Machine-Manager-(NATed))
  - [Method 2: Create KVM bridge with virsh command.](#Method-2:-Create-KVM-bridge-with-virsh-command.)
- [Extend KVM Virtual Machine disk size](#Extend-KVM-Virtual-Machine-disk-size)
  - [Shut down the Virtual Machine](#Shut-down-the-Virtual-Machine)
  - [Extend your KVM guest OS disk](#Extend-your-KVM-guest-OS-disk)
  - [Extend guest VM disk](#Extend-guest-VM-disk)
  - [Grow VM partition](#Grow-VM-partition)

## Installing KVM

**The first step is to install all packages needed to run KVM.**

```shell
$ sudo pacman -Syy
$ sudo reboot
$ sudo pacman -S qemu virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat
```

**Also install *ebtables* and *iptables* packages.**

```shell
$ sudo pacman -S ebtables iptables
```

**Install libguestfs on Arch**

libguestfs is a set of tools used to access and modify virtual machine disk images.

```shell
$ sudo pacman -S libguestfs
```

## Enable KVM

Start and enable libvirtd service to start at boot.

```shell
$ sudo systemctl enable libvirtd.service
$ sudo systemctl start libvirtd.service
```

## KVM Configuration

**We want to use our standard Linux user account to manage KVM, let's configure KVM to allow this.**

```shell
$ sudo nvim /etc/libvirt/libvirtd.conf
```

**Set the UNIX domain socket group ownership to libvirt, (around line 85)**

```shell
unix_sock_group = "libvirt"
```

**Set the UNIX socket permissions for the R/W socket (around line 102)**

```sh
unix_sock_rw_perms = "0770"
```

**Add your user account to libvirt group.**

```shell
$ sudo usermod -a -G libvirt $(whoami)
$ newgrp libvirt
```

**Restart libvirt daemon.**

```shell
$ sudo systemctl restart libvirtd.service
```

## Enable Nested Virtualisation

Nested virtualisation features enables you to run Virtual Machines inside a VM. Enable Nested virtualisation for `kvm_intel / kvm_amd` by enabling kernel module as shown.

```sh
### Intel Processor ###
$ sudo modprobe -r kvm_intel
$ sudo modprobe kvm_intel nested=1

### AMD Processor ###
$ sudo modprobe -r kvm_amd
$ sudo modprobe kvm_amd nested=1
```

To make this configuration persistent,run:

```shell
$ echo "options kvm-intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
```

Confirm that Nested Virtualisation is set to Yes:

```shell
### Intel Processor ###
$ systool -m kvm_intel -v | grep nested
    nested              = "Y"
    nested_early_check  = "N"
$ cat /sys/module/kvm_intel/parameters/nested 
Y

### AMD Processor ###
$ systool -m kvm_amd -v | grep nested
    nested              = "Y"
    nested_early_check  = "N"
$ cat /sys/module/kvm_amd/parameters/nested 
Y
```

## Bridged Networking

There are various ways of configuring Bridge Networking in Linux for use in KVM. The default network used by a Virtual Machine launched in KVM is **NAT network**. With NAT networking, a virtual network is created for the guest machines which is then mapped to host network to provide internet connectivity.

When you configure and use Bridged networking, guest operating systems access external network connected directly to the host machine. A bridge can be created either using Virtual Machine Manager, using **virsh** command line tool, by directly editing network scripts or using Linux Network management tools.

### Method 1: Creating Bridge Network using Virtual Machine Manager (NATed)

- Open Virtual Machine Manager, and go to **Edit > Connection Details > Virtual Networks**

- Configure a new network interface by clicking the **+** at the bottom of the window. Give the virtual network a name.

- Click the Forward button, on next window, provide virtual network information.

- Click forward and choose if to enable IPv6.

- Select the network type and forwarding policy.

- Finish the setting and save your configurations. The new Virtual network should show on the overview page.

A bridge on the host system is automatically created for the network.

```sh
$ brctl show virbr4      
bridge name	bridge id		STP enabled	interfaces
virbr4		8000.525400c2410a	yes		virbr4-nic
```

### Method 2: Create KVM bridge with virsh command.

Create a new bridge XML file.

```sh
$ nvim br10.xml
```

Add bridge details to the file.

```xml
<network>
  <name>br10</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='br10' stp='on' delay='0'/>
  <ip address='192.168.30.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.30.50' end='192.168.30.200'/>
    </dhcp>
  </ip>
</network>
```

To define a network from an XML file without starting it, use:

```shell
$ sudo virsh net-define  br10.xml
Network br1 defined from br10.xml
```

To start a (previously defined) inactive network, use:

```shell
$ sudo virsh net-start br10
Network br10 started
```

To set network to autostart at service start:

```shell
$ sudo virsh net-autostart br10
Network br10 marked as autostarted
```

Check to Confirm if autostart flag is turned to `yes` – Persistent should read yes as well.

```shell
$ sudo virsh net-list --all
 Name              State    Autostart   Persistent
----------------------------------------------------
 br10              active   yes         yes
 default           active   yes         yes
 docker-machines   active   yes         yes
 fed290            active   no          yes
 vagrant-libvirt   active   no          yes
```

Confirm bridge creation and IP address.

```shell
$ ip addr show dev br10
```

## Extend KVM Virtual Machine disk size

### Shut down the Virtual Machine

Before you can extend your guest machine Virtual disk, you need to shut it down.

```shell
$ sudo virsh list
 Id   Name    State
-----------------------
 4    rhel8   running
```

If your guest machine is in running state, power it off using its ID or Name.

```sh
$ sudo virsh shutdown rhel8
Domain rhel8 is being shutdown
```

Confirm that it is truly down before proceeding to manage its disks.

```shell
$ sudo virsh list          
 Id   Name   State
--------------------
```

### Extend your KVM guest OS disk

Locate your guest OS disk path.

```sh
$ sudo virsh domblklist rhel8
 Target   Source
-----------------------------------------------
 vda      /var/lib/libvirt/images/rhel8.qcow2
 sda      -

OR use:

$ sudo virsh dumpxml rhel8 | egrep 'disk type' -A 5
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/rhel8.qcow2'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
--
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

You can obtain the same information from the *Virtual Machine Manager* GUI. My VM disk is located in ‘/*var/lib/libvirt/images/rhel8.qcow2*‘.

```shell
$ sudo qemu-img info /var/lib/libvirt/images/rhel8.qcow2
image: /var/lib/libvirt/images/rhel8.qcow2
file format: qcow2
virtual size: 30G (42949672960 bytes)
disk size: 2.0G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: true
    refcount bits: 16
    corrupt: false
```

### Extend guest VM disk

Since we know the location of our Virtual Machine disk, let’s extend it to our desired capacity.

```shell
$ sudo qemu-img resize /var/lib/libvirt/images/rhel8.qcow2 +10G
```

Please note that *qemu-img* can’t resize an image which has snapshots. You will need to first remove all VM snapshots. See this example:

```shell
$ sudo virsh snapshot-list rhel8
 Name        Creation Time               State
--------------------------------------------------
 snapshot1   2019-04-16 08:54:24 +0300   shutoff

$ sudo virsh snapshot-delete --domain rhel8 --snapshotname snapshot1
Domain snapshot snapshot1 deleted

$ sudo virsh snapshot-list rhel8                                    
 Name   Creation Time   State
-------------------------------
```

Then extend the disk by using the `**+**‘ before disk capacity.

```shell
$ sudo qemu-img resize /var/lib/libvirt/images/rhel8.qcow2 +10G
Image resized.
```

You can also resize with virsh command. This requires domain to be running.

```shell
$ sudo qemu-img info /var/lib/libvirt/images/rhel8.qcow2
 image: /var/lib/libvirt/images/rhel8.qcow2
 file format: qcow2
 virtual size: 30G (42949672960 bytes)
 disk size: 2.0G
 cluster_size: 65536
 Format specific information:
     compat: 1.1
     lazy refcounts: true
     refcount bits: 16
     corrupt: false

$ sudo virsh start rhel8
$ sudo virsh blockresize rhel8 /var/lib/libvirt/images/rhel8.qcow2 40G
Block device '/var/lib/libvirt/images/rhel8.qcow2' is resized
```

Confirm disk size with fdisk command.

```shell
$ sudo fdisk -l /var/lib/libvirt/images/rhel8.qcow2    
Disk /var/lib/libvirt/images/rhel8.qcow2: 30.2 GiB, 32399818752 bytes, 63280896 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

### Grow VM partition

Now power up the VM

```shell
$ sudo virsh start rhel8
Domain rhel8 started
```

SSH to your VM as root user or using user account that has sudo.

```shell
$ ssh rhel8             
Last login: Fri Apr 19 06:11:19 2019 from 192.168.122.1
[jmutai@rhel8 ~]$ 
```

Check your new disk layout.

```shell
$ lsblk 
 NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
 sr0            11:0    1 1024M  0 rom  
 vda           252:0    0   40G  0 disk 
 ├─vda1        252:1    0    1G  0 part /boot
 └─vda2        252:2    0   29G  0 part 
   ├─rhel-root 253:0    0 26.9G  0 lvm  /
   └─rhel-swap 253:1    0  2.1G  0 lvm  [SWAP]
```
