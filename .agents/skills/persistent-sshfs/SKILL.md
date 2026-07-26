---
name: persistent-sshfs
description: Bash tool that brings up SSHFS mounts and retries the initial mount until it connects. Spawns one background worker per mount that waits for key-based SSH auth (retrying forever, no password fallback), then mounts with `sshfs -o reconnect`. IMPORTANT — it is NOT a live watchdog: each worker `break`s and exits once its mount is up, so once all mounts succeed the process exits 0 (it only blocks forever if a host's key auth never succeeds). Transient-drop survival comes from `sshfs -o reconnect` itself, not this script; nothing re-mounts a mount that fully dies unless you RUN THE SCRIPT AGAIN (its "mount whatever's currently down" logic makes re-running the real recovery mechanism — schedule it via cron/timer for that). Mounts are defined one-per-line in a plain-text file (`local_dir:user@host:port:remote_dir`), no other config. Use when the user wants SSHFS mounts brought up reliably at boot/login with retry-until-connected + sshfs reconnect for blips.
homepage: https://github.com/psyb0t/persistent-sshfs
user-invocable: true
permissions:
  shell: "Runs bash as itself; shells out to sshfs, mount, fusermount, and ssh to check, mount, and tear down SSHFS mounts."
  network: "Opens SSH/SFTP connections to every host listed in your mounts file, on the port you specify. No other network activity."
  filesystem: "Mounts remote filesystems (FUSE) at the local mount points you configure, creating those directories if missing. Reads your mounts config file and uses your existing SSH key-based auth — it refuses password auth outright."
metadata:
  { "openclaw": { "emoji": "🔗", "requires": { "bins": ["sshfs", "bash"] } } }
---

# persistent-sshfs

A bash script for the digital anarchist: it brings up your SSHFS mounts and keeps retrying the initial connection until each one is up, with the elegance of a cat walking across your keyboard.

**What it actually does (read the code, not the marketing):** per mount, a background worker waits for key-based SSH auth to succeed (retrying every 10s, forever, no password fallback), then mounts once with `sshfs -o reconnect` and **exits its loop the moment the mount is up**. The ongoing "survive a dropped connection" behaviour is provided by `sshfs`'s own `-o reconnect` flag — *not* by this script. This script does **not** poll-and-remount in a loop after the initial mount, and it does **not** revive a mount that `sshfs` gives up on entirely. To get genuine remount-on-death you re-run the script (it mounts whatever is currently down and skips what's up) — so the real "keep it persistent" pattern is a cron job / systemd timer that re-runs it, not a single long-lived process.

For install steps, the mounts-file syntax, and how to run/re-run it under cron/systemd, see [references/setup.md](references/setup.md).

## Security & Safety

This script mounts remote filesystems over SSH using **your** SSH keys, with **no password fallback** — `check_ssh_key_auth` loops forever (retrying every 10s) until key-based auth succeeds, so a broken key means it hangs, not that it silently degrades to a password prompt.

- **Only point this at hosts you trust.** A mount is a two-way door: FUSE surfaces the remote filesystem locally, so whatever the remote server hands back gets exposed under your local mount point, and anything you write locally is sent straight to that remote host. A compromised remote server is now sitting on your filesystem.
- **Keep your SSH private keys safe** — this script assumes `ssh-agent` or an unencrypted-at-rest key is already usable non-interactively (`BatchMode=yes`). It does not manage keys, generate them, or prompt for passphrases; that's on your existing SSH setup.
- The script's `cleanup()` trap unmounts everything and kills its background workers on `SIGINT`/`SIGTERM`. If it's killed with `SIGKILL` (`kill -9`) or the process dies uncleanly, mounts are left dangling — check `mount | grep fuse.sshfs` and `fusermount -u <path>` by hand.

## When To Use

- You have one or more remote directories you want mounted locally via SSHFS, and you want the initial mount to keep retrying until the host is reachable and key auth works (e.g. at boot before the network/VPN is up) instead of failing once and giving up.
- You want `sshfs`'s built-in `-o reconnect` to ride out transient network blips, and — via a cron job / systemd timer that re-runs this script — to have any mount that fully died brought back automatically.
- You're managing multiple SSHFS mounts to different hosts/ports and want them all brought up by one invocation instead of hand-rolling a mount-and-retry per mount.

## When NOT To Use

- You need password-based SSH auth — this script refuses it outright (`check_ssh_key_auth` only accepts `BatchMode=yes` + `PasswordAuthentication=no`, i.e. key-based).
- A one-off, single mount you'll tear down yourself — plain `sshfs` is simpler if you don't need auto-remount.
- Non-SSHFS mounts (NFS, SMB, etc.) — this script is SSHFS-specific (`fuse.sshfs` is hardcoded into the aliveness check).

## Mount Config Walkthrough

Mounts are defined in a plain-text file, one mount per line, colon-delimited:

```
local_dir:user@host:port:remote_dir
```

Real example:

```
/home/bob/mnt/build-box:bob@10.0.0.5:22:/srv/build
/home/bob/mnt/backup-nas:bob@nas.example.com:2222:/volume1/backups
```

- `local_dir` — where it mounts locally. Created with `mkdir -p` if it doesn't exist yet.
- `user@host` — SSH target, passed straight to `ssh`/`sshfs`.
- `port` — SSH port, passed as `-p` to `ssh` and `-o port=` to `sshfs`.
- `remote_dir` — path on the remote host to mount.

No YAML, no JSON — just that one line format, one mount per line. Blank/malformed lines aren't specially handled, so keep the file clean.

## Running It

```sh
persistent-sshfs mounts.txt
```

It spawns one background worker per mount. Each worker first blocks in `check_ssh_key_auth` (retries `ssh -o BatchMode=yes -o PasswordAuthentication=no` every 10s until key auth succeeds — forever, no password fallback), then enters a mount loop that retries `sshfs -o reconnect` every 10s **until the mount is up, at which point it `break`s and the worker exits.** The main process `wait`s on all workers — so:

- **Happy path:** every mount comes up, every worker exits, and the main process **exits 0**. It does *not* stay resident. The mounts persist because they're now real FUSE mounts (with `sshfs -o reconnect` handling blips), not because this process is still running.
- **Degraded path:** if a host's key auth never succeeds, that worker loops in `check_ssh_key_auth` forever, so the main process never exits. That's the *only* reason it would "block forever".

Because it self-terminates once mounted, the way to get ongoing remount-on-death is to **re-run it periodically** (cron/systemd timer): a fresh run re-mounts anything currently down and no-ops anything already up. `Ctrl-C`/`SIGTERM` on a still-running instance triggers `cleanup()` (unmount everything, kill workers). Set `LOG_LEVEL=DEBUG|INFO|ERROR` (default `INFO`) for verbosity.

See [references/setup.md](references/setup.md) for install, the env var, and the cron/systemd-timer recipes that actually keep mounts persistent.
