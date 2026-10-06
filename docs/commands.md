# Commands

All commands start with `/backup` and need the highest permission level (owner, level 4). The server console can always run them.

| Command | Summary |
|---|---|
| [`/backup start [name]`](#backup-start-name) | Start a backup in the background |
| [`/backup full [name]`](#backup-full-name) | Start a backup that reads every file again |
| [`/backup status`](#backup-status) | Show the current state and the last result |
| [`/backup list`](#backup-list) | List recent backups |
| [`/backup verify [name]`](#backup-verify-name) | Check the stored files of one backup, or of all |
| [`/backup diff <a> [b]`](#backup-diff-a-b) | Show what changed between two backups |
| [`/backup export <name>`](#backup-export-name) | Write a backup as a standalone `.zip` |
| [`/backup upload status`](#backup-upload-status) | Show how the upload to the remote is going |
| [`/backup upload now`](#backup-upload-now) | Upload to the remote now |
| [`/backup fetch <name>`](#backup-fetch-name) | Download a backup from the remote |
| [`/backup notify test`](#backup-notify-test) | Send a test message to the webhook |
| [`/backup gui`](#backup-gui) | Open the configuration screen |
| [`/backup reload`](#backup-reload) | Reload the configuration |
| [`/backup restore <name>`](#backup-restore-name) | Schedule a restore for the next server stop |
| [`/backup restore <name> only <path>…`](#backup-restore-name-only-path) | Schedule the restore of some files or folders only |
| [`/backup restore cancel`](#backup-restore-cancel) | Cancel a scheduled restore |

## `/backup start [name]`

Starts a backup in the background and returns immediately. You keep playing normally. Only what changed since the previous backup is copied and stored: see [How it works](how-it-works.md).

```
/backup start
/backup start before the big update
```

- Without a name, the backup is called after the current time in UTC: `2026-10-03_12-00-00`.
- With a name, it is added as a label: `2026-10-03_12-00-00_before_the_big_update`.
- The label is cleaned for safe use in file names: every character that is not a letter, digit, dot, hyphen or underscore becomes `_`, and it is cut to 48 characters. Spaces therefore become underscores.
- Only one backup, export or check can run at a time. If one is already running you get `A backup, export or check is already running. Use /backup status.`
- If a backup with the same name already exists (same second, same label), a number is added: `…_2`.

When the backup finishes, whoever started it gets a message (online operators also see it):

```
Backup '2026-10-03_12-00-00_before_the_big_update' completed: 3061 files (21 changed), 9.9 GiB of world,
170.0 MiB added, in 1.1 s (saving was paused for 440 ms, snapshot by copy)
```

| Part of the message | Meaning |
|---|---|
| `3061 files` | Files in the world, all of them listed in the backup |
| `(21 changed)` | Files that changed since the previous backup: only these were copied and stored |
| `9.9 GiB of world` | Size of the world at the moment of the backup |
| `170.0 MiB added` | What this backup added to the store, after compression. A backup of an unchanged world adds nothing |
| `in 1.1 s` | Total time of the backup, from start to the written manifest |
| `saving was paused for 440 ms` | How long world autosave was off. This is the only moment the world's files are held still |
| `snapshot by reflink` / `snapshot by copy` | How the snapshot was taken. See [How it works](how-it-works.md#reflink-or-copy) |
| `every file read` | Only for a full backup (see below) |

If a backup fails you get `Backup '<name>' failed: <reason> (see the server log for details)`. Autosave is always turned back on, whether the backup succeeded or not. See [Troubleshooting](troubleshooting.md).

## `/backup full [name]`

Like `/backup start`, but reads and hashes **every** file instead of trusting size and modification time. Use it in a quiet moment if you suspect that something changed without being noticed (see [How "unchanged" is decided](how-it-works.md#how-unchanged-is-decided)).

- Files already in the store are not stored again, so the space used does not grow.
- Without a reflink, autosave stays off for as long as a copy of the whole world takes (the same as the first backup of a world).
- With `fullScanEvery` in the [configuration](configuration.md#fullscanevery) you can make every N-th backup a full one.

## `/backup gui`

Opens the [configuration screen](configuration.md#the-configuration-screen) on your game. It only works for a player (not the console) whose game has the Tickless Backups mod; otherwise you are told so. The same screen opens with the **Backups** button in the pause menu, or with a key you can assign in Controls (not assigned by default).

## `/backup status`

Shows these things:

1. Whether something is running, and in which phase: `preparing`, `snapshotting`, `storing` or `pruning` for a backup (pruning = applying your retention rules), `exporting` or `verifying` for the other operations.
2. The last backup since the server started: name, how long ago, what it added and duration, in green if it succeeded or red if it failed.
3. When the next [automatic backup](configuration.md#schedule) is due, or that automatic backups are off.
4. A scheduled restore, if there is one (in yellow).
5. The last [check of the backups](configuration.md#verify) since the server started (made by `/backup verify` or by the schedule), and when the next automatic check is due.
6. Whether [notifications](configuration.md#notifications) are on.

```
Nothing is running.
Last backup: '2026-10-03_12-00-00', 12.4 min ago, 170.0 MiB added, took 1.1 s.
Next automatic backup: in 17.6 min.
```

The "last backup" information is kept in memory, so after a server restart it shows `No backup has run since the server started.` Use `/backup list` to see backups on disk.

## `/backup list`

Lists the most recent backups, newest first, with date (UTC), name, number of files, size of the world, and what each one added. It shows up to 10 and tells you how many older ones there are.

```
23 backups (the world size, and what each one added):
 2026-10-03 12:00 UTC  2026-10-03_12-00-00_before_the_big_update  3061 files, 9.9 GiB  (+170.0 MiB)
 2026-10-03 06:00 UTC  2026-10-03_06-00-00  3061 files, 9.9 GiB  (+1.2 MiB)
 ...
 ... and 13 older
```

- The date comes from inside the manifest, not from the file's date, so copying or moving backup files does not change it.
- Files in the `manifests` folder that are not valid manifests are skipped and logged as a warning. They are never listed and never deleted by retention, and while one exists nothing is deleted from the store.

## `/backup verify [name]`

Reads the stored files of one backup (or of **all** backups without a name) and checks each one against its checksum. A file that many backups share is read once. It runs in the background, one operation at a time, and does not touch the world.

```
/backup verify
/backup verify 2026-10-03_12-00-00
```

- Everything fine: `Check completed, everything is fine: 23 backups, 3082 stored files, 10.1 GiB read in 6.2 s.`
- Problems: you get how many files are damaged or missing and the first five. The full list is in the server log. A backup that needs a damaged or missing file cannot be restored; take a new `/backup full` so that the files come back into the store.

Use it now and then (for example after a disk problem, or before relying on an old backup), or after copying the backup folder to another place.

## `/backup diff <a> [b]`

Shows what changed **from** backup `a` **to** backup `b`. Without `b` the newest backup is used, so `/backup diff <name>` answers "what has changed since that backup". It only reads the two manifests (and the recipes of changed region files): it does not touch the world or the store, and it runs in the background.

```
/backup diff 2026-10-03_06-00-00
/backup diff 2026-10-03_06-00-00 2026-10-03_12-00-00
```

```
From '2026-10-03_06-00-00' to '2026-10-03_12-00-00':
 1 added (1.2 MiB), 0 removed (0 B), 21 changed (+8.1 MiB, 3.0 MiB of new content), 3039 unchanged (9.9 GiB).
  + region/r.5.5.mca (1.2 MiB)
  ~ region/r.0.0.mca: 4.1 MiB -> 4.2 MiB, 120.0 KiB differ (compared by pieces)
  ~ level.dat: 2.1 KiB -> 2.1 KiB, 2.1 KiB new (stored whole)
```

- A file is **added** if only `b` has it, **removed** if only `a` has it, and **changed** if both have it with a different content. The content decides (its checksum): a file that was only touched is not a change.
- Each list shows the 6 largest files and how many more there are.
- For a region file that both backups store in pieces, the command compares the pieces and says how many bytes of the new version are not in the old one: that is what a change really costs. For a file stored whole, all of its new size counts.
- Press **Tab** to complete the names. Comparing a backup with itself says there are no differences.
- It does not compare a backup with the world as it is now: that would mean reading the whole world while the server runs. To see what changed since the last backup, take a new backup and compare the two.

## `/backup export <name>`

Writes a backup as a normal `.zip`, which any zip tool opens and which you can restore by hand without the mod. It runs in the background.

```
/backup export 2026-10-03_12-00-00
```

- Press **Tab** to complete backup names, most recent first. Type any part of the name or of its date (`2026-10-05`, `14:45`, `update`) and Tab lists only the backups that match; the separators do not matter.
- The zip is always written in the world's `exports` folder (`backups/<world>/exports/<name>.zip`). The command takes no path, so it cannot write anywhere else.
- It is checked after it is written, and a file that is damaged in the store makes the export fail. A name inside the backup that would leave the folder when the zip is extracted (`..`, an absolute path, a drive) or that Windows treats in a special way (`NUL`, a trailing dot, ...) makes it fail too, before anything is written.
- It needs free space for the size of the world in the `exports` folder. If the zip already exists you get an error: delete or move it first.
- Exports are never deleted by retention.

## `/backup upload status`

Shows the remote (without secrets), what is going on (the backup being sent and how many bytes of it, or the one being downloaded), how the last upload run went and when the last successful one ended, and how many local backups the remote does not have yet. See [`upload`](configuration.md#upload).

```
Remote: S3 minio.example.com (uploads are on)
Nothing is being transferred.
Last upload run 3.2 min ago: 1 backups uploaded to S3 bucket "minecraft" at minio.example.com (14 files)
Last successful run: 3.2 min ago.
```

## `/backup upload now`

Starts an upload run now (it needs `upload.enabled`) and tells you how it went. It sends every local backup that the remote does not have, newest first. The run happens on its own thread: a backup that starts meanwhile is not delayed.

## `/backup fetch <name>`

Downloads a backup from the remote into the local store, so that you can list it, verify it, export it and [restore](#backup-restore-name) it, for example on a new machine. It works whenever a remote is configured, even with `upload.enabled` off. Press **Tab** to complete the names of the backups the remote had at the last look that you do not have.

- Everything that comes from the remote is checked: the manifest like any manifest, every stored file against the SHA-256 and the size it is named after (a file stored in pieces is joined and compared with its hash). Nothing damaged is ever installed, and **the manifest is written last**: a download that fails leaves no backup behind.
- Only what the local store lacks is downloaded. The command refuses a name that is already here, one that is not on the remote, a name with anything but letters, digits, dots, hyphens and underscores, a name that Windows treats in a special way (`CON`, `NUL`, a trailing dot, ...), and a manifest that is not the one of the name asked for.
- The remote is not trusted: a download stops at the size the file must have (a manifest at most 64 MiB), and if a cleanup of this machine deletes a file that was just downloaded (a download longer than the hour that protects unused files), it is downloaded again before the backup is declared complete.
- A fetched backup is a normal backup in your folder: **nothing is deleted when the server stops**, and it is still there after a restart. The retention leaves it alone only while the server keeps running; after a restart it counts like any other backup, so if it is older than your limits (`retention.maxAgeDays`, `retention.maxCount`, `retention.maxTotalBytes`) the **next backup** deletes it. Restore it or export it before that, or raise the limits.
- It does not restore anything: use `/backup restore <name>` afterwards.

## `/backup notify test`

Sends a test message to the [webhook](configuration.md#notifications) and says whether it was accepted. It fails at once if `notifications.url` is empty. The reason of a failure is in the server log (never the address). `/backup status` says whether notifications are on.

## `/backup reload`

Re-reads `config/tickless-backups.json`. New settings apply from the **next** backup; a backup already running is not affected.

If the file had a syntax error when the server started, the mod could not start. After you fix the file, `/backup reload` starts it without restarting the server. If the file is still invalid, you get `Could not load the configuration: …` and your file is **not** overwritten.

See [Configuration](configuration.md).

## `/backup restore <name>`

Schedules a restore of the named backup. **It does not change the world right away.** It checks the backup, then the world is replaced when you stop the server.

```
/backup restore 2026-10-03_12-00-00
```

- Press **Tab** to complete backup names, most recent first. Type any part of the name or of its date (`2026-10-05`, `14:45`, `update`) and Tab lists only the backups that match; the separators do not matter.
- Every file of the backup is checked first against its checksum. A damaged or incomplete backup is refused immediately: `Backup '…' is damaged and cannot be restored: …`.
- If there is no such backup: `There is no backup named '…'. Use /backup list.`
- Then stop the server with `/stop`. The world is replaced during shutdown. Your current world is kept as `<world>.pre-restore-<date>`.
- To bring back only a part of the world, see `/backup restore <name> only <path>…` below.

Running `/backup restore` **without a name does nothing**: it shows how to use the command and your three most recent backups. It can never replace the world by accident.

Read [Restoring a backup](restoring.md) before your first restore.

## `/backup restore <name> only <path>…`

Schedules the restore of **some files or folders** of a backup, leaving the rest of the world as it is. Like the restore of the whole world, it does not change anything right away: the backup is checked now, and the files are put back when the server stops.

```
/backup restore 2026-10-03_12-00-00 only DIM-1
/backup restore 2026-10-03_12-00-00 only playerdata/0b3a5c2e-1111-2222-3333-444455556666.dat
/backup restore 2026-10-03_12-00-00 only region level.dat
```

- A path is relative to the world folder and written with `/`. It can be a **file** or a **folder**. Give up to 20 paths, separated by spaces (a path that contains a space cannot be given to the command).
- A path that is not in the backup is refused (`The backup has no file or folder '…'`), so a typo never "restores nothing" silently. Paths that are empty, absolute or contain `..` are refused.
- Only the files that will be restored are checked against their checksums. Region files that were stored in pieces are rebuilt and checked like any other.
- **Nothing is deleted.** Each file that is replaced is moved, with its path, into `<world>.pre-restore-<date>` next to the world. If you restore a **folder**, the files that are in the world's folder but were not in the backup (created after it) are moved there too: the folder comes back as it was. If a file is restored that the world no longer has, there is nothing to keep.
- If anything fails while the files are put in place, the moves already made are undone and the world is as it was.

See [Restoring only part of the world](restoring.md#restoring-only-part-of-the-world).

## `/backup restore cancel`

Cancels a scheduled restore. Use it if you change your mind before stopping the server. If nothing was scheduled you get `No restore is pending.`
