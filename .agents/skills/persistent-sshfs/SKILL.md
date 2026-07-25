---
name: persistent-sshfs
description: Bash tool that keeps SSHFS mounts alive — spawns one watchdog per mount, checks `mount | grep fuse.sshfs` on a loop, and remounts anything that dropped. Mounts are defined one-per-line in a plain-text mounts file (`local_dir:user@host:port:remote_dir`), no config format beyond that. Run it via `persistent-sshfs mounts.txt`, kept alive in the foreground under a `systemd` unit, a `cron @reboot` line, or a `tmux`/`screen` session — it blocks forever watching its background workers, so whatever runs it must not treat exit as success. Use when the user wants SSHFS mounts that survive dropped connections / auto-remount after a network blip / a persistent remote filesystem mount that doesn't need babysitting.
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

A bash script for the digital anarchist: it keeps your SSHFS mounts up. It checks whether each configured mount is alive, and if it's not, it remounts it — on a loop, forever, with the elegance of a cat walking across your keyboard.

For install steps, the mounts-file syntax, and how to run it under cron/systemd/a loop, see [references/setup.md](references/setup.md).

## Security & Safety

This script mounts remote filesystems over SSH using **your** SSH keys, with **no password fallback** — `check_ssh_key_auth` loops forever (retrying every 10s) until key-based auth succeeds, so a broken key means it hangs, not that it silently degrades to a password prompt.

- **Only point this at hosts you trust.** A mount is a two-way door: FUSE surfaces the remote filesystem locally, so whatever the remote server hands back gets exposed under your local mount point, and anything you write locally is sent straight to that remote host. A compromised remote server is now sitting on your filesystem.
- **Keep your SSH private keys safe** — this script assumes `ssh-agent` or an unencrypted-at-rest key is already usable non-interactively (`BatchMode=yes`). It does not manage keys, generate them, or prompt for passphrases; that's on your existing SSH setup.
- The script's `cleanup()` trap unmounts everything and kills its background workers on `SIGINT`/`SIGTERM`. If it's killed with `SIGKILL` (`kill -9`) or the process dies uncleanly, mounts are left dangling — check `mount | grep fuse.sshfs` and `fusermount -u <path>` by hand.

## When To Use

- You have one or more remote directories you want mounted locally via SSHFS, and you're tired of `sshfs` silently dying on you after a network blip, VPN drop, laptop sleep, or SSH timeout.
- You want a "set it and forget it" watchdog process (foreground, under `systemd`, `cron @reboot`, or a `tmux` session) that re-establishes mounts without you noticing they went down.
- You're managing multiple SSHFS mounts to different hosts/ports and want them all supervised by one process instead of hand-rolling a retry loop per mount.

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

This blocks in the foreground: it spawns one background worker per mount (each does key-auth-check, then an infinite mount-retry loop with a 10s backoff), then `wait`s on all of them. Ctrl-C (or `SIGTERM`) triggers `cleanup()`, which unmounts everything and kills the workers. It does not fork/daemonize itself — run it under something that keeps it alive (`systemd`, `cron @reboot` + `nohup`/`disown`, or a `tmux`/`screen` session). Set `LOG_LEVEL=DEBUG|INFO|ERROR` (default `INFO`) to control log verbosity.

See [references/setup.md](references/setup.md) for install, the full env var, and concrete cron/systemd/loop recipes.
