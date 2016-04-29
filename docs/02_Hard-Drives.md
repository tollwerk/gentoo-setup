Hard Drives
===========

Partition preparations
----------------------

Start by drafting a **partition table**. Let's assume we have two physical hard drives (`/dev/sda` and `/dev/sdb`) where the first one will hold the operating system and the second one be dedicated to web hosting. Our partition table looks like this (paste into `/dev/mnt/gentoo/etc/fstab`):

```sh
/dev/sda1               /boot           ext2            noauto,noatime                  1 1
/dev/sda2               none            swap            sw                              0 0
/dev/sda3               /               ext4            noatime                         0 0
/dev/sda4               /mysql          ext4            noatime,nodev                   0 2
/dev/sdb1               /www            ext4            noatime,nodev,nosuid            0 2
/dev/dvd                /mnt/dvd        iso9660         noauto,ro                       0 0
none                    /proc           proc            defaults                        0 0
none                    /dev/shm        tmpfs           nodev,nosuid,noexec             0 0
```

### System drive (`/dev/sda`)

Use `fdisk` to create these partitions:

```sh
fdisk /dev/sda
```

#### /boot

```sh
fdisk > n
fdisk > p
fdisk > 1
fdisk >
fdisk > +128M
fdisk > a
fdisk > 1
```

#### /swap

```sh
fdisk > n
fdisk > p
fdisk > 2
fdisk >
fdisk > +2G
fdisk > t
fdisk > 2
fdisk > 82
```

#### /

```sh
fdisk > n
fdisk > p
fdisk > 3
fdisk >
fdisk > +45G
```

#### /mysql

```sh
fdisk > n
fdisk > p
fdisk > 4
fdisk >
fdisk >
```


### Web hosting drive (`/dev/sdb`)

```sh
fdisk /dev/sdb
```

#### /www

```sh
fdisk > n
fdisk > p
fdisk > 1
fdisk >
fdisk >
```

Filesystems
-----------

You need to **initialize file systems** on the newly created partitions. We typically use `ext2` for the `/boot` partition and `ext4` for all the others ([see why](https://mariadb.com/blog/what-best-linux-filesystem-mariadb)):

```sh
mke2fs /dev/sda1
mke2fs -j /dev/sda3
mke2fs -j /dev/sda4
mke2fs -j /dev/sdb1
```

### Initialize the `/swap` partition

```sh
mkswap /dev/sda2
swapon /dev/sda2
```

___
Continue by [installing the System Sources](installation/01_System-Sources.md).
