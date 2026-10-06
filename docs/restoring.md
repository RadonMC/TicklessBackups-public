# Restoring a backup

Tickless Backups never replaces your world while the server is running. A restore is a two-step process: you **schedule** it, and the world is **replaced when the server stops**. Your current world is never deleted.

## What a backup contains (and what it does not)

A backup contains the **world folder**: all dimensions, chunks, player data, statistics, advancements, maps and the other files Minecraft keeps in the world. Mods that store their data inside the world folder are included too.

A backup does **not** contain anything outside the world folder: `server.properties`, `ops.json`, `whitelist.json`, banned lists, the `mods` folder, mod configs and so on. Back those up separately if you need them.

## Step by step

1. List your backups and pick one:

   ```
   /backup list
   ```

2. Schedule the restore. Press **Tab** to complete the name:

   ```
   /backup restore 2026-10-03_12-00-00
   ```

   The mod checks the whole backup first (every file against its checksum). If it is damaged you are told right away and nothing is scheduled. Otherwise you see:

   ```
   Restore of '2026-10-03_12-00-00' is scheduled.
   It will be applied when the server stops (use /stop). Everything done in the world after that backup will be gone from the active world.
   The current world is not deleted: it is kept next to it as 'world.pre-restore-<date>'. Use /backup restore cancel to change your mind.
   ```

3. Stop the server with `/stop`.

4. While the server shuts down, the mod applies the restore. In the console log you will see:

   ```
   Applying the pending restore of '2026-10-03_12-00-00' to /path/to/server/world ...
   Restore of '2026-10-03_12-00-00' completed: 412 files. Previous world kept in: /path/to/server/world.pre-restore-20261003-130500
   ```

5. Start the server again. It loads the restored world.

## Changed your mind?

Before you stop the server:

```
/backup restore cancel
```

`/backup status` always tells you if a restore is scheduled.

## What happens to the old world

It is renamed, not deleted. It stays next to the world folder, with the date and time (UTC) of the restore:

```
server/
├── world/                                  ← the restored world
└── world.pre-restore-20261003-130500/      ← the world as it was before the restore
```

If a folder with that name already exists, a number is added (`…-2`).

**These folders are never removed automatically.** They take disk space, so delete them yourself when you are sure you do not need them.

### Undoing a restore

If you restored the wrong backup:

1. Stop the server.
2. Rename `world` to something else, for example `world.wrong`.
3. Rename `world.pre-restore-<date>` back to `world`.
4. Start the server.

## How it keeps your world safe

1. The files of the backup are rebuilt into a temporary folder (`.world.restoring`) and every file is checked against the checksum stored in the backup. No file can be larger than the backup declares, so a damaged or tampered store cannot fill the disk. The files get back the modification time they had, so the next backup does not copy them again. **Your world is not touched during this phase.** If anything is wrong (a damaged or missing file in the store, no `level.dat`), the restore stops and your world stays exactly as it was.
2. Only then are two folder renames done: the old world gets its `.pre-restore-<date>` name, and the extracted folder becomes the world. Renames within one folder are instant.
3. If the second rename fails, the old world is put back in its place automatically. In the very unlikely case that even this fails, the error message tells you where the original world is so you can put it back by hand.

A backup is a file that may have been copied, edited or fetched from a remote, so the names in it are not trusted either. A file name that leads outside the world (`..`, an absolute path, a drive such as `C:`, a backslash) is refused, and so are the names that Windows treats in a special way: devices (`CON`, `NUL`, `COM1`, `LPT1`, also with an extension such as `nul.txt`), a name that ends with a dot or a space, the characters `<>"|?*` and a `:` (an alternate data stream). A Minecraft world has no such names; if yours does, the restore stops before touching the world, with `Invalid file name in the backup`. The same rule applies to `/backup export`.

If the restore fails for any reason, the failure is written to the server log as `The restore of '…' FAILED: …`. The request is then discarded, so it is not retried at every shutdown. Fix the problem and run `/backup restore` again.

## Things to know

- **Use a normal stop.** The restore is applied during a normal shutdown (`/stop`, or your hosting panel's Stop button if it sends the stop command). If the server crashes or the process is killed, nothing is applied.
- **A scheduled restore survives restarts.** If the server crashes with a restore scheduled, the request stays on disk (`config/tickless-backups/pending-restore/<world>.json`) and is applied at the **next normal stop of that world**, which may be much later. A request is only ever applied to the world it was made for: in singleplayer, stopping a different world leaves it alone. On start the mod logs `A restore of '…' is pending and will be applied when the server stops.` Check `/backup status`, and use `/backup restore cancel` if you no longer want it.
- **Everything after the backup is lost from the active world.** That includes player progress, builds and player data. The old world is still in the `.pre-restore-<date>` folder if you need something back.
- **Players cannot connect between `/stop` and the next start.** Plan the restore for a quiet moment and warn your players.

## Restoring only part of the world

You do not always want to go back in time for the whole world. To bring back one dimension, one folder or specific files, for example the data of a single player who lost their inventory:

```
/backup restore 2026-10-03_12-00-00 only playerdata/0b3a5c2e-1111-2222-3333-444455556666.dat
/backup restore 2026-10-03_12-00-00 only DIM-1
```

It works like the restore of the whole world: the files are checked now, the request is scheduled, and it is applied when the server stops (`/backup status` shows it, `/backup restore cancel` cancels it). The rest of the world is **not touched**.

- A path is a file or a folder inside the world, such as `playerdata/<uuid>.dat`, `region` or `data/scoreboard.dat`, or the folder of a dimension. Look inside your world folder to see how the dimensions are named by your Minecraft version (for example `DIM-1` for the Nether in many versions): the path must be exactly the one in the backup.
- **Your files are never deleted.** What is replaced is moved, with its path, into `<world>.pre-restore-<date>` (the same kind of folder a full restore makes). If you restore a folder, what the world has in that folder and the backup does not (created after it) is moved there too, so the folder comes back as it was. A file that was only in the backup is simply put back.
- The files are rebuilt in a temporary folder and checked first. If one is damaged or missing from the store, nothing is changed.
- **A link that leads out of the world is never followed.** If a folder of the world is a symbolic link (or a junction) to somewhere else, a partial restore that would write or move a file through it is refused (`Refusing to restore through a link that leads out of the world`) and nothing is changed.
- If a file cannot be moved into place, the moves already made are undone, in reverse order, and the world is as it was before.
- Undoing it: stop the server and move the files from `<world>.pre-restore-<date>` back into the world, keeping their paths.

Restoring a folder of chunks while the rest of the world is newer can leave a visible seam where old chunks meet new ones, and restoring `level.dat` alone brings back the time, the weather and the game rules of that moment. Use it when you know what you are bringing back.

## Restoring by hand (without the mod)

A backup is made of a manifest and of the shared files in `objects`, which no ordinary tool reads. To restore without the mod, first turn the backup into a normal zip, while the mod is still installed:

```
/backup export 2026-10-03_12-00-00
```

This writes `backups/<world>/exports/2026-10-03_12-00-00.zip`. Keep the zips you may need without the mod (they are not deleted by retention). Then, any time:

1. Stop the server.
2. Rename your current world folder (for example `world` → `world.old`).
3. Create a new empty folder with the original world name (the `level-name` in `server.properties`, `world` by default).
4. Extract the `.zip` into it. The world files (`level.dat`, `region/`, `data/`, …) must end up directly inside the folder, not inside an extra subfolder.
5. Optionally delete `tickless-manifest.json` from the restored folder. It is the checksum list; Minecraft ignores it.
6. Start the server.
