# Changelog

All notable changes per release.

## [v1.0.1] — 2026-07-26

### Fixed
- **Agent skill accuracy.** Corrected the `.agents/skills/persistent-sshfs/` docs to describe what the script actually does. The previous wording claimed an ongoing "watchdog" that "checks on a loop and remounts anything that dropped" and "blocks forever" — the code does not do that: each per-mount worker retries only the *initial* mount, `break`s the moment the mount is up, and exits, so in the happy path the process exits 0 once all mounts are established. Transient-drop survival is provided by `sshfs -o reconnect`, not by this script; genuine remount-on-death is achieved by *re-running* the script (its "mount whatever's currently down" logic). Rewrote the SKILL description/body and the setup.md runtime notes and persistence recipes accordingly (systemd oneshot+timer / cron interval instead of a single resident `Type=simple` service).

## [v1.0.0] — 2026-07-25

### Added
- **ClawHub agent skill + publish pipeline.** Added `.agents/skills/persistent-sshfs/` (SKILL.md + `references/setup.md`) documenting the mounts-file syntax (`local_dir:user@host:port:remote_dir`), key-only SSH auth, and how to run it, plus a tag-gated pipeline that publishes the skill to ClawHub.
