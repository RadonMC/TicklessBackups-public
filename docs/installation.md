# Installation

## Requirements

- A Fabric server running Minecraft 26.1, 26.1.1, 26.1.2, 26.2 or 26.3.
- [Fabric Loader](https://fabricmc.net/use/server/) 0.17 or newer.
- [Fabric API](https://modrinth.com/mod/fabric-api) for your Minecraft version.
- Java 25 or newer.

Tickless Backups needs these parts of Fabric API: `fabric-command-api-v2`, `fabric-lifecycle-events-v1` and `fabric-networking-api-v1` (for the configuration screen); installing the full Fabric API covers all of them.

The jar is for both sides. A dedicated server needs it only on the server; to use the [configuration screen](configuration.md#the-configuration-screen) a player also needs the mod (and Fabric API) in their own game.

## Which jar do I download?

| Your Minecraft version | Jar to use |
|---|---|
| 26.1, 26.1.1 or 26.1.2 | the jar for 26.1 (`tickless-backups-1.0.0+26.1.jar`) |
| 26.2 | `tickless-backups-1.0.0+26.2.jar` |
| 26.3 | `tickless-backups-1.0.0+26.3.jar` |

Do not use a jar built for a different Minecraft version. Do not use the `-sources.jar` files: they contain source code, not the mod.

## Steps

1. Stop the server.
2. Copy the Tickless Backups jar and the Fabric API jar into the server's `mods` folder.
3. Start the server.
4. Check the console. You should see lines like:

   ```
   Initializing Tickless Backups v1.0.0 for Minecraft 26.1
   Backups ready: /path/to/your/server/backups
   ```

5. The first start creates `config/tickless-backups.json` with the default settings. See [Configuration](configuration.md) to change them.

## Who can run the commands?

All `/backup` commands need the highest permission level (owner, level 4). The server console always has it. To give a player access, make them an operator with level 4 in `ops.json`, for example:

```json
[
  { "uuid": "00000000-0000-0000-0000-000000000000", "name": "YourName", "level": 4 }
]
```

Restart the server (or reload the operator list) after editing `ops.json`.

## Where are my backups?

By default in a `backups` folder inside your server folder, with one subfolder per world (named after the world folder):

```
server/
├── world/
├── backups/
│   └── world/
│       ├── manifests/
│       │   ├── 2026-10-03_12-00-00.json
│       │   └── 2026-10-03_12-00-00_before-update.json
│       ├── objects/          the content of the files, shared by all the backups
│       ├── recipes/          how region files stored in pieces are rebuilt
│       └── exports/          zips made with /backup export
└── config/tickless-backups.json
```

A backup is the small file in `manifests`; the files it lists are in `objects` (and `recipes`), shared with the other backups. **Copy or move the whole `backups/<world>` folder**, not single files: a manifest without the `objects` folder is useless. To get a single file you can open with any tool, use [`/backup export`](commands.md#backup-export-name).

You can move this folder with the `backupDir` option. It must not be inside the world folder.

In singleplayer every world gets its own subfolder of `.minecraft/backups/` (for example `backups/New World/`), so the backups, `/backup list`, `/backup restore` and retention of one world never mix with another's.

## Updating

1. Stop the server.
2. Replace the old Tickless Backups jar with the new one.
3. Start the server.

Your config file and your existing backups are kept. New options that a newer version adds will use their default values until you add them to the file.

## Uninstalling

1. Stop the server.
2. Remove the Tickless Backups jar from `mods`.
3. Optionally delete `config/tickless-backups.json`.

Your backups in the `backups` folder are not touched, and they stay readable by hand: use `/backup export` before you remove the mod to turn the ones you want to keep into plain `.zip` files. Nothing is left behind in the world.

If you remove the mod while a restore is still scheduled, the restore will not happen (it is applied by the mod). You can delete the `config/tickless-backups/` folder to clean up.

## Using it with a hosting provider

Many hosting panels let you upload mods and edit files in `config/`. You need console access to run `/backup` commands (or be able to run them in game as a level 4 operator). If your host runs the server inside a container, see [Troubleshooting](troubleshooting.md) for the note about reflink.
