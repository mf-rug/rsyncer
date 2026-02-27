# rsyncer

A minimal CLI tool to rsync a named directory from a remote server to your local machine.

Instead of typing out full remote paths, you give rsyncer a folder name and it finds it for you — useful when working with HPC clusters where job output lands in various subdirectories.

## Features

- Search for a directory by name across configured remote paths
- Interactively pick when multiple matches are found
- `--recent`: show the most recently modified directories, fast-tracked via SLURM job history
- `--filter`: select which file extensions to include before syncing
- Auto-creates the local destination directory
- Single-file Python script, no dependencies beyond the standard library

## Requirements

- Python 3.6+
- `rsync`
- SSH access to the remote server (key-based or 2FA with ControlMaster)
- *(Optional)* SLURM (`sacct`) on the server for `--recent` speed

## Installation

```bash
cp rsyncer ~/.local/bin/rsyncer
chmod +x ~/.local/bin/rsyncer
```

On first run, rsyncer will prompt you for your server and search paths, **test the
SSH connection**, and write a config file to `~/.config/rsyncer/config.json`.
If the connection test fails, the config is not saved — fix SSH first.

## Configuration

Config is stored at `~/.config/rsyncer/config.json`:

```json
{
  "server": "hpc",
  "search_paths": ["/home/user", "/scratch/user"],
  "rsync_flags": "-auz --info=progress2 -h",
  "max_depth": 5
}
```

| Key | Description | Default |
|-----|-------------|---------|
| `server` | SSH target — an SSH config alias (e.g. `hpc`) or `user@hostname` | — |
| `search_paths` | Directories to search on the server | — |
| `rsync_flags` | Flags passed to rsync | `-auz --info=progress2 -h` |
| `max_depth` | Max `find` depth when searching | `5` |

An SSH config alias is recommended because it carries IdentityFile, ControlMaster,
and other settings. See the SSH setup section below.

## SSH setup for 2FA clusters

If your server requires two-factor authentication, batch SSH calls will fail
because they cannot prompt for a code. Set up SSH ControlMaster multiplexing:

Add to `~/.ssh/config`:
```
Host hpc
    HostName login.cluster.example.com
    User your-username
    IdentityFile ~/.ssh/your-key
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 4h
```

Then:
```bash
mkdir -p ~/.ssh/sockets
ssh hpc   # log in once with your 2FA code
```

All subsequent rsyncer calls will reuse the connection for 4 hours.
Use `hpc` as the server value during rsyncer setup.

## Usage

### Sync a folder by name

```bash
rsyncer my_run_001
```

Searches for a directory named `my_run_001` on the server and rsyncs it into `./my_run_001/`.

- Leading paths and trailing slashes are stripped automatically, so `rsyncer /scratch/user/my_run_001/` works too.
- If multiple matches are found, you are prompted to pick one.
- If the local directory doesn't exist, you are asked before it is created.

### Filter by file extension

```bash
rsyncer my_run_001 --filter
```

Before syncing, lists all file extensions found in the remote directory and lets you select which to include.

### Non-interactive / scripted use

```bash
rsyncer my_run_001 --yes
```

Skips all confirmation prompts:
- Single server match: syncs without asking
- Multiple matches: auto-selects the first one and prints it
- Local directory creation: creates without asking

```bash
rsyncer my_run_001 --yes --dest /path/to/local/dir/
```

Sync into a specific local directory instead of `./<folder_name>/`.

### Browse recently active directories

```bash
rsyncer --recent
rsyncer --recent 30
rsyncer --recent --days 14
rsyncer --recent --all
```

| Flag | Description |
|------|-------------|
| `--recent [N]` | Show N most recently modified directories (default: 20) |
| `--days D` | How many days back to look in SLURM history (default: 29) |
| `--all` | Skip SLURM, search all `search_paths` directly (slow on large filesystems) |

When SLURM is available, `--recent` queries `sacct` for recent job working directories, then finds the most recently touched files within those. You can interactively exclude directories from the search before it runs.

The output is a ranked list you can use as input to the next `rsyncer <folder>` call.

## License

MIT
