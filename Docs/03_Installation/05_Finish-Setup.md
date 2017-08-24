Finish Setup
============

Exit the temporary environment, unmount all file systems and reboot:

```sh
exit
cd / 
umount /mnt/gentoo/mysql /mnt/gentoo/www /mnt/gentoo/boot /mnt/gentoo/proc /mnt/gentoo
shutdown -r now
```

If all went well, you should be able to boot into your newly created system for the first time.

Troubleshooting
---------------

If your system doesn't boot properly and you need to fix something, enter it by [booting via the Live CD](../01_Live-CD.md) again. Simply mount your partitions and `chroot` into your system environment:

```sh
mount /dev/sda3 /mnt/gentoo
mount /dev/sda1 /mnt/gentoo/boot
mount /dev/sda4 /mnt/gentoo/mysql
mount /dev/sdb1 /mnt/gentoo/www
mount -t proc none /mnt/gentoo/proc
chroot /mnt/gentoo /bin/bash
env-update
source /etc/profile
export PS1="(chroot) $PS1"
```

___
Now that your new Gentoo box is up and running, you will want to [install some Software](../04_Software/01_Overview.md) on it.
