System Sources
==============

Directory structure
-------------------

Start by creating the necessary directory structure (according to your [partition table](../02_Hard-Drives.md#partition-preparations)) and mount your partitions:

```sh
mount /dev/sda3 /mnt/gentoo
mkdir /mnt/gentoo/boot /mnt/gentoo/mysql /mnt/gentoo/www
mount /dev/sda1 /mnt/gentoo/boot
mount /dev/sda4 /mnt/gentoo/mysql
mount /dev/sdb1 /mnt/gentoo/www
```

Download & extract sources
--------------------------

You should find the current Gentoo sources somewhere [here](http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/) (hardened sources are [not recommended / supported any longer as of August 2017](https://www.gentoo.org/support/news-items/2017-08-19-hardened-sources-removal.html)).

```sh
cd /mnt/gentoo
wget http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20170817.tar.bz2
tar xvjpf stage*.tar.bz2
wget http://distfiles.gentoo.org/releases/snapshots/current/portage-latest.tar.bz2
tar xvjf portage*.tar.bz2 -C /mnt/gentoo/usr
mkdir /mnt/gentoo/usr/portage/distfiles
```

Basic configuration
-------------------

Edit `/mnt/gentoo/etc/portage/make.conf` to look something like this:

```sh
CHOST="x86_64-pc-linux-gnu"
CFLAGS="-O2 -march=nocona -mtune=nocona -fomit-frame-pointer -pipe -fno-strict-aliasing"
CXXFLAGS="${CFLAGS}"
ACCEPT_KEYWORDS="amd64"
PORTAGE_RSYNC_RETRIES="3"
MAKEOPTS="-j9"
AUTOCLEAN="yes"
FEATURES="ccache parallel-fetch"
CCACHE_SIZE="2G"
CCACHE_DIR="/tmp/ccache"
LINGUAS="de"
GENTOO_MIRRORS="ftp://ftp.tu-clausthal.de/pub/linux/gentoo ftp://ftp.uni-erlangen.de/pub/mirrors/gentoo ftp://ftp.tu-clausthal.de/pub/linux/gentoo"
USE="authlib bcmath bindist bzip2 cgi corefonts ctype curl encode exif ffmpeg
     filter ftp gd geoip gif hash imap inifile intl ipv6 jpeg jpeg2k json lcms
     mhash mmap mp3 mysql mysqli network nss pcntl pdo pic png posix
     postscript python sasl simplexml smtp soap sockets spell sqlite ssse3 svg
     symlink sysvipc tiff tokenizer truetype unicode userlocales x264 xattr
     xml xmlreader xmlrpc xmlwriter xsl xslt zip -dri -fortran -xorg"
APACHE2_MODULES="byrequests security cgi cgid actions alias auth_basic auth_digest authn_anon authn_dbd authn_dbm authn_core authz_core authn_default authn_file authz_dbm authz_default authz_groupfile authz_host authz_owner authz_user autoindex cache dav dav_fs dav_lock dbd deflate dir disk_cache env expires ext_filter file_cache filter headers http2 ident imagemap include info log_config logio mem_cache mime mime_magic mod_cgid mod_xml2enc negotiation proxy proxy_ajp proxy_balancer proxy_fcgi proxy_connect proxy_html proxy_http rewrite setenvif slotmem_shm so socache_shmcb speling status unique_id unixd userdir usertrack vhost_alias"
APACHE2_MPMS="event"
STAGE1_USE="gcc64"
ACCEPT_LICENSE="*"
PHP_TARGETS="php5-6 php7-0"
RUBY_TARGETS="ruby19 ruby22"
PYTHON_SINGLE_TARGET="python3_4"
PYTHON_TARGETS="python2_7 python3_4"
LINGUAS="de"

# Set PORTDIR for backward compatibility with various tools:
#   gentoo-bashcomp - bug #478444
#   euse - bug #474574
#   euses and ufed - bug #478318
PORTDIR="/usr/portage"
PAX_MARKINGS="XT"
```

The `MAKEOPTS` option should be set to `<number of the server's cores> + 1`. Find out how many cores your server has by typing:

```sh
nproc
```

Switch to the new installation
------------------------------

Mount the `/proc` file system:

```sh
mount -t proc none /mnt/gentoo/proc
```

Copy over the network interface configuration (and DNS server info

```sh
cp /etc/conf.d/net /mnt/gentoo/conf.d/net
cp /etc/resolv.conf /mnt/gentoo/etc
```

Switch to the new environment
```
chroot /mnt/gentoo /bin/bash
env-update
source /etc/profile
export PS1="(chroot) $PS1"
```
___
You are now "inside" the newly created Gentoo installation. Go on by [building the Kernel](02_Kernel.md).
