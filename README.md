# persistent-sshfs

[![CI](https://github.com/psyb0t/persistent-sshfs/actions/workflows/pipeline.yml/badge.svg?branch=master)](https://github.com/psyb0t/persistent-sshfs/actions/workflows/pipeline.yml)
[![version](https://raw.githubusercontent.com/psyb0t/persistent-sshfs/badges/version.svg)](https://github.com/psyb0t/persistent-sshfs/releases)
[![license](https://raw.githubusercontent.com/psyb0t/persistent-sshfs/badges/license.svg)](LICENSE)

Welcome to the chaotic depths of `persistent-sshfs`, where stability meets anarchy, and your SSHFS mounts cling to your server like a rebel to a cause. No more do we bow down to the tyranny of lost connections and manual remounts; `persistent-sshfs` is here to disrupt the status quo.

## What Is This Madness?

`persistent-sshfs` is a bash script for the digital anarchist, a tool that ensures your SSHFS mounts are as persistent as your disdain for authority. It tirelessly checks if your mounts are alive, and if they're not, it revives them like a phoenix from the ashes. And it does all this with the elegance of a cat walking across your keyboard – silently, effectively, and with a touch of chaos.

## Getting Started

To join the revolution, you'll need:

- A Linux machine (the more arcane, the better)
- SSHFS (because that's the whole point)
- Disdain for manual labor (optional but recommended)

### Installation

Installing `persistent-sshfs` is as easy as spray-painting an anarchy symbol on a firewall. Just run these commands, and watch the magic unfold:

1. Use `wget` to pull the script straight from the ethers of the internet:

   ```sh
   wget https://raw.githubusercontent.com/psyb0t/persistent-sshfs/master/persistent-sshfs
   ```

2. Empower the script with the ability to act:

   ```sh
   chmod +x persistent-sshfs
   ```

3. Move it to `/usr/local/bin` to unleash it upon your system:
   ```sh
   sudo mv persistent-sshfs /usr/local/bin/
   ```

Now, `persistent-sshfs` is not just installed; it's ready to conspire with you, lying in wait for its moment to act.

## Usage

To wield `persistent-sshfs`, you must provide it with a list of mounts, like whispering secrets into the void. Create a file (`mounts.txt`) and inscribe it with your desires:

```
/home/local/dir:user@remote.server:port:/remote/dir
/home/local/dir2:user@remote2.server:port:/remote/dir2
```

Invoke `persistent-sshfs` with the path to your list as a sacrifice:

```sh
persistent-sshfs mounts.txt
```

And behold as it tirelessly works to maintain your mounts, defying the very laws of connectivity and physics.

## Agent integrations

The [skill](.agents/skills/persistent-sshfs) works in any agent that reads `.agents/skills/`, and
installs natively in the clients below.

### Claude Code

```bash
claude plugin marketplace add psyb0t/agents
claude plugin install persistent-sshfs@psyb0t
```

### Codex

```bash
codex plugin marketplace add psyb0t/agents
codex plugin add persistent-sshfs@psyb0t
```

Installed this way, Codex invokes the skill as `$persistent-sshfs:persistent-sshfs`. Codex also
picks the skill up automatically with no install in any repo containing `.agents/skills/`, where
it invokes as plain `$persistent-sshfs`.

### OpenClaw

The skill is published to ClawHub on every release:

```bash
openclaw skills install @psyb0t/persistent-sshfs
```

## Features

- **Resurrection**: `persistent-sshfs` will continuously check and ensure that all specified mounts are alive and well, reviving any that have fallen.
- **Rebellion Against Passwords**: It refuses to acknowledge mounts that rely on passwords. SSH keys are the only currency it accepts.
- **Anarchy in the Background**: It runs in the shadows, in the background, ensuring your mounts are persistent without bothering your conscious mind.

## Why `persistent-sshfs`?

Because life is too short for manual remounts. Because we believe in automation, in the beauty of a script that does the work for you. And because, in the heart of every sysadmin, there's a little bit of an anarchist waiting to break free.

Join us, and let's make those mounts persistent. Together, we can create a world where no mount ever goes unmounted again. A world of `persistent-sshfs`.
