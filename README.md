# Tickless Backups

Reliable, asynchronous world backups for Fabric servers.

Tickless Backups takes a consistent snapshot of your world while the server keeps running, then compresses and verifies it on background threads. The server's main thread never waits for the copy, the compression or the verification, so your TPS stays steady while a backup runs.

## Features

- **Asynchronous and "tickless"**: copying, zipping and verifying never run on the server thread.
- **Atomic snapshots**: autosave is paused only while the snapshot is taken, then resumed immediately. Compression works on the copy.
- **Copy-on-write snapshots**: on filesystems that support reflink (Btrfs, XFS with reflink, APFS, recent OpenZFS) the snapshot takes milliseconds and almost no extra space. Everywhere else it falls back to a safe file-by-file copy.
- **Verified archives**: every backup is a `.zip` with a SHA-256 manifest. It is re-read and checked before it becomes a final file.
- **Retention**: automatically delete old backups by count, age and/or total size. The newest backup is never deleted.
- **One-command restore**: `/backup restore <name>` verifies the backup and applies it when the server stops. Your current world is kept next to it, never deleted.
- **Safe by design**: autosave is never left off, and an administrator's own `/save-off` is never undone.

## Requirements

| Requirement | Version |
|---|---|
| Minecraft (Java Edition) | 26.1, 26.1.1, 26.1.2, 26.2 or 26.3 |
| [Fabric Loader](https://fabricmc.net/use/server/) | 0.17 or newer |
| [Fabric API](https://modrinth.com/mod/fabric-api) | required |
| Java | 25 or newer |

Tickless Backups is designed for dedicated servers. It has no other dependencies.

## Quick start

1. Put the Tickless Backups jar **and Fabric API** in your server's `mods` folder.
2. Start the server. A default config is created at `config/tickless-backups.json`.
3. As a server owner (permission level 4) or from the console, run:

   ```
   /backup start
   ```

4. When it finishes you will see a message like:

   ```
   Backup '2026-10-03_12-00-00' completed: 412 files, 38.4 MiB in 2.1 s
   (saving was paused for 85 ms, snapshot by reflink)
   ```

   Your backup is in the `backups` folder next to your world.

## Commands at a glance

| Command | What it does |
|---|---|
| `/backup start [name]` | Start a backup in the background, with an optional label |
| `/backup status` | Show the running backup, the last result and any pending restore |
| `/backup list` | List the most recent backups |
| `/backup reload` | Reload `config/tickless-backups.json` |
| `/backup restore <name>` | Schedule a restore, applied when the server stops |
| `/backup restore cancel` | Cancel a scheduled restore |

See [Commands](docs/commands.md) for details.

## Documentation

- [Installation](docs/installation.md)
- [Commands](docs/commands.md)
- [Configuration](docs/configuration.md)
- [Restoring a backup](docs/restoring.md)
- [How it works](docs/how-it-works.md)
- [Troubleshooting and FAQ](docs/troubleshooting.md)

## Compatibility notes

- One jar covers Minecraft 26.1, 26.1.1 and 26.1.2. Minecraft 26.2 and 26.3 each have their own jar. Download the one that matches your server version.
- The mod is developed and tested mainly on 26.1. Builds for the other versions compile against the matching Minecraft version; if you hit a problem on any version, please open an issue.
- Loader: **Fabric**. Quilt is not tested.

## Planned

- zstd compression
- Off-site upload (SFTP, S3, WebDAV)

## License

All Rights Reserved.
