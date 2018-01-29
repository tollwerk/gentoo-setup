phpMyAdmin
==========

Install phpMyAdmin to `/www/htdocs` by running

```sh
emerge phpmyadmin
chown -R apache:apache /www/htdocs/phpmyadmin
```

Create an Apache virtual host core `/etc/apache2/vhosts.d/default_pma_vhost.include` for inclusion by each account:

```Apache
ServerAdmin info@tollwerk.de
ErrorLog /var/log/apache2/ssl_error_log
<IfModule mod_log_config.c>
        TransferLog /var/log/apache2/ssl_access_log
</IfModule>
SSLEngine on
SSLOptions +StdEnvVars +ExportCertData
SSLVerifyDepth  5

DocumentRoot "/www/htdocs/phpmyadmin"

<Directory "/www/htdocs/phpmyadmin">
        RewriteEngine on
        RewriteCond %{REQUEST_METHOD} ^(TRACE|TRACK)
        RewriteRule .* - [F]

        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
```

___
[Back to Overview](01_Overview.md)
