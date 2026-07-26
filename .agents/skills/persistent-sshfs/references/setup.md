# Setup

## Requirements

- A Linux machine (the arcane-er, the better)
- `sshfs` installed (this IS the whole point — the script shells out to `sshfs`, `mount`, `fusermount`, and `ssh`)
- Working SSH key-based auth to every host you list — no passwords accepted, ever

Install `sshfs` per distro:

```sh
# Debian/Ubuntu
sudo apt install sshfs

# Fedora/RHEL
sudo dnf install fuse-sshfs

# Arch
sudo pacman -S sshfs
```

## Install the Script

```sh
wget https://raw.githubusercontent.com/psyb0t/persistent-sshfs/master/persistent-sshfs
chmod +x persistent-sshfs
sudo mv persistent-sshfs /usr/local/bin/
```

Note: default branch is `master` (no release tags exist) — the raw URL above tracks `master` directly.

## Mounts File Syntax

One mount per line, colon-delimited, no header, no comments support:

```
local_dir:user@host:port:remote_dir
```

Example (`mounts.txt`):

```
/home/bob/mnt/build-box:bob@10.0.0.5:22:/srv/build
/home/bob/mnt/backup-nas:bob@nas.example.com:2222:/volume1/backups
```

The script parses each line with a regex splitting on `:` into `mountpoint`, `userhost`, `port`, `remotedir` — keep entries exactly 4 fields, in that order. `local_dir` is created via `mkdir -p` if missing.

## Invocation

```sh
persistent-sshfs mounts.txt
```

Runs in the **foreground**. Per mount it:

1. Spawns a background subshell.
2. Calls `check_ssh_key_auth` — retries `ssh -o BatchMode=yes -o PasswordAuthentication=no -p <port> <user@host> exit` every 10s until it succeeds. This blocks forever on bad/missing keys — it will never fall back to a password prompt.
3. Enters a retry loop: check `mount | grep "on $mountpoint type fuse.sshfs"`; if already mounted, `break` immediately; otherwise run `sshfs -o reconnect -o port="$port" "$userhost:$remotedir" "$mountpoint"`; on failure, log and retry after 10s; **on success, `break` and the worker exits.** The loop's only job is to get the mount up once. After that, transient drops are ridden out by `sshfs`'s own `-o reconnect` — **this script does not poll-and-remount, and does not revive a mount that `sshfs` abandons entirely.**

The main process `wait`s on all per-mount background workers. Since each worker exits as soon as its mount is up, **once every mount succeeds the workers all exit and the main process exits 0** — it does not stay resident. The only way it keeps running is if some host's key auth never succeeds (step 2 loops forever). `SIGINT`/`SIGTERM` on a still-running instance trigger `cleanup()`: kill all workers, `fusermount -u` every mount currently up.

**Consequence for persistence:** because the process self-terminates once mounted, a single long-lived run is NOT a watchdog. To auto-remount a mount that later dies, **re-run the script** — a fresh run `break`s past whatever is still mounted and re-mounts only what's down. The persistence recipes below therefore use a **periodic re-run** (systemd timer / cron interval), not a single resident process.

## Environment Variables

| Variable | Default | What It Does |
|----------|---------|---------------|
| `LOG_LEVEL` | `INFO` | One of `DEBUG`, `INFO`, `ERROR`. Controls verbosity of the `log()` helper — higher levels (`DEBUG`) are more verbose. Anything unrecognized is treated as below `ERROR` (logs nothing). |

No CLI flags exist beyond the single positional `<mountpoints_file>` argument — usage is enforced with `Usage: $0 <mountpoints_file>` if you call it wrong.

## Keeping Mounts Persistent — Re-Run Recipes

The script mounts everything and then **exits** (see the runtime note above); it is not a resident daemon. So "persistence" = (a) get the mounts up at boot, and (b) re-run periodically to bring back any that died. `sshfs -o reconnect` covers the transient blips in between.

### systemd oneshot + timer (recommended)

A `oneshot` service does the mounting; a `timer` re-runs it on an interval so a fully-dropped mount gets re-established.

```ini
# /etc/systemd/system/persistent-sshfs.service
[Unit]
Description=persistent-sshfs — mount/remount SSHFS mounts
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/persistent-sshfs /home/bob/mounts.txt
Environment=LOG_LEVEL=INFO
User=bob
```

```ini
# /etc/systemd/system/persistent-sshfs.timer
[Unit]
Description=Re-run persistent-sshfs to remount anything that dropped

[Timer]
OnBootSec=30s
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now persistent-sshfs.timer
journalctl -u persistent-sshfs -f
```

> Note: don't use `Type=simple` + `Restart=on-failure` here — the process exits **0** once mounts are up, so `on-failure` never fires and you'd get a single mount pass with no re-check. The timer is what makes it recurring. (If a host is *unreachable*, the run blocks in the auth-retry loop instead of exiting; set a `TimeoutStartSec=` on the service if you don't want a hung host to hold the oneshot open.)

### cron — boot + interval

```sh
crontab -e
```

```
# bring mounts up at boot
@reboot     /usr/local/bin/persistent-sshfs /home/bob/mounts.txt >> /home/bob/persistent-sshfs.log 2>&1
# re-run every minute to remount anything that died (no-ops mounts already up)
* * * * *   /usr/local/bin/persistent-sshfs /home/bob/mounts.txt >> /home/bob/persistent-sshfs.log 2>&1
```

The `@reboot` line handles startup; the interval line is the actual remount-on-death mechanism, since the script exits after each pass. A run against already-up mounts returns almost immediately (each worker `break`s on "already mounted").

### tmux / screen (manual / interactive)

```sh
tmux new -d -s sshfs-mounts 'persistent-sshfs /home/bob/mounts.txt'
tmux attach -t sshfs-mounts   # to watch logs
```

Fine for a one-shot interactive bring-up on a workstation you're watching. Note it will still exit once mounts are up (or hang on an unreachable host) — for unattended survival across reboots and mount deaths, use the systemd timer or the cron interval above.

## Teardown

`Ctrl-C` (or `kill -TERM <pid>` / `systemctl stop persistent-sshfs`) runs `cleanup()`: kills background workers, then `fusermount -u`s every mount point still showing as `fuse.sshfs` in `mount`. If the process is `kill -9`'d, cleanup never runs — check manually:

```sh
mount | grep fuse.sshfs
fusermount -u /path/to/stale/mount
```
