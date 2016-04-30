MySQL
=====

Emerge MySQL:

```sh
emerge mysql
```

Edit the `[mysqld]` section in `/etc/mysql/my.cnf` to look something like this:

```sh
[mysqld]
[mysqld]
default-time-zone       = +0:00
ft_min_word_len           = 3
character-set-server      = utf8
user                      = mysql
port                      = 3306
socket                    = /var/run/mysqld/mysqld.sock
pid-file                  = /var/run/mysqld/mysqld.pid
log-error                 = /var/log/mysql/mysqld.err
basedir                   = /usr
datadir                   = /mysql
tmpdir                    = /tmp
lc_messages_dir           = /usr/share/mysql
lc_messages               = de_DE

log-bin                   = mysql-bin
server-id                 = 1
sync_binlog               = 1
max_binlog_size           = 10M
binlog_format             = ROW
expire_logs_days          = 2
skip-external-locking
#skip-networking
#skip-name-resolve
skip-external-locking
#skip-show-database
safe-user-create          = 1
key_buffer_size           = 128M
sort_buffer_size          = 4M
join_buffer_size          = 4M
read_buffer_size          = 4M
read_rnd_buffer_size      = 8M
myisam_sort_buffer_size   = 64M
max_allowed_packet        = 24M
thread_cache_size         = 8
table_open_cache          = 3000
open_files_limit          = 10000
query_cache_type          = 1
query_cache_size          = 256M
query_cache_limit         = 4M
max_heap_table_size       = 512M
tmp_table_size            = 512M
long_query_time           = 4
slow_query_log            = 1
slow_query_log_file       = /mysql/log-slow-queries.log
myisam_recover
local-infile              = 0
max_connections           = 300
max_user_connections      = 200
max_connect_errors        = 999999
#bind-address             = 192.168.0.4
#bind-address             = 127.0.0.1
#log-update               = /path-to-dedicated-directory/hostname

# you need the debug USE flag enabled to use the following directives,
# if needed, uncomment them, start the server and issue
# #tail -f /tmp/mysqld.sql /tmp/mysqld.trace
# this will show you *exactly* what's happening in your server ;)

#log                                            = /tmp/mysqld.sql
#gdb
#debug                                          = d:t:i:o,/tmp/mysqld.trace
#one-thread

# the rest of the innodb config follows:
# don't eat too much memory, we're trying to be safe on 64Mb boxes
# you might want to bump this up a bit on boxes with more RAM
innodb_buffer_pool_size = 128M
#
# i'd like to use /var/lib/mysql/innodb, but that is seen as a database :-(
# and upstream wants things to be under /var/lib/mysql/, so that's the route
# we have to take for the moment
#innodb_data_home_dir           = /var/lib/mysql/
#innodb_log_group_home_dir      = /var/lib/mysql/
# you may wish to change this size to be more suitable for your system
# the max is there to avoid run-away growth on your machine
innodb_data_file_path = ibdata1:10M:autoextend:max:128M
# we keep this at around 25% of of innodb_buffer_pool_size
# sensible values range from 1MB to (1/innodb_log_files_in_group*innodb_buffer_pool_size)
innodb_log_file_size = 5M
# this is the default, increase it if you have very large transactions going on
innodb_log_buffer_size = 8M
# this is the default and won't hurt you
# you shouldn't need to tweak it
innodb_log_files_in_group=2
# see the innodb config docs, the other options are not always safe
innodb_flush_log_at_trx_commit = 1
innodb_lock_wait_timeout = 50
innodb_file_per_table
```

**Install the database** and prepare the internal tables:

```sh
/usr/share/mysql/scripts/mysql_install_db --basedir=/usr
```

**Start the database** and add it to the default runtime level.

```sh
rc-update add mysql default
/etc/init.d/mysql start
```

Set a **password for the MySQL root user** and secure the instance:

```sh
mysqladmin -u root password 'xyz123'
mysql --password=xyz123
mysql > drop database test;
mysql > use mysql;
mysql > delete from db;
mysql > delete from user where not (host="localhost" and user="root");
mysql > flush privileges;
mysql > quit
```
___
[Back to Overview](01_Overview.md)