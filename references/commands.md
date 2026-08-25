# Command Patterns

Replace every angle-bracket placeholder with a value discovered from the target servers. These are patterns, not permission to run commands or bypass approval requirements.

## Source discovery

```sh
grep -E '^[[:space:]]*root |fastcgi_pass' /etc/nginx/sites-enabled/<domain>.conf
grep -R -E '^[[:space:]]*listen[[:space:]]*=[[:space:]]*127.0.0.1:<port>' /etc/php/*/fpm/pool.d
du -sh /home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp core version --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp config get DB_NAME --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp db size --size_format=MB --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp db check --path=/home/<site-user>/htdocs/<domain>
df -h /home
find /home/<site-user>/htdocs/<domain> -xdev -printf '.' | wc -c
```

## Package and validate

```sh
install -d -m 700 /root/migration-backups/<domain>
tar -czf /root/migration-backups/<domain>/<domain>-files.tar.gz -C /home/<site-user>/htdocs <domain>
sudo -u <site-user> wp db export - --path=/home/<site-user>/htdocs/<domain> | gzip -1 > /root/migration-backups/<domain>/<domain>-database.sql.gz
chmod 600 /root/migration-backups/<domain>/<domain>-files.tar.gz /root/migration-backups/<domain>/<domain>-database.sql.gz
gzip -t /root/migration-backups/<domain>/<domain>-database.sql.gz
tar -tzf /root/migration-backups/<domain>/<domain>-files.tar.gz >/dev/null
sha256sum /root/migration-backups/<domain>/<domain>-files.tar.gz /root/migration-backups/<domain>/<domain>-database.sql.gz
```

## Destination creation

```sh
getent passwd <site-user>
ls -ld /etc/nginx/sites-enabled/<domain>.conf /home/<site-user> 2>/dev/null
SITE_PASS=$(openssl rand -hex 24)
DB_PASS=$(openssl rand -hex 24)
clpctl site:add:php --domainName=<domain> --phpVersion=<php-version> --vhostTemplate='WordPress' --siteUser=<site-user> --siteUserPassword="$SITE_PASS"
clpctl db:add --domainName=<domain> --databaseName=<database> --databaseUserName=<db-user> --databaseUserPassword="$DB_PASS"
```

If a temporary credentials file is necessary:

```sh
umask 077
printf 'Site: %s\nSite user: %s\nSite password: %s\nDatabase: %s\nDatabase user: %s\nDatabase password: %s\n' '<domain>' '<site-user>' "$SITE_PASS" '<database>' '<db-user>' "$DB_PASS" > /root/<domain>-migration-credentials.txt
```

## Transfer and restore

From a workstation with both SSH aliases configured:

```sh
scp -3 <old-host>:/root/migration-backups/<domain>/<domain>-files.tar.gz <old-host>:/root/migration-backups/<domain>/<domain>-database.sql.gz <new-host>:/root/migration-backups/<domain>/
```

On the new server:

```sh
sha256sum /root/migration-backups/<domain>/<domain>-files.tar.gz /root/migration-backups/<domain>/<domain>-database.sql.gz
mv /home/<site-user>/htdocs/<domain>/index.php /home/<site-user>/htdocs/<domain>/index.php.cloudpanel-placeholder-<date>
tar --keep-old-files -xzf /root/migration-backups/<domain>/<domain>-files.tar.gz -C /home/<site-user>/htdocs
stat -c '%U:%G %a %n' /home/<site-user>/htdocs/<domain> /home/<site-user>/htdocs/<domain>/wp-config.php
cp -a /home/<site-user>/htdocs/<domain>/wp-config.php /root/migration-backups/<domain>/wp-config.php.old-server
clpctl db:import --databaseName=<database> --file=/root/migration-backups/<domain>/<domain>-database.sql.gz
sudo -u <site-user> wp config set DB_PASSWORD "$DB_PASS" --path=/home/<site-user>/htdocs/<domain> --quiet
```

If the database name or user differs between servers, update those constants separately after verifying the intended values.

## Destination verification

```sh
sudo -u <site-user> wp db check --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp option get siteurl --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp option get home --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp core verify-checksums --path=/home/<site-user>/htdocs/<domain>
find /home/<site-user>/htdocs/<domain> -xdev -printf '.' | wc -c
curl -k -sS -o /dev/null -w 'HTTP %{http_code}\nRedirect %{redirect_url}\n' --resolve <domain>:443:127.0.0.1 https://<domain>/
tail -n 30 /home/<site-user>/logs/nginx/error.log /home/<site-user>/logs/php/error.log
nginx -t
```

## Exact cleanup after successful verification

First list files on each server:

```sh
find /root/migration-backups/<domain> -maxdepth 1 -type f -printf '%s bytes  %p\n' | sort
```

Delete only explicitly resolved files; do not copy this command until every path is verified:

```sh
rm -- /root/migration-backups/<domain>/<domain>-files.tar.gz /root/migration-backups/<domain>/<domain>-database.sql.gz
rm -- /root/migration-backups/<domain>/wp-config.php.old-server /root/<domain>-migration-credentials.txt
```

Verify cleanup:

```sh
find /root/migration-backups/<domain> -maxdepth 1 -type f -printf '%p\n'
test ! -e /root/<domain>-migration-credentials.txt
```
