Live CD
=======

Start by [downloading](https://www.gentoo.org/downloads/) and booting into a current **Gentoo Minimal Installation CD**.

```sh
boot > gentoo
```

During the boot process, select the **keyboard layout** of your liking, e.g. `10` for German. When boot process has finished, check whether the **network interfaces** have already been brought up:

```sh
ifconfig
```

Manually **configure your interface(s)** in case the automatic setup has failed. You can find your interface names like `eth0` or `enp9s0` in the output of `ifconfig`:

```sh
net-setup enp9s0
```

**Test connectivity** e.g. by `ping`ing a server somewhere on the internet:

```sh
ping -c 3 google.com
```

Set a simple **password for the `root` user** that is used during the setup process only:

```sh
passwd root
```

More recent Live CDs require you to **enable root access via SSH** in `/etc/ssh/sshd_config`:

```sh
PermitRootLogin yes
```

**Fire up the SSH service** and log in with PuTTY using `root` as username and your password.

```sh
/etc/init.d/sshd start
```

___
Continue via SSH by [setting up the hard drives](02_Hard-Drives.md) (if this is a new install) or `chroot`ing into your system if you need to [troubleshoot something](03_Installation/05_Finish-Setup.md#troubleshooting).
