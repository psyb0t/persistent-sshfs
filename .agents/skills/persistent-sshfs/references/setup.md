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
3. Enters an infinite loop: check `mount | grep "on $mountpoint type fuse.sshfs"`; if not mounted, run `sshfs -o reconnect -o port="$port" "$userhost:$remotedir" "$mountpoint"`; on failure, log and retry after 10s; on success, break out and let the mount ride (SSHFS's own `-o reconnect` handles transient drops — this script's loop is what catches it if `sshfs` gives up entirely and unmounts).

The main process `wait`s on all per-mount background workers, so it never exits on its own. `SIGINT`/`SIGTERM` trigger `cleanup()`: kill all workers, `fusermount -u` every mount currently up.

## Environment Variables

| Variable | Default | What It Does |
|----------|---------|---------------|
| `LOG_LEVEL` | `INFO` | One of `DEBUG`, `INFO`, `ERROR`. Controls verbosity of the `log()` helper — higher levels (`DEBUG`) are more verbose. Anything unrecognized is treated as below `ERROR` (logs nothing). |

No CLI flags exist beyond the single positional `<mountpoints_file>` argument — usage is enforced with `Usage: $0 <mountpoints_file>` if you call it wrong.

## Keeping It Alive — Persistent Run Options

The script does not daemonize or fork itself. Pick one:

### systemd (recommended)

```ini
# /etc/systemd/system/persistent-sshfs.service
[Unit]
Description=persistent-sshfs mount watchdog
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/persistent-sshfs /home/bob/mounts.txt
Environment=LOG_LEVEL=INFO
Restart=on-failure
RestartSec=5
User=bob

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now persistent-sshfs
journalctl -u persistent-sshfs -f
```

### cron `@reboot`

```sh
crontab -e
```

```
@reboot /usr/local/bin/persistent-sshfs /home/bob/mounts.txt >> /home/bob/persistent-sshfs.log 2>&1
```

`@reboot` starts it once at boot; the script's own internal loop is what handles reconnects afterward. There's no cron-driven periodic re-check needed — the script IS the watchdog, cron just launches it.

### tmux / screen (manual / interactive)

```sh
tmux new -d -s sshfs-mounts 'persistent-sshfs /home/bob/mounts.txt'
tmux attach -t sshfs-mounts   # to watch logs
```

Fine for a workstation you control interactively; use systemd or cron for anything that needs to survive a reboot unattended.

## Teardown

`Ctrl-C` (or `kill -TERM <pid>` / `systemctl stop persistent-sshfs`) runs `cleanup()`: kills background workers, then `fusermount -u`s every mount point still showing as `fuse.sshfs` in `mount`. If the process is `kill -9`'d, cleanup never runs — check manually:

```sh
mount | grep fuse.sshfs
fusermount -u /path/to/stale/mount
```
