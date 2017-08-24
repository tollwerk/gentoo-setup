Kernel Setup
============

Preparations
------------

Review the **available profiles** and make sure `hardened/linux/amd64` is selected:

```sh
eselect profile list
eselect profile set 14
```

Synchronize the portage tree and **install the most recent version of portage** (just to be sure ...):

```sh
emerge --sync && emerge portage && etc-update
```

Install the **compiler cache** and **use flag editor** for convenience reasons and speeding up things:

```sh
emerge ccache ufed
```

Kernel
------

Emerge the **Kernel sources**, configure the Kernel according to your hardware and other needs and finally build it:

```sh
USE="-doc symlink" emerge gentoo-sources
cd /usr/src/linux
make menuconfig
make clean && make
```

If you need help to find out which Ethernet card you have, you may use `ethtool` to find out:

```sh
emerge -av ethtool 
ethtool -i eno1 
```

___
You have now prepared the Kernel for your new box. Time for some [more configuration](03_Basic-Configuration.md) before we can really use it.
