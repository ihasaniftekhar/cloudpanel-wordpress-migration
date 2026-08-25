---
name: cloudpanel-wordpress-migration
description: Safely migrate WordPress sites between CloudPanel servers over SSH, including discovery, validated file and database transfer, destination restoration, pre-DNS testing, and authorized temporary-backup cleanup. Use when moving one CloudPanel-hosted WordPress domain at a time.
---

# CloudPanel WordPress Migration

Migrate one WordPress domain at a time from an old CloudPanel server to a new CloudPanel server. Preserve the live source site, keep public DNS unchanged until the user approves cutover, and make every consequential step observable and reversible until final verification.

## Operating contract

- Treat the old server as the current production source.
- Never delete or modify live source website files or its live database.
- Do not change DNS or install a public certificate unless the user separately requests it.
- Inspect before changing. Resolve the exact domain, site user, document root, PHP version, database name, sizes, and destination conflicts from the servers rather than assuming them.
- Run commands one at a time in a persistent interactive SSH session when the environment supports it. Show the exact command before execution and expose its output immediately. Do not place secrets in commands, output, or chat.
- Use generated shell variables for passwords. Store a root-only temporary credential file only when operationally necessary; remove it after successful restoration and verification when the user has requested ephemeral credentials.
- Create temporary archives outside `htdocs`. Restrict them to root-only access.
- Preserve CloudPanel placeholder files by renaming them. Never delete or silently overwrite them.
- Use exact paths for cleanup. Do not use broad globs, unresolved variables, recursive deletion, or inferred targets.
- If a check fails, stop before cleanup. Keep all temporary artifacts needed for diagnosis or rollback and report the failure.

## Workflow

### 1. Discover and validate the source

Use read-only checks to determine:

- Nginx vhost and document root
- Linux site user and ownership
- PHP-FPM pool/version
- WordPress version
- database name and approximate size, without displaying its password
- site file size and free disk space

Run `wp db check`. Treat engine-specific messages such as an unsupported `CHECK TABLE` operation as notes only when the overall command succeeds and no table reports corruption.

Record a source entry count for later comparison. When security inspection is requested, perform it before packaging; do not treat generic string matches as proof of malware or authorize deletion from them.

### 2. Create validated temporary backups

Create a domain-specific root-only staging directory. Package the entire domain directory from its parent so the archive retains the domain directory name. Export the WordPress database through WP-CLI and gzip it.

Validate both archives before transfer:

- `gzip -t` for the database archive
- `tar -tzf ... >/dev/null` for the site archive
- SHA-256 for both files

If validation fails, do not transfer or restore.

### 3. Prepare the destination

Confirm that neither the domain vhost nor intended site user already exists. If either exists, inspect it and stop rather than overwriting it.

Create a CloudPanel PHP site using:

- the same PHP version as the source
- the `WordPress` vhost template
- a clear domain-derived site user unless the user specifies another name
- a strong generated password held in a shell variable

Create a database and database user using strong generated credentials. Keep secrets out of terminal output. If a temporary credential record is needed, write it with mode `0600` under `/root` and plan to delete it after final verification.

### 4. Transfer and verify

Transfer only the validated file and database archives. Keep the source copies until restoration succeeds.

Compute destination SHA-256 hashes and require exact matches with the source hashes. Do not extract or import a mismatched archive.

### 5. Restore without destructive replacement

Inspect the CloudPanel-created document root. Rename its placeholder `index.php` to a timestamped placeholder filename; do not delete it.

Extract with `tar --keep-old-files` into the site user's `htdocs` parent. Verify ownership before deciding whether any ownership correction is necessary.

Preserve the imported `wp-config.php` temporarily, import the database through CloudPanel, and update only the required database constants. Use quiet commands and shell variables so passwords are not displayed.

### 6. Verify before cleanup

All applicable checks must pass:

- new database credentials connect successfully
- `wp db check` succeeds on the destination
- `siteurl` and `home` are correct
- source and destination entry counts match, allowing only explicitly preserved destination placeholders
- WordPress core verifies against official checksums
- a local `curl --resolve` request to the new server returns the expected HTTP response without relying on public DNS
- site-specific Nginx and PHP error logs contain no migration-caused errors
- `nginx -t` succeeds; distinguish unrelated pre-existing warnings from errors

Do not claim public DNS, mail, cron, external APIs, checkout, or authenticated application flows are verified unless they were actually tested.

### 7. Clean up temporary artifacts

Only after every required verification succeeds, and only when the user authorized cleanup:

1. List exact temporary files on both servers.
2. Delete the exact file and database archives from the old server.
3. Delete the exact file and database archives plus temporary `wp-config.php` backup from the new server.
4. Delete the temporary plaintext migration-credentials file from the new server.
5. Verify each staging directory contains no files.

Cleanup authorization never includes the live source site, its live database, the restored destination, the active destination `wp-config.php`, or the preserved CloudPanel placeholder.

## Command patterns

Read [references/commands.md](references/commands.md) when executing a migration. Adapt every placeholder from live discovery and retain the safety gates above.

## Completion report

Report the domain, site user, PHP and WordPress versions, source sizes, checksum result, database result, local HTTP result, log result, DNS status, and exactly which temporary artifacts were deleted. State explicitly that the old live site and database were untouched.
