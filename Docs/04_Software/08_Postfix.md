Postfix & Co.
=============

Install Postfix and some other necessary packages by running

```sh
echo "mail-filter/maildrop -tools" >> /etc/portage/package.use/web
emerge pam_mysql cyrus-sasl postfix courier-imap maildrop postgrey
```

Create an Postfix user:

```sh
useradd -d /var/spool/mail -s /bin/false vmail
uid=`cat /etc/passwd | grep vmail | cut -f 3 -d :`
groupadd -g $uid vmail
mkdir -p /var/spool/mail/accounts
chown vmail: /var/spool/mail
```

Create a helper script `/usr/local/bin/mailrestart` to restart all mail services:

```sh
#!/bin/sh

/etc/init.d/courier-authlib restart
/etc/init.d/saslauthd restart
/etc/init.d/postfix restart
```

Postfix configuration
---------------------

Configure Postfix part 1: `/etc/postfix/main.cf`:

```sh
#myhostname = fqdn.example.com
#mydomain = example.com
#inet_interfaces = all
#mynetworks = 168.100.189.0/28, 127.0.0.0/8
#mail_spool_directory = /var/spool/mail
#home_mailbox = .maildir/

###########################################################
# CUSTOM CONFIGURATION
###########################################################

mydestination = localhost
local_transport = local
local_recipient_maps = $alias_maps $virtual_mailbox_maps unix:passwd.byname
local_destination_concurrency_limit = 2
default_destination_concurrency_limit = 20
tls_random_source = dev:/dev/urandom

smtpd_sasl_auth_enable = yes
smtpd_sasl2_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain =
broken_sasl_auth_clients = yes

#smtpd_use_tls = yes
#smtpd_tls_auth_only = yes
smtpd_tls_CAfile = /etc/ssl/postfix/startssl-ca.pem
smtpd_tls_key_file = /etc/ssl/postfix/tollwerk_de.decrypted.key
smtpd_tls_cert_file = /etc/ssl/postfix/tollwerk_de_with_chain.pem
smtpd_tls_loglevel = 3
smtpd_tls_received_header = yes
smtpd_tls_session_cache_timeout = 3600s

smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/postgrey, check_recipient_access mysql:/etc/postfix/mysql/recipient.cf, reject_unauth_destination, permit
smtpd_sender_restrictions = check_sender_access mysql:/etc/postfix/mysql/sender.cf
smtpd_client_restrictions = check_client_access mysql:/etc/postfix/mysql/client.cf
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/postgrey, check_recipient_access mysql:/etc/postfix/mysql/recipient.cf, reject_unauth_destination, permit

smtp_sasl_auth_enable = no
smtp_use_tls = yes
smtp_tls_note_starttls_offer = yes

alias_maps = mysql:/etc/postfix/mysql/aliases.cf
relocated_maps = mysql:/etc/postfix/mysql/relocated.cf
transport_maps = mysql:/etc/postfix/mysql/transport.cf
relay_domains = mysql:/etc/postfix/mysql/relay.cf
relay_recipient_maps = mysql:/etc/postfix/mysql/relay-recipient.cf
virtual_maps = mysql:/etc/postfix/mysql/virtual.cf
virtual_alias_maps = mysql:/etc/postfix/mysql/virtual.cf
virtual_mailbox_base = /var/spool/mail
virtual_minimum_uid = 1000
virtual_mailbox_maps = mysql:/etc/postfix/mysql/virtual-maps.cf
virtual_uid_maps = mysql:/etc/postfix/mysql/virtual-uid.cf
virtual_gid_maps = mysql:/etc/postfix/mysql/virtual-gid.cf
virtual_transport = mysql:/etc/postfix/mysql/transport.cf

maildrop_destination_recipient_limit = 1
maildrop_destination_concurrency_limit = 1
mailbox_size_limit = 307200000
message_size_limit = 40960000

enable_original_recipient = yes

header_checks = pcre:/etc/postfix/maps/header_checks
body_checks = pcre:/etc/postfix/maps/body_checks

smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, aECDH, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CDC3-SHA, KRB5-DE5, CBC3-SHA
smtpd_tls_dh1024_param_file = /etc/ssl/dhparams.pem
```

This configuration assumes that you are using MySQL-based lookup maps for various variables. Please install the necessary bindings into the directory `/etc/postfix/mysql`. You can let the tollwerk Cockpit do this for your.

Configure Postfix part 2: `/etc/postfix/master.cf`:

```sh
smtp      inet  n       -       n       -       -       smtpd -v
submission inet n       -       n       -       -       smtpd -v
...
maildrop  unix  -       n       n       -       -       pipe
  flags=ODRhu user=vmail:vmail argv=/usr/bin/maildrop -w 90 -d ${user}@${nexthop} ${recipient} ${user} ${nexthop} ${sender}
...
# YAA!
yaa    unix    -       n       n       -       -       smtp
```

Configure the known aliases `/etc/mail/aliases`:

```sh
# Well-known aliases -- these should be filled in!
root:           monitoring@tollwerk.de
operator:       monitoring@tollwerk.de
```

Start and enable boot-time start of Postfix:

```sh
/usr/bin/newaliases
/etc/init.d/postfix start
rc-update add postfix default
```

Postfix logging
---------------

Assuming you're using *syslog-ng* for logging, make Postfix write into separate log files by creating a configuration file `/etc/syslog-ng/postfix.conf` with the following contents:

```sh
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

Finally, add this line to *syslog-ng*'s configuration


Courier IMAP
------------

Adjust the POP3 (`/etc/courier-imap/pop3d.cnf`) and IMAP (`/etc/courier-imap/imapd.cnf`) configurations:

```sh
C=DE
ST=Bavaria
L=Germany
O=tollwerk GmbH
OU=Automatically-generated POP3 SSL key
CN=fqdn.example.com
emailAddress=postmaster@fqdn.example.com
```

Configure the IMAP service `/etc/courier-imap/imapd`:

```sh
# This is all on one line
IMAP_CAPABILITY="IMAP4rev1 CHILDREN NAMESPACE THREAD=ORDEREDSUBJECT THREAD=REFERENCES SORT AUTH=CRAM-MD5 AUTH=CRAM-SHA1 IDLE"

MAXPERIP=20

# Separate line
IMAPDSTART=YES

IMAP_ENHANCEDIDLE=1
```

Configure the encrypted IMAP service `/etc/courier-imap/imapd-ssl`:

```sh
IMAPDSSLSTART=YES
```

Configure the POP3 service `/etc/courier-imap/pop3d`:

```sh
MAXPERIP=20
POP3DSTART=YES
```

Configure the encrypted POP3 service `/etc/courier-imap/pop3d-ssl`:

```sh
POP3DSSLSTART=YES
```

Configure the authlib daemon `/etc/courier/authlib/authdaemonrc`:

```sh
authmodulelist="authmysql authpam"
```

Configure the MySQL authentication `/etc/courier/authlib/authmysqlrc`:

```sh
MYSQL_SERVER            localhost
MYSQL_USERNAME          <user>
MYSQL_PASSWORD          <password>
MYSQL_DATABASE          cockpit
MYSQL_USER_TABLE        login
MYSQL_CRYPT_PWFIELD    crypt
#MYSQL_CLEAR_PWFIELD     clear
MYSQL_UID_FIELD         uid
MYSQL_GID_FIELD         gid
MYSQL_LOGIN_FIELD       email
MYSQL_HOME_FIELD        homedir
MYSQL_MAILDIR_FIELD     maildir
```


Cyrus SASLv2
------------

Configure Cyrus SASLv2 `/etc/sasl2/smtpd.conf`:

```sh
mech_list: PLAIN LOGIN
pwcheck_method: saslauthd
allow_plaintext: true
auxprop_plugin: mysql
sql_engine: mysql
sql_hostnames: localhost
sql_user: cockpit
sql_passwd: w6%;qGj9
sql_database: cockpit
sql_select: select plain from login WHERE email='%u'
```

Configure the SASL daemon `/etc/conf.d/saslauthd`:

```sh
SASLAUTHD_OPTS="${SASLAUTH_MECH} -a rimap -r"
SASLAUTHD_OPTS="${SASLAUTHD_OPTS} -O localhost"
```

Start and enable boot-time start of the SASL daemon:

```sh
/etc/init.d/saslauthd start
rc-update add saslauthd default
```

Courier Maildrop
----------------

Configure Courier Maildrop `/etc/courier/maildropmysql.config`:

```sh
hostname localhost
database cockpit
dbuser               <user>
dbpw                 <password>
dbtable login
default_uidnumber 1007
default_gidnumber 1007
uid_field email
uidnumber_field uid
gidnumber_field gid
maildir_field maildir
homedirectory_field homedir
```

`/etc/maildroprc`:

```sh
if ( $SIZE < 26144 )
{
    exception {
       xfilter "/usr/bin/spamassassin"
    }
}

if (/^X-Spam-Flag: *YES/)
{
    exception {
        to "$HOME/$DEFAULT/.Spam/"
    }
}
else
{
    exception {
        to "$HOME/$DEFAULT"
    }
}
```

Yaa! Autoresponder
------------------

TODO

Postgrey
--------

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


@see http://www.phparchitecture.com/howto_show.php?id=2
___


[Back to Overview](01_Overview.md)
