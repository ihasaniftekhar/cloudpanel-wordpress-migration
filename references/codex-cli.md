# Codex CLI Execution

Use this mode when the migration controller is Codex CLI or another shell-first agent environment.

## SSH model

- Create a private controller-side temporary SSH control directory with `mktemp -d` and mode `0700`.
- Maintain one OpenSSH multiplexed connection per configured host alias when the route supports it.
- Keep host-key verification enabled. Stop on a changed or unexpected host key.
- Run migration work as explicit foreground `ssh host 'command'` calls rather than typing into interactive shells.
- Label output `[SOURCE]`, `[DEST]`, `[TRANSFER]`, or `[VERIFY]`.
- When prefixing or logging output through a pipeline, use `set -o pipefail` and preserve the SSH process exit status.

Recommended connection settings are `ControlMaster=auto`, `ControlPersist=15m`, `ServerAliveInterval=30`, `ServerAliveCountMax=3`, and `ConnectTimeout=15`.

If multiplexing is unavailable through a ProxyJump or execution sandbox, use explicit foreground SSH commands without multiplexing. Do not weaken host-key verification.

## Foreground work

Keep packaging, validation, transfer, restore, comparison, and cleanup commands attached. SSH master processes may persist in the background, but do not use `nohup`, `disown`, `setsid`, or a hidden background job for primary migration work.

Use real progress emitted by `scp`, `pv`, or the execution environment. For a quiet long command, announce it and poll the same process. Parallel transfer parts are allowed only after storage and routing inspection; validate the reassembled SHA-256 and include every part in cleanup.

Close remaining master connections and remove only the exact controller-created control directory after all work finishes.
