Apache / PHP
============

What we're setting up here is a combination of Apache 2.4 with HTTP2, PHP-FPM and PHP 7. The procedure roughly follows the excellent [article of Nathan Zachary](http://z-issue.com/wp/apache-2-4-the-event-mpm-php-via-mod_proxy_fcgi-and-php-fpm-with-vhosts/) about this topic.

Installation
------------

Begin by configuring Apache to use the Event MPM and particular modules in your `/etc/portage/make.conf`:

```sh
APACHE2_MODULES="byrequests security cgi cgid actions alias auth_basic auth_digest authn_anon authn_dbd authn_dbm authn_core authz_core authn_default authn_file authz_dbm authz_default authz_groupfile authz_host authz_owner authz_user autoindex cache dav dav_fs dav_lock dbd deflate dir disk_cache env expires ext_filter file_cache filter headers http2 ident imagemap include info log_config logio mem_cache mime mime_magic mod_cgid mod_xml2enc negotiation proxy proxy_ajp proxy_balancer proxy_fcgi proxy_connect proxy_html proxy_http rewrite setenvif slotmem_shm so socache_shmcb speling status unique_id unixd userdir usertrack vhost_alias"
APACHE2_MPMS="event"
```

Define some use flags for the packages involved in `/etc/portage/package.use/web` (you will have to create this file):

```sh
www-servers/apache threads
dev-lang/php cgi fpm threads
app-eselect/eselect-php fpm
```

You might have to explicitly allow PHP 7 as long as it's considered experimental by adding the `~amd64` keyword to `/etc/portage/package.keywords/web` (you will have to create this file):

```sh
dev-lang/php ~amd64
```

Next, emerge Apache and PHP and prepare the document root:

```
emerge apache mod_xml2enc php
cd /var/www
rm -R *
ln -s /www localhost
mkdir -p /www/htdocs /www/vhtdocs /www/accounts
chown -R apache:apache /www
```

Configure Apache to use enable SSL and PHP in `/etc/conf.d/apache2`:

```sh
APACHE2_OPTS="-D SSL -D PROXY -D PHP -D HTTP2"
```

Ensure that `mod_cgid` is loaded instead of `mod_cgi` in `/etc/apache2/httpd.conf`:

```sh
# LoadModule cgi_module modules/mod_cgi.so
LoadModule cgid_module modules/mod_cgid.so
```

As PHP is compiled without the `apache2` use flag, there won't be a configuration file for `mod_php`, so we have to put the PHP handler definitions directly into `/etc/apache2/httpd.conf`. There will also be some more configuration, partly for [shared hosting](../05_Shared_Hosting/01_Webhosting.md):

```apache
###########################################################################
# CUSTOM CONFIGURATION
###########################################################################

# Add PHP handler and directory index
<IfDefine PHP>
    <IfModule mod_mime.c>
        AddHandler application/x-httpd-php .php .php5 .phtml
        AddHandler application/x-httpd-php-source .phps
    </IfModule>
    DirectoryIndex index.php index.phtml
</IfDefine>

# xml2enc module
LoadFile /usr/lib/libxml2.so
LoadModule xml2enc_module modules/mod_xml2enc.so

# Security settings
ServerTokens Minimal
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On
SSLCompression off
Header add Strict-Transport-Security "max-age=15768000"
# Strict-Transport-Security: "max-age=15768000 ; includeSubDomains"
SSLCipherSuite 'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'

# Shared hosting
ServerName localhost
Listen 80
Listen 443

# For IPv6 usage this would look more like this
# Listen 80.84.220.126:80
# Listen [2001:4ba0:92e0:126::1]:80

# Include virtual hosts
Include /www/accounts/*/vhost.conf
```

**Start Apache** and add it to the default runlevel:

```sh
rc-update add apache2 default
/etc/init.d/apache2 start
```

Please see the [shared hosting](../05_Shared_Hosting/01_Webhosting.md) section for information about creating Apache Virtual Hosts.


PHP pool manager configuration
------------------------------

To use multiple PHP versions and select a particular version for each virtual host, you need to repeat the following steps for each version you want to install (replace `$VERSION` with e.g. `7.0`). 

First, **configure the pool manager** by adapting its values in `/etc/php/fpm-php$VERSION/php-fpm.conf`. By default, the pool manager will consider all pool definitions matching `/etc/php/fpm-php$VERSION/fpm.d/*.conf`, but we will change this for [shared hosting purposes](../05_Shared_Hosting/01_Webhosting.md):

```ini
[global]
error_log = /var/log/apache2/php-fpm-$VERSION.log
events.mechanism = epoll
emergency_restart_threshold = 0
include=/www/accounts/*/fpm-$VERSION.conf
```
Create a version specific alias for the pool manager service, start it and add it to the default runlevel:

```sh
ln -s /etc/init.d/php-fpm /etc/init.d/php-fpm-php$VERSION
/etc/init.d/php-fpm-php$VERSION start
rc-update add php-fpm-php$VERSION default
```

Add virtual hosts and configurations as described under [shared hosting](../05_Shared_Hosting/01_Webhosting.md) and reload PHP-FPM after each adaption:

```sh
/etc/init.d/php-fpm-php$VERSION reload
```


PHP configuration
-----------------

As with the pool manager each version of PHP has to be configured separately via `/etc/php/fpm-php$VERSION/php.ini`:

```ini
short_open_tag = Off
realpath_cache_size = 1M
expose_php = Off
max_execution_time = 240
max_input_vars = 1500
memory_limit = 512M
display_errors = Off
log_errors = On
error_log = /var/log/apache2/php_$VERSION_log
default_charset = "utf-8"
enable_dl = Off
upload_max_filesize = 100M
post_max_size = 100M
allow_url_fopen = On
allow_url_include = On
date.timezone = Europe/Berlin
pcre.backtrack_limit = 1000000
```

___
[Back to Overview](01_Overview.md)