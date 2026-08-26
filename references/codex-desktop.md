# Codex Desktop Execution

Use this mode when the migration runs inside Codex Desktop and the user wants to watch commands and output live.

## Terminal model

- Open one persistent PTY SSH session for the source alias and one for the destination alias.
- Keep both sessions connected during active work. If either session expires, reconnect it and verify the server identity before continuing.
- Send one consequential command at a time to the appropriate session.
- Announce the server and sanitized command purpose before execution.
- Show command output directly in the conversation or open the live terminal panel when the product supports it.
- Poll the same session for quiet or long-running commands. Never resend a command merely because it has not produced output yet.
- Keep the user updated at least once per minute while work is running.

## Long operations

Keep archive creation, validation, transfer, extraction, and import attached to a visible foreground session. Use real progress output when available. For a quiet operation, report that it is still running and optionally use a separate read-only session to inspect actual file growth or process state.

The Desktop terminal session is an observability mechanism, not permission to combine dependent migration stages or bypass safety gates.

## Secrets

Do not print generated passwords. Hold them in the destination shell or a root-only temporary credential file when reconnectability requires it. Delete that file only after restoration, verification, live comparison, and authorized cleanup all succeed.
