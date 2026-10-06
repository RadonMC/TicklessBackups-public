# Troubleshooting and FAQ

## Messages and what to do

These are the messages you may see in chat or in the server log.

### The mod does not start

| Message | Cause | Fix |
|---|---|---|
| `Tickless Backups is not running: Invalid JSON in tickless-backups.json: …` | The config file has a syntax error (a missing comma or quote, for example). | Fix `config/tickless-backups.json`, then run `/backup reload`. The file is never overwritten. |
| `Tickless Backups is not running.` | The mod did not start for another reason. | Read the server log from the start for a `Tickless Backups could not start` line. |
| `Could not load the configuration: …` | `/backup reload` found an invalid file. | Fix the file and run `/backup reload` again. |
| The server will not start at all | Missing Fabric API, wrong Minecraft version or old Java. | See [Installation](installation.md). The error in the log names the missing dependency. Java 25 or newer is required. |

### A backup fails

A failed backup is reported as `Backup '<name>' failed: <reason> (see the server log for details)`. **Autosave is always turned back on**, so a failed backup never leaves your world unsaved.

| Reason | Cause | Fix |
|---|---|---|
| `The backup folder cannot be inside the world folder` | `backupDir` points inside the world folder. | Change `backupDir` in the config (see [Configuration](configuration.md#backupdir)). |
| `The backup folder cannot be inside …/.tickless-staging` | `backupDir` points inside the temporary snapshot folder, which is wiped at every backup. | Change `backupDir` in the config. |
| `Not enough free space for the snapshot: about N bytes needed, M available` | Not enough room on the disk that holds the world (only checked for a normal copy, not for a reflink). | Free some space. |
| `Not enough free space for the changed files: …` | Not enough room in `backupDir` for the files that changed. | Free space, move `backupDir` to a bigger disk, or tighten `retention`. |
| `Stored object is damaged: …` / `File changed while it was stored: …` | A file was not written correctly to the store, or changed after the snapshot. Nothing damaged is kept. | Run the backup again. If it keeps happening, check the disk of `backupDir`. |
| `TimeoutException` (or a timeout message) while pausing | Minecraft did not finish writing pending data within `drainTimeoutSeconds`, or the server was lagging so much that the pause could not even start. Autosave is not left off in either case. | Raise `drainTimeoutSeconds`, or check if the disk or the server is overloaded. |
| `Files changed during the snapshot (is autosave really paused?): …` | Something kept writing to world files during the copy, for example a `/save-on` or `/save-all` by an operator, or another plugin or mod. | Do not use `/save-on` or `/save-all` during a backup; run it again. |
| `level.dat is missing from the snapshot (it was being replaced during the copy)` | Minecraft was rewriting `level.dat` at that moment. The mod already retries 3 times. | Run the backup again. If it keeps happening, tell us. |
| `The server is shutting down` | A backup was requested while the server was stopping. | Normal. Run it again after the next start. |
| `interrupted` | The server stopped while the backup was running. | Normal. Run it again after the next start. |

If a backup fails and the message is not clear, open the server log: the mod logs the full error with `Backup '<name>' failed`.

### Warnings in the log

| Message | Meaning |
|---|---|
| `Snapshot attempt 1/3 failed (…), retrying` | A file changed during the copy. The snapshot is retried automatically. Occasional ones are harmless. |
| `Retention failed (backup '…' is still valid)` | Old backups could not be deleted (for example a file is locked). The new backup is fine. |
| `Backup ignored because its manifest is unreadable: …` | A `.json` file in the `manifests` folder is not a valid manifest. It is skipped and never deleted, and while it exists no stored file is deleted. | Inspect it; delete or move it if it is not a backup. |
| `The previous backup cannot be read (…): every file will be read` | The manifest of the previous backup is damaged. The new backup reads every file, so it is slower, but complete. | Nothing to do. |
| `N files of the previous backup are missing from the store: they will be read again` | Some stored files were deleted or lost. The new backup stores them again. | Run `/backup verify` to see if other backups are affected. |
| `Cleanup of the store skipped, nothing was deleted: the manifest … cannot be read (…)` | Retention could not tell which stored files are still used, so it deleted none. The backups are fine. | Fix or remove the manifest named in the message. |
| `Reflink clone timed out after 10 minutes, using the normal copy` | The reflink took too long, so the normal copy is used instead. The backup still succeeds. (If the filesystem simply does not support reflink, the mod falls back to the normal copy silently.) |
| `Reflink failed 3 times in a row on '…': it will not be tried there again until the server restarts` | The filesystem of the world does not seem to support reflink. Backups use the normal copy, without the extra scan a reflink needs. If you fixed the filesystem, restart the server; if you know it never supports reflink, you can set `allowReflink` to `false`. |
| `Field SomeClass.someField not found: draining the write queue of that storage is disabled` | A future Minecraft version changed an internal detail. Backups still run, but with less protection for that part. Please report it with your Minecraft version. |

### Restore problems

| Message | Cause | Fix |
|---|---|---|
| `There is no backup named '…'. Use /backup list.` | The name is wrong. | Use Tab to complete names. |
| `Backup '…' is damaged and cannot be restored: …` | A file of the backup is damaged or missing in the store. | Run `/backup verify` to see what is wrong, and use another backup. |
| `Missing backup name. Nothing was restored.` | You ran `/backup restore` without a name. | Add a name, or use Tab. |
| `The restore of '…' FAILED: …` (in the log) | The restore could not be applied during shutdown. | Your world is untouched, or already restored back automatically. Fix the cause named in the message and run `/backup restore` again. |
| `Restore failed and the rollback failed too: the original world is in … and must be put back by hand in …` | Very unlikely: both folder renames failed (a file in use, a permissions problem). | Do exactly what the message says: put the named folder back to the world's name. |
| `Backup '…' was deleted in the meantime (retention?).` | The backup was deleted while it was being checked. | Choose another backup with `/backup list`. |
| `Restore not scheduled: Not enough free space for the restore: …` | There is not enough room next to the world to extract the backup (the current world is kept, so the backup needs its full size). | Free space and run `/backup restore` again. |
| `Not enough free space for the restore: …` (in the log, at shutdown) | The space was enough when the restore was scheduled, but not any more. | Free space and run `/backup restore` again. |
| `File missing from the store: …` / `Checksum mismatch: …` / `File larger than declared in the manifest: …` | A stored file is missing, damaged or was modified by hand (found while restoring at shutdown). The world is untouched. | Use another backup, and run `/backup verify`. |
| `Ignoring the pending restore: …` | The restore request file could not be read. | The file is renamed to `<world>.json.invalid` in `config/tickless-backups/pending-restore/`. Inspect or delete it, then schedule the restore again. |
| `Ignoring the pending restore of '…': it was requested for …, not for …` | The request was made for another world with the same folder name (for example the game folder was moved). It is never applied to the wrong world. | The file is renamed to `.invalid`. Schedule the restore again from the right world. |

See [Restoring a backup](restoring.md) for the full picture.

### Export and check problems

| Message | Cause | Fix |
|---|---|---|
| `A backup, export or check is already running. Use /backup status.` | Only one of them runs at a time. | Wait for it to finish (`/backup status`). |
| `Export failed: There is no backup named '…'. Use /backup list.` | The name is wrong. | Use Tab to complete names. |
| `Export failed: …exports/<name>.zip` (file already exists) | The zip was already exported. | Delete or move the old zip, then export again. |
| `Export failed: Stored file is damaged: …` | A file of that backup is damaged in the store. Nothing is left behind. | Run `/backup verify` and use another backup. |
| `Check found N damaged or missing files` | `/backup verify` found files that do not match their checksum. The first five are shown, the full list is in the server log. | Backups that need those files cannot be restored. Take a new `/backup full` so that the files return to the store, then delete the broken backups. Check your disk. |

## FAQ

### Will a backup lag my server?

The copy, hashing, storing and verification run on background threads at low priority, so the server thread does not wait for them. The only work on the server thread is tiny, plus the optional save before the backup (`saveBeforeBackup`), which is about the size of an autosave. Running a backup on a very small or heavily loaded machine will still use CPU and disk, since those are shared with the server.

### Is it safe to run a backup while players are online?

Yes, that is what it is designed for.

### Can I schedule automatic backups?

Not yet: the mod has no built-in scheduler. Today you can run `/backup start` from anything that can send commands to your server console: your hosting panel's scheduler, or a cron job / systemd timer using RCON or a console wrapper. Retention then cleans up old backups automatically.

### Does it back up my mods, configs and `server.properties`?

No. Only the world folder is backed up. See [Restoring a backup](restoring.md#what-a-backup-contains-and-what-it-does-not).

### Can I back up only one dimension?

No. A backup is always the whole world folder.

### My server runs in Docker or on a hosting panel. Is reflink available?

It depends on the filesystem under the world folder, not on the mod. The layered filesystem inside a container often does not support reflink, and then the mod uses the normal copy automatically (backups still work, but autosave stays paused for the whole copy). If your world is on a volume or bind mount stored on Btrfs or XFS with reflink, it can use the fast path. The backup message shows which method was used.

### How big are the backups?

The **first** backup of a world takes about the size of the world: region files are already compressed by Minecraft, so expect modest savings. After that, every backup adds only the files that changed since the previous one, and a file shared by several backups is stored once. `/backup list` shows what each backup added. Backups do not take "N times the world": a region that changed is stored in pieces, so a backup adds roughly the chunks that changed (the first time a region changes it is stored again in full, in pieces; after that only the pieces that differ). Use `retention` to cap the total.

### Can I copy backups to another computer or to the cloud?

The mod can do it for you: see [`upload`](configuration.md#upload) (an S3-compatible storage or a WebDAV server). By hand it is also possible, but copy the **whole** `backups/<world>` folder (or sync it as a folder): a backup is a small manifest plus the shared files in `objects`, and a manifest without them is useless. Files are only ever added to `objects` or deleted when no backup uses them, so a sync tool that copies new and removed files works well. Make it ignore the `tmp` folder, which holds files being written. For a single standalone file, use `/backup export`. Built-in off-site upload is planned but not available yet.

### Can I open a backup with a normal zip program?

Not directly: a backup is a manifest and shared files. Use `/backup export <name>` to write it as a standard `.zip`, which any zip program opens.

### Can I delete a single backup by hand?

You can delete its file in `manifests`, but the files it alone used stay in `objects` until the next time retention cleans the store (after a backup that deletes something). Do not delete files in `objects` by hand: other backups may need them (`/backup verify` will tell if you did).

### The upload to the remote fails

`/backup upload status` shows the last failure, and the server log has it too (never with the address or a secret). Common causes:

- `HTTP 403 (SignatureDoesNotMatch)` or `(InvalidAccessKeyId)`: a wrong key pair, or a wrong `region`. Wrong credentials are not repeated.
- `HTTP 404 (NoSuchBucket)`: the bucket does not exist or `bucket` is misspelt.
- `The remote could not be reached`: the address, the port or the network. With MinIO on another machine you need `https`, or `allowInsecureHttp` set to `true`.
- `the server does not support MOVE` (WebDAV): the server cannot be used, because a file must appear only when it is complete.
- `HTTP 401` (WebDAV): wrong `username` or `password`.
- A backup that "is not uploaded because a file of it is missing": retention deleted something it needed. The next backup is uploaded normally.

After a failure the next backup (or `/backup upload now`) sends what is missing; nothing has to be repaired by hand.

### A hook does not run, or the backup says a hook failed

The log has `Hook '<name>' could not be started`, `failed with exit code N` or `did not finish in N s and was stopped`, and what the command printed (`[hook <name>] …`). The command is a list of arguments, not a shell line: use the full path of the program, and `["/bin/sh", "-c", "…"]` if you need a shell. With `"onFailure": "continue"` (the default) the backup goes on anyway.

### I get no notifications

Use `/backup notify test`. If it fails, the log says `HTTP <status>` or the type of the error (never the address): a 404 usually means the webhook was deleted or the address is wrong. Check that `notifications.url` is set and that the event you expect is on (`backupCompleted` is off by default).

### Where do I report a bug?

Open an issue in this repository. Please include your Minecraft version, the mod version, and the relevant part of the server log (look for lines that start with `Backup`, `Snapshot`, `Stored`, `Retention` or `Restore`).

### `Invalid file name in the backup`

The restore (or the export) found a name that cannot be created safely on every system: it leads outside the folder, or it is a Windows device (`NUL`, `CON`, `COM1`, ...), ends with a dot or a space, or has one of `<>"|?*:`. Your world was not touched. A Minecraft world has no such names, so the backup was probably changed or comes from a remote you do not trust. See [Restoring a backup](restoring.md#how-it-keeps-your-world-safe).

### `The configuration file was changed while it was being saved`

You (or a program) edited `tickless-backups.json` at the moment the configuration screen saved. The screen wrote nothing, so your edit is safe. Open the screen again and repeat the change.

### The screen does nothing when I press Save twice quickly

A player can ask the server for the screen once every half second, and to save or start a backup once a second; faster requests are dropped, so that a client that repeats them cannot keep the server busy.

### `Refusing to restore through a link that leads out of the world`

A folder of the world is a symbolic link or a junction to another place. A partial restore does not write through it. Restore the whole world, or replace the link with a real folder.
