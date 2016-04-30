Basic software
==============

Useful tools
------------

Install some basic packages like the **system logger**, a **cron daemon**, a time protocol client, logfile rotation and other tools:

```sh
emerge syslog-ng vixie-cron ntp logrotate app-admin/sudo gentoolkit app-crypt/gnupg chkrootkit
rc-update add syslog-ng default
rc-update add vixie-cron default
rc-update add ntp-client default
```

Bootloader
----------

Prepare for and install the GRUB2 bootloader:

```sh
mount /boot
emerge grub
grub2-install --grub-setup=/bin/true /dev/sda
```

Now install the Kernel you [configured and compiled](02_Kernel.md#kernel) earlier:

```sh
cd /usr/src/linux
make install
```

Configure GRUB2 to recognize your kernel (repeat this step every time you add a new kernel):

```sh
mount /boot
grub2-mkconfig -o /boot/grub/grub.cfg
```

___
You should now be able to boot into the new system for the first time. Let's [finish the setup](05_Finish-Setup.md) and do just that!