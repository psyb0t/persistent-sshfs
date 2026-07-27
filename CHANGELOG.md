# Changelog

All notable changes per release.

## [v1.0.5] — 2026-07-27

### Fixed
- The Codex subsection of `## Agent integrations` was missing its install command — it told readers to add the `psyb0t/agents` marketplace and stopped. Added the missing line: `codex plugin add persistent-sshfs@psyb0t`.
- Corrected the invocation prose, which conflated two different situations: installed via the marketplace, the skill is invoked as `$persistent-sshfs:persistent-sshfs`; picked up automatically (no install) from a repo's own `.agents/skills/`, it's invoked as plain `$persistent-sshfs`.

## [v1.0.4] — 2026-07-27

### Added
- Agent-integration manifests: `.agents/.claude-plugin/plugin.json` and `.agents/.codex-plugin/plugin.json`, making the existing `.agents/skills/persistent-sshfs` skill installable as a plugin in Claude Code and Codex via the `psyb0t/agents` marketplace.
- `## Agent integrations` section in the README with the Claude Code, Codex, and OpenClaw install commands.

## [v1.0.3] — 2026-07-27

### Added
- Added a GitHub Actions CI status badge to the README.

## [v1.0.2] — 2026-07-27

### Added
- Added self-hosted version and license badges; wired a badges job into pipeline.yml.

## [v1.0.1] — 2026-07-26

### Fixed
- **Agent skill accuracy.** Corrected the `.agents/skills/persistent-sshfs/` docs to describe what the script actually does. The previous wording claimed an ongoing "watchdog" that "checks on a loop and remounts anything that dropped" and "blocks forever" — the code does not do that: each per-mount worker retries only the *initial* mount, `break`s the moment the mount is up, and exits, so in the happy path the process exits 0 once all mounts are established. Transient-drop survival is provided by `sshfs -o reconnect`, not by this script; genuine remount-on-death is achieved by *re-running* the script (its "mount whatever's currently down" logic). Rewrote the SKILL description/body and the setup.md runtime notes and persistence recipes accordingly (systemd oneshot+timer / cron interval instead of a single resident `Type=simple` service).

## [v1.0.0] — 2026-07-25

### Added
- **ClawHub agent skill + publish pipeline.** Added `.agents/skills/persistent-sshfs/` (SKILL.md + `references/setup.md`) documenting the mounts-file syntax (`local_dir:user@host:port:remote_dir`), key-only SSH auth, and how to run it, plus a tag-gated pipeline that publishes the skill to ClawHub.
