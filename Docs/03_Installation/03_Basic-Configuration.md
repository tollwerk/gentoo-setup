Basic configuration
===================

Partition table
---------------

If you haven't [done so already](../02_Hard-Drives.md#partition-preparations), it's finally time to create your `/etc/fstab` with about this content:
 
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

Network interface
-----------------

Your **network interface configuration** `/etc/conf.d/net` should look something like this (`ifconfig` style; you might have copied it over from the Live CD part):

```sh
config_enp9s0="80.84.220.126 broadcast 80.84.220.255 netmask 255.255.255.0"
routes_enp9s0="default via 80.84.220.1"
```

If you plan to use **IPv6**, this might more look like this (`ip` style here; watch out for the newline before the IPv6 gateway definition!):

```sh
config_enp9s0="80.84.220.126/24 2001:4ba0:92e0:0126::1/48"
routes_enp9s0="default via 80.84.220.1
default via 2001:4ba0:92e0::1"
```

Depending on the name of your network interface (here `enp9s0`), you will have to **create a symlink** in `/etc/init.d/` before you can **add the interface to your default runlevel** so that it will start up every time you boot the box:

```sh
cd /etc/init.d
ln -s net.lo net.enp9s0
rc-update add net.enp9s0 default
```

More config work ...
--------------------

**Set your hostname** and add the service to the default runlevel:

```sh
nano /etc /conf.d/hostname
# ...
rc-update add hostname default
```

Add your hostname to `/etc/hosts` (replace `$host` with the name you defined `/etc/hostname`):

```sh
127.0.0.1       localhost
80.84.220.126   $host.example.com
::1             localhost
```

Set your preferred **keyboard layout**:

```sh
nano /etc/conf.d/keymaps
```

If you don't want to use UTC as your server time, configure your **system clock** to use `local` in `/etc/conf.d/hwclock` and select the appropriate timezone (here: `Europe/Berlin`) by symlinking it:

```sh
nano /etc/conf.d/hwclock
rm -f /etc/localtime
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

Restrict the **locales** available on your system by editing `/etc/locale.gen` appropriately:

```sh
nano /etc/locale.gen
```

**Configure SSH** by editing `/etc/ssh/sshd_config` to temporarily allow root logins with regular passwords but don't forget to change this later as it might introduce weak security.
 
```sh
PermitRootLogin yes
```

Finally, make sure SSH has been added to the default runlevel to fire up on each machine boot:

```sh
rc-update add sshd default
```

___
We'll now finish our setup by [installing some basic packages](04_Basic-Software.md) before booting into our new system for the first time.