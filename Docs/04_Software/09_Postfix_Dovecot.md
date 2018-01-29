Postfix & Co.
=============

In general, follow the excellent instructions of [Thomas Leistner](https://thomas-leister.de/mailserver-debian-stretch/) for setting up Postfix and other necessary programms. For your Gentoo box, however, please follow the adaptions below.

Unbound installation
--------------------

```
emerge unbound
unbound-anchor -a /etc/dnssec/root-anchors.txt
rc-update add unbound default
/etc/init.d/unbound start
```

If the command `dig @127.0.0.1 denic.de +short +dnssec` succeeds, you can add

```
nameserver 127.0.0.1
```

as the first line to your `/etc/resolv.conf`. Your local machine will now be used as DNS resolver.

Cockpit installation
--------------------

Cockpit is a Tollwerk specific, Laravel based web interface for managing mailboxes, forwardings and autoresponders and is not open source.

Dovecot installation
--------------------

Prepare the `vmail` user and directories:

```
mkdir -p /var/mail/accounts /var/mail/sieve/global
useradd -s /bin/false -d /var/mail vmail
chown -R vmail:vmail /var/mail
chmod -R 770 /var/mail
```

Install Dovecot:

```
echo "net-mail/dovecot sieve managesieve" >> /etc/portage/package.use/mail
emerge mysql dovecot
rm -r /etc/dovecot/dovecot.conf
rc-update add dovecot default
```

Replace the configuration `/etc/dovecot/dovecot.conf` with (and make sure to replace `mail.example.com` with your real FQDN):

```
# SSL configuration
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem
ssl_dh = </etc/dovecot/dh.pem
ssl_cipher_list = EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA256:EECDH:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!IDEA:!ECDSA:kEDH:CAMELLIA128-SHA:AES128-SHA
ssl_prefer_server_ciphers = yes

# Dovecot services
service imap-login {
    inet_listener imap {
        port = 143
    }
}
service pop3-login {
  inet_listener pop3 {
    port = 110
  }
}
service managesieve-login {
    inet_listener sieve {
        port = 4190
    }
}
service lmtp {
    unix_listener /var/spool/postfix/private/dovecot-lmtp {
        mode = 0660
        group = postfix
        user = postfix
    }
    user = vmail
}
service auth {
    unix_listener /var/spool/postfix/private/auth {
        mode = 0660
        user = postfix
        group = postfix
    }
    unix_listener auth-userdb {
        mode = 0660
        user = vmail
        group = vmail
    }
}

# Protocol settings
protocols = imap pop3 lmtp sieve
protocol imap {
    mail_plugins = $mail_plugins quota imap_quota imap_sieve
    mail_max_userip_connections = 20
    imap_idle_notify_interval = 29 mins
}
protocol lmtp {
    postmaster_address = postmaster@mail.example.com
    mail_plugins = $mail_plugins sieve
}

# Client authentication
disable_plaintext_auth = yes
auth_mechanisms = plain login
passdb {
    driver = sql
    args = /etc/dovecot/dovecot-sql.conf
}
userdb {
    driver = sql
    args = /etc/dovecot/dovecot-sql.conf
}

# Mail location
mail_uid = vmail
mail_gid = vmail
mail_privileged_group = vmail
mail_home = /var/mail/accounts/%d/%n
mail_location = maildir:~/.maildir:LAYOUT=fs

# Mailbox configuration
namespace inbox {
    inbox = yes
    separator = /
    mailbox Spam {
        auto = subscribe
        special_use = \Junk
    }
    mailbox Trash {
        auto = subscribe
        special_use = \Trash
    }
    mailbox Drafts {
        auto = subscribe
        special_use = \Drafts
    }
    mailbox Sent {
        auto = subscribe
        special_use = \Sent
    }
}

# Mail plugins
plugin {
    sieve_plugins = sieve_imapsieve sieve_extprograms
    sieve_before = /var/mail/sieve/global/spam-global.sieve
    sieve = file:/var/mail/sieve/%d/%n/scripts;active=/var/mail/sieve/%d/%n/active-script.sieve
    
    # Spam learning
    # From elsewhere to Spam folder
    imapsieve_mailbox1_name = Spam
    imapsieve_mailbox1_causes = COPY
    imapsieve_mailbox1_before = file:/var/mail/sieve/global/learn-spam.sieve

    # From Spam folder to elsewhere
    imapsieve_mailbox2_name = *
    imapsieve_mailbox2_from = Spam
    imapsieve_mailbox2_causes = COPY
    imapsieve_mailbox2_before = file:/var/mail/sieve/global/learn-ham.sieve

    sieve_pipe_bin_dir = /usr/bin
    sieve_global_extensions = +vnd.dovecot.pipe

    quota = maildir:User quota
    quota_exceeded_message = Benutzer %u hat das Speichervolumen überschritten. / User %u has exhausted allowed storage space.
}

auth_verbose=yes
auth_debug=yes
auth_debug_passwords=yes
mail_debug=yes
verbose_ssl=yes
log_path = /var/log/dovecot.log
# If you want everything in one file, just don't specify info_log_path
#info_log_path = /var/log/dovecot-info.log
```

For every mail domain you want to serve, create a matching SSL certificat via `certbot` add a section like this:

```
local_name mail.somedomain.com {
        ssl_cert = </etc/letsencrypt/live/mail.somedomain.com/fullchain.pem
        ssl_key = </etc/letsencrypt/live/mail.somedomain.com/privkey.pem
}
```

If you're planning to migrate emails from another (IMAP) server, add this to the end of your `dovecot.conf`:

```
# Migration from Golem
imapc_host = 80.82.223.126

# Authenticate as masteruser / masteruser-secret, but use a separate login user.
# If you don't have a master user, remove the imapc_master_user setting.
imapc_user = %u
#imapc_master_user = masteruser
imapc_password = "0LIMv?7&"

imapc_features = rfc822.size
imapc_features = $imapc_features fetch-headers
mail_prefetch_count = 20
```

Add the Dovecot database bindings by invoking the Cockpit CLI command `cockpit:dovecot`:

```
php /path/to/cockpit/artisan cockpit:dovecot /etc/dovecot/dovecot-sql.conf
```

Postfix installation
--------------------

```
echo "mail-filter/rspamd ~amd64" > /etc/portage/package.keywords/mail
echo "net-mail/dovecot sieve managesieve" >> /etc/portage/package.use/mail
emerge mysql postfix dovecot rspamd unbound
```

Chech your hosts `/etc/hosts` file for the proper entries:

```
127.0.0.1   localhost
127.0.1.1   mail.example.com  mail
```

Write the FQDN also to `/etc/mailname`:

```
echo $(hostname -f) > /etc/mailname
```

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
