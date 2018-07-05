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

Cockpit is a Tollwerk specific, Laravel based web interface for managing mailboxes, forwardings and autoresponders and is not open source. The following steps assume you've got cockpit installed the typical way.

Dovecot installation
--------------------

Prepare the `vmail` user and directories:

```
mkdir -p /var/spool/mail/accounts /var/spool/mail/sieve/global
useradd -s /bin/false -d /var/mail vmail
chown -R vmail:vmail /var/spool/mail
chmod -R 770 /var/spool/mail
```

Install Dovecot:

```
echo "net-mail/dovecot sieve managesieve" >> /etc/portage/package.use/mail
emerge mysql dovecot
rm -r /etc/dovecot/dovecot.conf
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
mail_home = /var/spool/mail/accounts/%d/%n
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

For every mail domain you want to serve, create a matching SSL certificate via `certbot` add a section like this:

```
local_name mail.somedomain.com {
        ssl_cert = </etc/letsencrypt/live/mail.somedomain.com/fullchain.pem
        ssl_key = </etc/letsencrypt/live/mail.somedomain.com/privkey.pem
}
```

If you're planning to migrate emails from another (IMAP) server, add this to the end of your `dovecot.conf`:

```
# Migration from Golem
imapc_host = 123.123.123.123

# Authenticate as masteruser / masteruser-secret, but use a separate login user.
# If you don't have a master user, remove the imapc_master_user setting.
imapc_user = %u
#imapc_master_user = masteruser
imapc_password = "seCr3T"

imapc_features = rfc822.size
imapc_features = $imapc_features fetch-headers
mail_prefetch_count = 20
```

Create the Diffie Hellman parameters:

```
cd /etc/dovecot
openssl dhparam 4096
```

Add the Dovecot database bindings by invoking the Cockpit CLI command `cockpit:dovecot`:

```
php /path/to/cockpit/artisan cockpit:dovecot /etc/dovecot/dovecot-sql.conf
```

Add the following 3 files to `/var/mail/sieve/global/`:

* `spam-global.sieve`:

    ```
    require "fileinto";

    if header :contains "X-Spam-Flag" "YES" {
        fileinto "Spam";
    }

    if header :is "X-Spam" "Yes" {
        fileinto "Spam";
    }
    ```

* `learn-spam.sieve`:

    ```
    require ["vnd.dovecot.pipe", "copy", "imapsieve"];
    pipe :copy "rspamc" ["learn_spam"];
    ```

* `spam-global.sieve`:

    ```
    require ["vnd.dovecot.pipe", "copy", "imapsieve"];
    pipe :copy "rspamc" ["learn_ham"];
    ```

Start Dovecot:

```
rc-update add dovecot default
/etc/init.d/unbound start
```

Postfix installation
--------------------

```
echo "mail-filter/rspamd ~amd64" > /etc/portage/package.keywords/mail
emerge postfix rspamd
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

Replace the Postfix configuration file `/etc/postfix/main.cf` with:

```
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
inet_interfaces = 127.0.0.1, 85.114.145.198
myhostname = daigoro.tollwerk.net

maximal_queue_lifetime = 1h
bounce_queue_lifetime = 1h
maximal_backoff_time = 15m
minimal_backoff_time = 5m
queue_run_delay = 5m

tls_ssl_options = NO_COMPRESSION
tls_high_cipherlist = EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA256:EECDH:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!IDEA:!ECDSA:kEDH:CAMELLIA128-SHA:AES128-SHA

smtp_tls_security_level = dane
smtp_dns_support_level = dnssec
smtp_tls_policy_maps = mysql:/etc/postfix/sql/tls-policy.cf
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
smtp_tls_protocols = !SSLv2, !SSLv3
smtp_tls_ciphers = high
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

smtpd_tls_security_level = may
smtpd_tls_protocols = !SSLv2, !SSLv3
smtpd_tls_ciphers = high
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtpd_tls_cert_file=/etc/letsencrypt/live/daigoro.tollwerk.net/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/daigoro.tollwerk.net/privkey.pem

smtpd_sasl_type=dovecot
smtpd_sasl_path=private/auth
smtpd_sasl_auth_enable=yes

smtpd_milters = unix:/var/lib/rspamd/rspamd.sock
non_smtpd_milters = unix:/var/lib/rspamd/rspamd.sock
milter_protocol = 6
milter_mail_macros =  i {mail_addr} {client_addr} {client_name} {auth_authen}
milter_default_action = accept

smtpd_relay_restrictions =      reject_non_fqdn_recipient
                                reject_unknown_recipient_domain
                                permit_mynetworks
                                permit_sasl_authenticated
                                reject_unauth_destination
smtpd_recipient_restrictions =  permit_mynetworks
                                permit_sasl_authenticated
                                check_policy_service unix:private/postgrey
                                check_recipient_access mysql:/etc/postfix/sql/recipient-access.cf
smtpd_client_restrictions =     permit_mynetworks
                                check_client_access hash:/etc/postfix/without_ptr
                                reject_unknown_client_hostname
smtpd_helo_required =           yes
smtpd_helo_restrictions =       permit_mynetworks
                                reject_invalid_helo_hostname
                                reject_non_fqdn_helo_hostname
                                reject_unknown_helo_hostname
smtpd_data_restrictions =       reject_unauth_pipelining

mua_relay_restrictions =        reject_non_fqdn_recipient,reject_unknown_recipient_domain,permit_mynetworks,permit_sasl_authenticated,check_policy_service unix:private/postgrey,reject
mua_sender_restrictions =       permit_mynetworks,reject_non_fqdn_sender,reject_sender_login_mismatch,permit_sasl_authenticated,reject
mua_client_restrictions =       permit_mynetworks,permit_sasl_authenticated,reject

postscreen_access_list =        permit_mynetworks
                                cidr:/etc/postfix/postscreen_access
postscreen_blacklist_action =   drop
postscreen_greet_action =       drop

postscreen_dnsbl_threshold =    2
postscreen_dnsbl_sites =        ix.dnsbl.manitu.net*2
#                               zen.spamhaus.org*2
postscreen_dnsbl_action =       drop

virtual_alias_maps =            mysql:/etc/postfix/sql/aliases.cf
virtual_mailbox_maps =          mysql:/etc/postfix/sql/accounts.cf
virtual_mailbox_domains =       mysql:/etc/postfix/sql/domains.cf
virtual_transport =             mysql:/etc/postfix/sql/transport.cf
transport_maps =                mysql:/etc/postfix/sql/transport.cf
local_recipient_maps =          $virtual_mailbox_maps

mailbox_size_limit =            0
message_size_limit =            52428800
biff =                          no
append_dot_mydomain =           no

recipient_delimiter =           +
compatibility_level =           2
inet_protocols =                ipv4
debugger_command =              PATH=/bin:/usr/bin:/usr/local/bin; (strace -p $process_id 2>&1 | logger -p mail.info) & sleep 5
```

Replace the Postfix service configuration file `/etc/postfix/master.cf` with:

```
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================

smtp      inet  n       -       y       -       1       postscreen
    -o smtpd_sasl_auth_enable=no
smtpd     pass  -       -       y       -       -       smtpd
dnsblog   unix  -       -       y       -       0       dnsblog
tlsproxy  unix  -       -       y       -       0       tlsproxy
submission inet n       -       y       -       -       smtpd
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_sasl_type=dovecot
    -o smtpd_sasl_path=private/auth
    -o smtpd_sasl_security_options=noanonymous
    -o smtpd_client_restrictions=$mua_client_restrictions
    -o smtpd_sender_restrictions=$mua_sender_restrictions
    -o smtpd_relay_restrictions=$mua_relay_restrictions
    -o milter_macro_daemon_name=ORIGINATING
    -o smtpd_sender_login_maps=mysql:/etc/postfix/sql/sender-login-maps.cf
    -o smtpd_helo_required=no
    -o smtpd_helo_restrictions=
    -o cleanup_service_name=submission-header-cleanup
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
submission-header-cleanup unix n - n    -       0       cleanup
    -o header_checks=regexp:/etc/postfix/submission_header_cleanup
yaa       unix  -       n       n       -       -       smtp
```

Add the header cleanup rules to `/etc/postfix/submission_header_cleanup`:

```
/^Received:/            IGNORE
/^X-Originating-IP:/    IGNORE
/^X-Mailer:/            IGNORE
/^User-Agent:/          IGNORE
```

Create the Postfix MySQL bindings:

```
mkdir /etc/postfix/sql
php /www/accounts/cockpit/data/artisan cockpit:postfix /etc/postfix/sql
chmod -R 640 /etc/postfix/sql
```

Create some whitelists:

```
touch /etc/postfix/without_ptr
touch /etc/postfix/postscreen_access
postmap /etc/postfix/without_ptr
newaliases
```

### Setup script for `chroot`ed environment

The instructions imply that most Postfix services run in a chrooted environment. In that case, Postfix expects to find several resources under `/var/spool/postfix/path/to/file` instead of `/path/to/file`. Most importantly, you need to do this:

```sh
mkdir /var/spool/postfix/etc
chown -R postfix:root /var/spool/postfix/etc
```

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

### Logging & service start

Configure separate log files for Postfix by creating the file `/etc/syslog-ng/postfix.conf`:

```
destination mailinfo { file("/var/log/postfix/mail.info"); };
destination mailwarn { file("/var/log/postfix/mail.warn"); };
destination mailerr { file("/var/log/postfix/mail.err"); };

filter f_mail { facility(mail); };
filter f_info { level(info); };
filter f_warn { level(warn); };
filter f_err { level(err); };
log { source(src); filter(f_mail); filter(f_info); destination(mailinfo); };
log { source(src); filter(f_mail); filter(f_warn); destination(mailwarn); };
log { source(src); filter(f_mail); filter(f_err); destination(mailerr); };
```

and including it in `/etc/syslog-ng/syslog-ng.conf`:

```
@include "postfix.conf"
```

Add a logrotate snippet `/etc/logrotage.d/postfix`:

```
/var/log/postfix/mail.* {
  missingok
  notifempty
  weekly
  rotate 3
  compress
  sharedscripts
  postrotate
    /etc/init.d/postfix reload > /dev/null 2>&1 || true
  endscript
}
```

Restart the logging service and start Postfix:

```
mkdir /var/log/postfix
/etc/init.d/syslog-ng restart
rc-update add postfix default
/etc/init.d/postfix start
```

### Rspamd

Configure and install Rspamd as [described here](https://thomas-leister.de/mailserver-debian-stretch/#grundkonfiguration)


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
___


[Back to Overview](01_Overview.md)
