# Command Patterns

Replace every angle-bracket placeholder with a value discovered from the target servers. These are patterns, not permission to run commands or bypass approval requirements.

## Controller SSH setup

Keep control sockets in a private controller-side temporary directory, never inside a website tree:

```sh
SSH_CONTROL_DIR=$(mktemp -d "${TMPDIR:-/tmp}/cloudpanel-migrate-ssh.XXXXXX")
chmod 700 "$SSH_CONTROL_DIR"
SSH_CONTROL_PATH="$SSH_CONTROL_DIR/cm-%C"
```

Use the configured host aliases and retain normal host-key verification:

```sh
ssh -MNf -o ControlMaster=yes -o ControlPersist=15m -o ControlPath="$SSH_CONTROL_PATH" -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <old-host>
ssh -MNf -o ControlMaster=yes -o ControlPersist=15m -o ControlPath="$SSH_CONTROL_PATH" -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <new-host>
ssh -o ControlPath="$SSH_CONTROL_PATH" -O check <old-host>
ssh -o ControlPath="$SSH_CONTROL_PATH" -O check <new-host>
```

Run actual work in the foreground with explicit server labels. When prefixing or logging pipeline output, use `set -o pipefail` so a failed SSH command is not masked by `sed` or `tee`.

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

If foreground `scp` is too slow and inspection proves parallel split transfer is safe, create root-only contiguous parts, transfer each visibly, reassemble in numeric order, and require the reassembled SHA-256 to match the original before extraction. Inventory and delete every split part during authorized cleanup.

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

## Fresh live source-to-destination comparison

Run these after destination verification and before cleanup. Generate comparable manifests from each document root. Exclude only the active `wp-config.php` and the explicitly preserved destination placeholder; report those exclusions.

```sh
cd /home/<site-user>/htdocs/<domain>
find . -xdev -type f ! -name wp-config.php ! -name 'index.php.cloudpanel-placeholder-*' -printf '%P\0' | sort -z | xargs -0 -r sha256sum
find . -xdev -type l -printf '%P -> %l\n' | sort
find . -xdev -type f -printf '%s\n' | awk '{count++; bytes += $1} END {printf "files=%d bytes=%.0f\n", count, bytes}'
```

Compare the source and destination manifest outputs on the controller without modifying either server. Absolute paths must not enter the manifests because document roots differ.

For a logical database comparison, capture table names and row counts, then create near-simultaneous normalized exports:

```sh
sudo -u <site-user> wp db tables --all-tables-with-prefix --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp db query "SELECT TABLE_NAME, TABLE_ROWS FROM information_schema.TABLES WHERE TABLE_SCHEMA = DATABASE() ORDER BY TABLE_NAME" --skip-column-names --path=/home/<site-user>/htdocs/<domain>
sudo -u <site-user> wp db export - --skip-comments --path=/home/<site-user>/htdocs/<domain> | sha256sum
sudo -u <site-user> wp db size --size_format=b --path=/home/<site-user>/htdocs/<domain>
```

Treat `TABLE_ROWS` as approximate for InnoDB. A normalized export hash or equivalent logical comparison carries more weight. If a live site changes between the two captures, report drift and do not force equality or clean up rollback artifacts until the user decides how to proceed.

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

Close the exact controller-created SSH masters after completion:

```sh
ssh -o ControlPath="$SSH_CONTROL_PATH" -O exit <old-host>
ssh -o ControlPath="$SSH_CONTROL_PATH" -O exit <new-host>
rm -r -- "$SSH_CONTROL_DIR"
```
