# CloudPanel WordPress Migration Skill

A portable AI-agent skill for safely migrating WordPress websites between CloudPanel servers over SSH.

The workflow was designed for Codex, Claude, Hermes, and other agents that support Markdown-based skills or reusable system instructions. It emphasizes source preservation, visible command execution, checksum-verified transfers, pre-DNS testing, and exact cleanup only after a successful restoration.

## What it covers

- Discovers the live CloudPanel site configuration instead of assuming paths or versions.
- Validates WordPress database tables before backup and after restoration.
- Archives and checksums website files and the database.
- Creates the destination site with the matching PHP version and WordPress vhost template.
- Preserves CloudPanel placeholder files rather than deleting or overwriting them.
- Tests the restored site locally without changing public DNS.
- Deletes temporary archives, rollback copies, and plaintext migration credentials only after all verification gates pass and cleanup is authorized.
- Never authorizes deleting the live source website or database.

## Installation

Clone or download this repository, then place the repository folder in the skills directory used by your agent.

Common locations include:

```text
Codex:  ~/.codex/skills/cloudpanel-wordpress-migration/
Claude: ~/.claude/skills/cloudpanel-wordpress-migration/
Hermes: use the configured skills directory for your installation
```

If an agent does not support skill folders, provide `SKILL.md` as reusable instructions and keep `references/commands.md` available as its execution reference.

## Usage

Ask the agent to use the skill and identify the source and destination SSH hosts:

```text
Use the cloudpanel-wordpress-migration skill to migrate example.com
from cloudpanel-old to cloudpanel-new. Keep DNS unchanged until I approve it.
```

The user remains responsible for providing authorized SSH access and approving any destructive cleanup not already included in the request.

## Repository contents

- `SKILL.md` — operating contract and migration workflow
- `references/commands.md` — adaptable CloudPanel, WP-CLI, SSH, archive, and verification command patterns

## License

MIT
