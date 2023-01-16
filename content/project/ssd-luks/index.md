---
title: Disk Encryption in Linux
summary: Disk encryption with XFS file system
tags:
  - linux
  - sec
date: 2023-01-13
external_link: /project/ssd-luks
---

## Linux encryption methods

There are two methods to encrypt your data:

### Filesystem stacked level encryption

1. [eCryptfs](https://launchpad.net/ecryptfs) – It is a cryptographic stacked Linux filesystem. eCryptfs stores  cryptographic metadata in the header of each file written, so that  encrypted files can be copied between hosts; the file will be decrypted  with the proper key in the Linux kernel keyring. This solution is widely used, as the basis for Ubuntu’s Encrypted Home Directory, natively  within Google’s ChromeOS, and transparently embedded in several network  attached storage (NAS) devices.
2. [EncFS](http://www.arg0.net/encfs) -It provides an encrypted filesystem in user-space. It runs without any special permissions and uses the FUSE library and Linux kernel module  to provide the filesystem interface. You can find links to source and  binary releases below. EncFS is open source software, licensed under the GPL. 

### Block device level encryption

1. [Loop-AES](https://sourceforge.net/projects/loop-aes/) – Fast and transparent file system and swap encryption package for  linux. No source code changes to linux kernel. Works with 3.x, 2.6, 2.4, 2.2 and 2.0 kernels. 
2. [VeraCrypt](https://www.veracrypt.fr/) – It is free open-source disk encryption software for Windows 7/Vista/XP, Mac OS X and Linux based on TrueCrypt codebase.
3. [dm-crypt+LUKS](https://gitlab.com/cryptsetup/cryptsetup) – dm-crypt is a transparent disk encryption subsystem in Linux kernel  v2.6+ and later and DragonFly BSD. It can encrypt whole disks, removable media, partitions, software RAID volumes, logical volumes, and files. 

## Encrypting external drives

It's not common to separate an internal hard drive from its computer, but external drives are designed to travel. As technology gets smaller  and smaller, it's easier to put a portable drive on your keychain and  carry it around with you every day. The obvious danger, however, is that these are also pretty easy to misplace. I've found abandoned drives in  the USB ports of hotel lobby computers, business center printers,  classrooms, and even a laundromat. Most of these didn't include personal information, but it's an easy mistake to make.

You can mitigate against misplacing important data by encrypting your external drives.

LUKS and its frontend `cryptsetup` provide a way to do  this on Linux. As Linux does during installation, you can encrypt the  entire drive so that it requires a passphrase to mount it.

## How to encrypt an external drive with LUKS

First, you need an empty external drive (or a drive with contents  you're willing to erase). This process overwrites all the data on a drive, so if you have data that you want to keep on the drive, *back it up first*.

`sudo pacman -S cryptsetup`

### 1. Find your drive

I used a small USB thumb drive. To protect you from accidentally  erasing data, the drive referenced in this article is located at the  imaginary location `/dev/sdX`. Attach your drive and find its location:

```
$ lsblk
sda    8:0    0 111.8G  0 disk 
sda1   8:1    0 111.8G  0 part /
sdb    8:112  1  57.6G  0 disk 
sdb1   8:113  1  57.6G  0 part /mydrive
sdX    8:128  1   1.8G  0 disk 
sdX1   8:129  1   1.8G  0 part
```

I know that my demo drive is located at `/dev/sdX` because I recognize its size (1.8GB), and it's also the last drive I attached (with `sda` being the first, `sdb` the second, `sdc` the third, and so on). The `/dev/sdX1` designator means the drive has 1 partition.

If you're unsure, remove your drive, look at the output of `lsblk`, and then attach your drive and look at `lsblk` again.

Make sure you identify the correct drive because encrypting it overwrites *everything on it*. My drive is not empty, but it contains copies of documents I have  copies of elsewhere, so losing this data isn't significant to me.

### 2. Clear the drive

We need to change the permissions of the newly-minted filesystem that is in the encrypted harddisk/file container.

Using the example in the example where `/dev/mapper/$name` has already been mounted to  `/media/mount_point` with a command like: 

```
sudo mount /dev/mapper/$name /media/mount_point
```

Open a terminal in your account and type **enter this command**: 

> **`sudo chown -R yourUserName /media/mount_point`**

For example if joe has a mount point named myEncryptedHD he should do:

```
sudo chown -R joe /media/myEncryptedHD
```

What this does is Joe changes the owner of all files in `myEncrypted` to `joe`, and he now has read and write access. Life is good.

If you decide you don't want R/W permissions anymore, just `sudo chown -R root /media/mount_point` and you'll revoke your rights.To proceed, destroy the drive's partition table by overwriting the drive's head with zeros:

```
$ sudo dd if=/dev/zero of=/dev/sdX count=4096
```

This step isn't strictly necessary, but I like to start with a clean slate.

### 3. Format your drive for LUKS

The `cryptsetup` command is a frontend for managing LUKS volumes. The `luksFormat` subcommand creates a sort of LUKS vault that's password-protected and can house a secured filesystem.

When you create a LUKS partition, you're warned about overwriting data and then prompted to create a passphrase for your drive:

```
$ sudo cryptsetup luksFormat /dev/sdX
WARNING!
========
This will overwrite data on /dev/sdX irrevocably.
Are you sure? (Type uppercase yes): YES
Enter passphrase: 
Verify passphrase:
```

### 4. Open the LUKS volume

Now you have a fully encrypted vault on your drive. Prying eyes,  including your own right now, are kept out of this LUKS partition. So to use it, you must open it with your passphrase. Open the LUKS vault with `cryptsetup open` along with the device location (`/dev/sdX`, in my example) and an arbitrary name for your opened vault:

```
$ cryptsetup open /dev/sdX vaultdrive
```

I use `vaultdrive` in this example, but you can name your vault anything you want, and you can give it a different name every time you open it.

LUKS volumes are opened in a special device location called `/dev/mapper`. You can list the files there to check that your vault was added:

```
$ ls /dev/mapper
control  vaultdrive
```

You can close a LUKS volume at any time using the `close` subcommand:

```
$ cryptsetup close vaultdrive
```

This removes the volume from `/dev/mapper`.

### 5. Create a filesystem

Now that you have your LUKS volume decrypted and open, you must  create a filesystem there to store data in it. In my example, I use XFS, but you can use ext4 or JFS or any filesystem you want:

```
$ sudo mkfs.xfs -f -L myvault /dev/mapper/vaultdrive
```

## Mount and unmount a LUKS volume

You can mount a LUKS volume from a terminal with the `mount` command. Assume you have a directory called `/mnt/hd` and want to mount your LUKS volume there:

```
$ sudo cryptsetup open /dev/sdX vaultdrive
$ sudo mount /dev/mapper/vaultdrive /mnt/ssd
```

LUKS also integrates into popular Linux desktops. For instance, when I  attach an encrypted drive to my workstation running KDE or my laptop  running GNOME, my file manager prompts me for a passphrase before it  mounts the drive.

  ![LUKS requesting passcode to mount drive](https://opensource.com/sites/default/files/uploads/luks-mount-gui.png)

## Permissions

We need to change the permissions of the newly-minted filesystem that is in the encrypted harddisk/file container.

Using the example in the example where `/dev/mapper/vaultdrive` has already been mounted to  `/mnt/ssd` with a command like: 

```
sudo mount /dev/mapper/vaultdrive /mnt/ssd
```

Open a terminal in your account and type **enter this command**: 

> **`sudo chown -R yourUserName /mnt/mount_point`**

For example if arksec has a mount point named sdd he should do:

```
sudo chown -R arksec /mnt/ssd
```

What this does is arksec changes the owner of all files in `ssd` to `arksec`, and he now has read and write access. Life is good.

If you decide you don't want R/W permissions anymore, just `sudo chown -R root /mnt/ssd` and you'll revoke your rights.
