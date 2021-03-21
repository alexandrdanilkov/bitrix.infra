
### File system permissions

All files touched by MySQL use UID/GID 1001, and PHP and Nginx use UID/GID 1000.
Running `scripts/fix-rights.sh` script would set the permissions appropriately for all
containers to run correctly.

It might be easier to switch everything to User and Group 1000 for consistency later.

### Relevant parts of Bitrix config

<details><summary>bitrix/php_interface/dbconn.php</summary>

```php
define('BX_CRONTAB_SUPPORT', true);

define("BX_USE_MYSQLI", true);
define("DBPersistent", true);
define("DELAY_DB_CONNECT", true);
$DBType = "mysql";
$DBHost = "localhost";
$DBName = "<DBNAME>";
$DBLogin = "<DBUSER>";
$DBPassword = "<DBPASSWORD>";
define('BX_TEMPORARY_FILES_DIRECTORY', '/tmp');

define("BX_CACHE_TYPE", "memcache");
define("BX_CACHE_SID", "prod"); // or "dev" in case of dev config
define("BX_MEMCACHE_HOST", "memcached");
define("BX_MEMCACHE_PORT", "11211");
define('BX_SECURITY_SESSION_MEMCACHE_HOST', 'memcached-sessions');
define('BX_SECURITY_SESSION_MEMCACHE_PORT', 11211);
```

</details>

<details><summary>bitrix/.settings.php</summary>

```php
  'session' => array (
  'value' =>
  array (
    'mode' => 'default',
    'handlers' =>
    array (
      'general' =>
      array (
        'type' => 'memcache',
        'host' => 'memcached-sessions',
        'port' => '11211',
      ),
    ),
  ),
  'readonly' => true,
  ),
  'connections' =>
  array (
    'value' =>
    array (
      'default' =>
      array (
        'className' => '\\Bitrix\\Main\\DB\\MysqliConnection',
        'host' => 'localhost',
        'database' => '<DBNAME>',
        'login' => '<DBUSER>',
        'password' => '<DBPASSWORD>',
        'options' => 2.0,
      ),
    ),
    'readonly' => true,
  ),
```

</details>

<details><summary>bitrix/.settings_extra.php</summary>

```php
<?php
return array(
  'cache' => array(
    'value' => array(
      'type' => 'memcache',
      'memcache' => array(
        'host' => 'memcached',
        'port' => '11211',
      ),
      'sid' => "prod" // or "dev" in case of dev config
    ),
  ),
);
?>
```

</details>

## What's inside?

### Core

- [Nginx](https://www.nginx.com/) with [brotli](https://github.com/google/ngx_brotli)
  proxying requests to php-fpm and serving static assets directly
- [php-fpm](https://www.php.net/manual/en/install.fpm.php) for bitrix with msmtp for mail sending
- [Percona MySQL](https://www.percona.com/software/mysql-database/percona-server)
  because of it's monitoring capabilities
- [memcached](https://memcached.org/) for bitrix cache, plus additional only for user sessions

### Optional

- PHP cron container with same settings as PHP serving web requests
- [adminer](https://www.adminer.org/) as phpmyadmin alternative for work with MySQL
- [pure-ftpd](https://www.pureftpd.org/project/pure-ftpd/) for ftp access
- [certbot](https://certbot.eff.org/) for HTTPS certificate generation
- [zabbix-agent](https://www.zabbix.com/zabbix_agent) for monitoring

## File structure

### /config

- `cron/php-cron.cron` is a list of cron tasks to run in php-cron container,
  only `cron_events.php` is required for Bitrix and others are specific to this site,
  [must](http://manpages.ubuntu.com/manpages/trusty/man8/cron.8.html) be owned by root:root
  and have access rights 0644 - fixable by running `scripts/fix-rights.sh`

- `cron/host.cron` is a list of cron tasks to run on the host machine

- `mysql/my.cnf` is a MySQL configuration, applied on top of package-provided my.cnf

- `nginx` directory contains the build Dockerfile, as well as following (HTTPS) configuration:
  - pagespeed setup
  - bitrix proxy, separate for dev and prod
  - adminer proxy
  - HTTP to HTTPS redirects
  - stub status page listening on localhost for Zabbix monitoring

- `php-fpm` directory contains the build Dockerfile and php configuration, applied on top of package-provided one

### /logs

`mysql`, `nginx`, `php` logs. cron and msmtp logs will be written to the `php` directory.

### /scripts

Bunch of scripts, see their source code for purpose and comments.

### /web

Site files in directories `web/prod` and `web/dev`.

### /private

- `private/environment` is a directory with environment files for docker-compose

    - `private/environment/mysql.env` should contain the following variables:

      ```bash
      MYSQL_ROOT_PASSWORD=mysql_root_password
      MYSQL_USER=bitrix_user
      MYSQL_PASSWORD=bitrix_mysql_password
      ```

    - `private/environment/ftp.env` should contain the following variables:
  
      ```bash
      FTP_USER_NAME=ftp_username
      FTP_USER_PASS=ftp_password
      ```

    - `private/environment/certbot.env` should contain Yandex PDD API key for [actionm/certbot-dns-pddyandex](https://github.com/actionm/certbot-dns-pddyandex/):

      ```bash
      # Get your API key from https://pddimp.yandex.ru/api2/admin/get_token_result
      API_KEY="insert_key_there"
      ```

- `private/letsencrypt` directory will be filled with certificates after certbot run (see instruction below)

- `private/mysql-data` directory will be filled with database data automatically after the start of mysql container

- `private/mysqld` directory will contain MySQL unix socket for connections without network

- `private/msmtprc` is a file with [msmtp configuration](https://wiki.archlinux.org/index.php/Msmtp)

## Routines

## Cleaning cache

There are two memcached instances in use, one for site cache and another for sessions. Here are the commands
to clean them completely:

```shell
# to flush site cache
echo "flush_all" | docker exec -i memcached /usr/bin/nc 127.0.0.1 11211
# to flush all user sessions
echo "flush_all" | docker exec -i memcached-sessions /usr/bin/nc 127.0.0.1 11211
```

[Here](https://github.com/memcached/memcached/wiki/Commands) is the complete list of commands you can send to it.

## Certificate renewal

At this moment, DNS verification of a wildcard certificate is set up automatically trough [Yandex PDD](https://yandex.com/dev/connect/directory/api/concepts/domains/dns-records-via-pdd.html).

To renew the certificate manually, run the following command and follow the interactive prompt:

```shell
docker-compose run --rm --entrypoint "\
  certbot certonly \
    --email mail@example.com \
    -d example.com -d *.example.com \
    --agree-tos \
    --manual \
    --preferred-challenges dns" certbot
```