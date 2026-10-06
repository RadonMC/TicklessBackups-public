# Tickless Backups

Reliable, asynchronous world backups for Fabric servers.

Tickless Backups takes a consistent snapshot of your world while the server keeps running, then stores and verifies it on background threads. It copies and stores only what changed since the previous backup, so a backup of a world of tens of gigabytes takes a fraction of a second of paused autosave and adds only the changed files. The server's main thread never waits for the copy, the hashing or the verification, so your TPS stays steady while a backup runs.

## Features

- **Asynchronous and "tickless"**: copying, hashing, storing and verifying never run on the server thread.
- **Incremental backups**: only the files that changed since the previous backup are copied and stored; a file that did not change exists once in the store, however many backups contain it. A region file that changes is stored in pieces cut along its chunks, so one changed chunk adds only a few KB, not the whole region. The time autosave stays off and the space used depend on how much changed, not on the size of the world.
- **Configuration screen**: start backups and change the schedule, the retention and the other options from a screen in the game (key, pause menu or `/backup gui`). It edits the same `config/tickless-backups.json`, also on a dedicated server. See [Configuration](docs/configuration.md#the-configuration-screen).
- **Automatic backups**: every so many minutes and/or at fixed times of the day, skipped while nobody is online. Off by default; see [Configuration](docs/configuration.md#schedule).
- **Progress bar**: while a backup, export or check runs, operators see a bar in the action bar, such as `Saving [||||||||||||                  ] 42%`, with a colour for each phase.
- **Atomic snapshots**: autosave is paused only while the snapshot is taken, then resumed immediately. On filesystems that support reflink (Btrfs, XFS with reflink, APFS, recent OpenZFS) the snapshot takes milliseconds and almost no extra space; everywhere else only the changed files are copied.
- **Verified storage**: every file is named after its SHA-256, written atomically and read back before it is trusted. `/backup verify` checks everything on demand, and `/backup export` writes any backup as a standalone `.zip`.
- **Retention**: automatically delete old backups by count, age and/or total size. The newest backup is never deleted.
- **Automatic verification**: `/backup verify` on a schedule (every N hours, all backups or only the newest). The result is in `/backup status`, the log and the notifications.
- **Off-site upload**: copy every backup to an S3-compatible storage (AWS S3, MinIO, Backblaze B2, Cloudflare R2, ...) or a WebDAV server, written on the Java HTTP client with no SDK. Only what the remote lacks is sent, the manifest last, on a thread of its own with retries; optional and conservative remote retention (off by default); `/backup fetch` brings a backup back, verified by hash. Credentials never reach the game screen.
- **zstd compression**: an option next to deflate, written by a pure-Java library bundled in the jar (no native code); existing stores keep working.
- **Hooks**: run your own command (an argument list, no shell, with a timeout) before a backup, while autosave is paused, or after it. A hook never leaves autosave off and never breaks a backup unless you configure it to.
- **Notifications**: a webhook (Discord or generic JSON) is told when a backup fails, a check finds problems or the disk is almost full, and optionally when a backup completes. Sent in the background; the address is a secret and is never logged or shown in the game.
- **Backup diff**: `/backup diff <a> [b]` shows the files added, removed and changed between two backups, with sizes, and for region files how many bytes really differ.
- **One-command restore**: `/backup restore <name>` verifies the backup and applies it when the server stops. Your current world is kept next to it, never deleted.
- **Partial restore**: `/backup restore <name> only <path>…` brings back only a dimension, a folder or specific files (such as one player's data), keeping every replaced file in a safety folder.
- **Safe by design**: autosave is never left off, and an administrator's own `/save-off` is never undone.

## Requirements

| Requirement | Version |
|---|---|
| Minecraft (Java Edition) | 26.1, 26.1.1, 26.1.2, 26.2 or 26.3 |
| [Fabric Loader](https://fabricmc.net/use/server/) | 0.17 or newer |
| [Fabric API](https://modrinth.com/mod/fabric-api) | required |
| Java | 25 or newer |

Tickless Backups works on dedicated servers and in singleplayer (the integrated server). It has no other dependencies to install: the small library it uses for zstd is bundled inside the jar.

## Quick start

1. Put the Tickless Backups jar **and Fabric API** in your server's `mods` folder.
2. Start the server. A default config is created at `config/tickless-backups.json`.
3. As a server owner (permission level 4) or from the console, run:

   ```
   /backup start
   ```

4. When it finishes you will see a message like:

   ```
   Backup '2026-10-03_12-00-00' completed: 412 files (21 changed), 38.4 MiB of world, 2.1 MiB added, in 0.4 s
   (saving was paused for 85 ms, snapshot by reflink)
   ```

   Your backup is in `backups/<world folder>/` next to your world (each world has its own subfolder).

## Commands at a glance

| Command | What it does |
|---|---|
| `/backup start [name]` | Start a backup in the background, with an optional label |
| `/backup full [name]` | Same, but read every file again instead of trusting size and time |
| `/backup status` | Show the running backup, the last result and any pending restore |
| `/backup list` | List the most recent backups |
| `/backup verify [name]` | Check the stored files of one backup, or of all |
| `/backup diff <a> [b]` | Show what changed between two backups (`b` defaults to the newest) |
| `/backup export <name>` | Write a backup as a standalone `.zip` |
| `/backup upload status` | Show how the upload to the remote is going |
| `/backup upload now` | Upload to the remote now |
| `/backup fetch <name>` | Download a backup from the remote, verified |
| `/backup notify test` | Send a test message to the webhook |
| `/backup gui` | Open the configuration screen (needs the mod on your game) |
| `/backup reload` | Reload `config/tickless-backups.json` |
| `/backup restore <name>` | Schedule a restore, applied when the server stops |
| `/backup restore <name> only <path>…` | Schedule the restore of some files or folders only |
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

- Off-site upload over SFTP (S3 and WebDAV are done; SFTP needs a host key check and a heavier library, so it was left out)

## License

All Rights Reserved.
