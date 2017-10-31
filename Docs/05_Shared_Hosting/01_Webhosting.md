Shared Webhosting Setup
=======================

We mostly use our Gentoo boxes for hosting multiple web projects / accounts simultaneously. The necessary infrastructure mostly lives on the `/www` partition:

```sh
/www
|-- accounts
|   |-- account1
|   |   |-- fpm-5.6.conf
|   |   |-- certbot.ini
|   |   `-- vhost.conf
|   `-- account2
|       |-- fpm-7.0.conf
|       `-- vhost.conf
|-- htdocs
`-- vhtdocs
    |-- account1
    |   `-- index.php
    `-- account2
        `-- index.php
```

The directory `/www/vhtdocs` contains the web accounts, each of them in a separate directory containing their respective data.

For each of the accounts there's also a directory below `/www/accounts` containing the corresponding configuration files:

* `vhost.conf` contains the [Apache](../04_Software/05_Apache-PHP.md#installation) Virtual Host definition
* `fpm-*.conf` contains the [PHP-FPM pool](../04_Software/05_Apache-PHP.md#php-pool-manager-configuration) definition. The filename decides which pool manager / PHP version to use. There may only be one pool definition per account.
* `certbot.ini` contains all information necessary for issuing [Certbot](../04_Software/06_Certbot.md) certificates for this account.

Apache Virtual Host (`vhost.conf`)
----------------------------------

A typical account VHost looks like this:

```apache
<VirtualHost *:443>
        ServerName mysql.example.com
        Include /etc/apache2/vhosts.d/default_pma_vhost.include
        Include /www/accounts/account1/vhost_fpm.include
        Include /www/accounts/account1/vhost_ssl.include
</VirtualHost>

<VirtualHost *:443>
        SSLEngine on
        SSLOptions +StdEnvVars +ExportCertData
        SSLVerifyDepth  5
        Include /www/accounts/account1/vhost_ssl.include
        Include /www/accounts/account1/vhost.include
</VirtualHost>

<VirtualHost *:80>
        Include /www/accounts/account1/vhost.include
</VirtualHost>
```

The virtual host uses a couple of auxiliary includes to not repeat itself.

### General vhost properties (`vhost.include`)

There's `vhost.include` for the general host definition:

```Apache
ServerName example.com
ServerAlias *.example.com
CustomLog /var/log/apache2/access_staging_log combined
DocumentRoot "/www/vhtdocs/account1"
<Directory "/www/vhtdocs/account1">
        RewriteEngine on
        RewriteCond %{REQUEST_METHOD} ^(TRACE|TRACK)
        RewriteRule .* - [F]

        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>

Include /www/accounts/account1/vhost_fpm.include
```

### SSL properties (`vhost_ssl.include`)

There's `vhost_ssl.include` for the SSL settings:

```Apache
SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
SSLCertificateFile /etc/letsencrypt/live/example.com/cert.pem
SSLCertificateChainFile /etc/letsencrypt/live/example.com/chain.pem
```

### PHP properties (`vhost_fpm.include`)

And then there's `vhost_fpm.include` for the PHP settings:

```Apache
<FilesMatch "\.php$">
        SetHandler "proxy:unix:///var/run/php-fpm/account1.sock|fcgi://account1/"
</FilesMatch>

<Proxy fcgi://account1/ enablereuse=on max=10>
</Proxy>
```

PHP pool manager definition (`fpm-*.conf`)
------------------------------------------

A pool manager definition could look like this:

```ini
;; example.com

[account1]
listen = /var/run/php-fpm/account1.sock
listen.owner = account1
listen.group = apache
listen.mode = 0660
user = account1
group = apache
pm = dynamic
pm.start_servers = 3
pm.max_children = 100
pm.min_spare_servers = 2
pm.max_spare_servers = 5
pm.max_requests = 10000
request_terminate_timeout = 300
```

Please don't forget to **create a Linux user** for every `listen.owner` / `user` value you want to use in the pool definition:

```sh
useradd -g apache -s /bin/false account1
```


Certbot configuration (`certbot.ini`)
-------------------------------------

The Certbot definition typically looks like this (of which only the first two lines need to be adapted for each account):

```ini
domains = example.com, www.example.com
webroot-path = /www/vhtdocs/example

rsa-key-size = 4096
email = info@tollwerk.de
text = True
authenticator = webroot
renew-by-default = true
agree-tos = true
```

Issuing a certificate is as easy as:

```sh
certbot -c /www/accounts/example/certbot.ini certonly
/etc/init.d/apache2 reload
```

For this to have any effect, please make sure that your Apache Virtual Host really defines an SSL container.

A bash script `/usr/local/bin/certbot-all` updating all certificates in a row (e.g. to be run by cron) could look like this:

```sh
#!/bin/sh

for certbotini in `find /www/accounts/* -name "certbot.ini"`; do
    account="`dirname $certbotini`";
    echo "Certbot \"`basename $account`\"!";
    certbot -c $certbotini certonly;
done;

/etc/init.d/apache2 reload;
```

A cronjob running this command once a month would look like this:

```sh
0       0       1       *       *       /usr/local/bin/certbot-all
```

**UPDATE**: Alternatively you can use the new `renew` command of Certbot:

```sh
0       0       1       *       *       /usr/bin/certbot renew
```

Logrotate settings
------------------

Add the following script to `/etc/logrotate.d/accounts` to let `logrotate` take care of your account logs:

```bash
# Accounts logrotate snippet for Gentoo Linux
#
/www/accounts/*/log/*log {
  su root account
  copytruncate
  missingok
  notifempty
  sharedscripts
  postrotate
  /etc/init.d/apache2 reload > /dev/null 2>&1 || true
  endscript
}
```
