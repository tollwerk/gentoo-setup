Postfix & Co.
=============

In general, follow the excellent instructions of [Thomas Leistner](https://thomas-leister.de/mailserver-debian-stretch/) for setting up Postfix and other necessary programms. For your Gentoo box, however, please follow the adaptions below.

Additions
---------

### Setup script for `chroot`ed environment

Create and run `/usr/local/bin/setup-postfix-chroot` with the following contents (obtained from [here](https://raw.githubusercontent.com/tmtm/postfix/master/examples/chroot-setup/LINUX2)), otherwise Postfix might not be able to resolve DNS queries when run from within a `chroot`ed environment:

```sh
#! /bin/sh

CP="cp -p"

cond_copy() {
  # find files as per pattern in $1
  # if any, copy to directory $2
  dir=`dirname "$1"`
  pat=`basename "$1"`
  lr=`find "$dir" -maxdepth 1 -name "$pat"`
  if test ! -d "$2" ; then exit 1 ; fi
  if test "x$lr" != "x" ; then $CP $1 "$2" ; fi
}

set -e
umask 022

POSTFIX_DIR=${POSTFIX_DIR-/var/spool/postfix}
cd ${POSTFIX_DIR}

mkdir -p etc lib usr/lib/zoneinfo
test -d /lib64 && mkdir -p lib64

# find localtime (SuSE 5.3 does not have /etc/localtime)
lt=/etc/localtime
if test ! -f $lt ; then lt=/usr/lib/zoneinfo/localtime ; fi
if test ! -f $lt ; then lt=/usr/share/zoneinfo/localtime ; fi
if test ! -f $lt ; then echo "cannot find localtime" ; exit 1 ; fi
rm -f etc/localtime

# copy localtime and some other system files into the chroot's etc
$CP -f $lt /etc/services /etc/resolv.conf /etc/nsswitch.conf etc
$CP -f /etc/host.conf /etc/hosts /etc/passwd etc
ln -s -f /etc/localtime usr/lib/zoneinfo

# copy required libraries into the chroot
cond_copy '/lib/libnss_*.so*' lib
cond_copy '/lib/libresolv.so*' lib
cond_copy '/lib/libdb.so*' lib
if test -d /lib64; then
  cond_copy '/lib64/libnss_*.so*' lib64
  cond_copy '/lib64/libresolv.so*' lib64
  cond_copy '/lib64/libdb.so*' lib64
fi

postfix reload
```


### rspamd socket

For whatever reasons, communicating with *rspamd* using a TCP connection didn't succeed. To solve this, simply use the UNIX domain socket instead (in `/etc/postfix/main.cf`):

```sh
smtpd_milters = unix:/var/lib/rspamd/rspamd.sock
non_smtpd_milters = unix:/var/lib/rspamd/rspamd.sock
```

### Chrooted environment

The instructions imply that most Postfix services run in a chrooted environment. In that case, Postfix expects to find several resources under `/var/spool/postfix/path/to/file` instead of `/path/to/file`. Most importantly, you need to do this:

```sh
mkdir /var/spool/postfix/etc
cp -f /etc/services /var/spool/postfix/etc/services
chown -R postfix:root /var/spool/postfix/etc
```

### Yaa! Autoresponder

The Yaa! script itself is not really available anymore, but it's included in the tollwerk Cockpit distribution:

```sh
emerge IO-stringy DBD-mysql Net-Server DBI Net-Server-Mail

tar -xjvf /path/to/cockpit/resources/yaa/yaa_0.3.1.tar.bz2 -C /usr/local
cd /usr/local
ln -s yaa-0.3.1 yaa
cp yaa/conf/yaa.conf.sample yaa/conf/yaa.conf
```

Use the tollwerk Cockpit CLI command `cockpit:yaa` to get a proper set of configuration values for editing the Yaa! configuration file `yaa/conf/yaa.conf`:

```sh
php /path/to/cockpit/artisan cockpit:yaa
nano yaa/conf/yaa.conf
```

Start the Yaa! process (**needs to be done after every reboot!**):

```sh
/usr/local/yaa/bin/yaa.pl --action=start
```

**ATTENTION**: Don't forget to correct the Yaa! `$duration_interval` in the config file to `3600` once you got it to work! You can stop the service with:

```sh
/usr/local/yaa/bin/yaa.pl --action=stop
`````

### Postgrey

Install Postgrey:

```sh
emerge postgrey
```

Configure `/etc/conf.d/postgrey`

```sh
POSTGREY_TYPE="unix"
POSTGREY_DELAY=30
```

Start and enable boot-time start of Postgrey:
```sh
/etc/init.d/postgrey start
rc-update add postgrey default
```

Implement Postgrey checks into the Postfix configuration `/etc/postfix/main.cf`:

```sh
smtpd_recipient_restrictions =  permit_mynetworks
                                permit_sasl_authenticated
                                check_policy_service unix:private/postgrey
                                check_recipient_access mysql:/etc/postfix/sql/recipient-access.cf
mua_relay_restrictions = reject_non_fqdn_recipient,reject_unknown_recipient_domain,permit_mynetworks,permit_sasl_authenticated,check_policy_service unix:private/postgrey,reject
```

## TODO

* Siehe Gitlab-Issues
___


[Back to Overview](01_Overview.md)
