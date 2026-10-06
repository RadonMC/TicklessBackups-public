# How it works

This page explains what happens during a backup, so you know what to expect from your server and your disk.

## The idea

A normal backup tool copies your world while the server is writing to it, so the copy can be inconsistent, and copying a big world can freeze the server. It also copies and stores **everything** every time, although most of a world does not change between two backups. Tickless Backups avoids all three:

1. It briefly freezes the world's files and takes a **snapshot** of **what changed** since the previous backup.
2. It lets the server go back to normal immediately.
3. It hashes, stores and checks the changed files in the background, off the server thread.

So the time autosave stays off, the work done and the space used depend on **how much changed**, not on how big the world is.

The server's main thread (the one that runs ticks) only does tiny, instant jobs: switching autosave off and on, handing pending writes to Minecraft's I/O threads, and printing messages. It never waits for the disk.

## What happens during a backup

```
1. (optional) normal world save           saveBeforeBackup
2. pause autosave, wait for pending writes to reach the disk
3. snapshot: copy only the changed files  reflink or copy
4. autosave back ON immediately           ← the world is free again
5. hash + store the changed files         background, on the snapshot
6. write the manifest                     the backup now exists
7. retention (delete old backups and the files nobody uses any more)
```

| Phase | Where it runs | Duration |
|---|---|---|
| 1. Save before backup | server thread | a short spike, like an autosave |
| 2. Pause and wait for writes | background thread | usually well under a second |
| 3. Snapshot | background threads | milliseconds with reflink; with a normal copy, as long as the **changed** files take to copy |
| 4. Resume | instant | — |
| 5. Hash and store | background threads, low priority | proportional to what changed, but the world is already free |
| 6. Manifest | background thread | instant |
| 7. Retention | background thread | usually well under a second |

The message at the end of every backup tells you how long autosave was paused (`saving was paused for …`). That is the only window in which the world's files are held still. Players normally do not notice it: the game keeps running; it just does not write chunks to disk for that moment.

### The progress bar

While a backup, an export, a check, an upload or a download runs, the players who can use `/backup` see a bar in the action bar (the line above the hotbar), refreshed ten times a second: `Saving [||||||||||||                  ] 42%`, 30 cells long. Its colour tells the phase: yellow preparing, orange snapshot, cyan saving, violet cleanup, blue export, teal check, pink upload, amber download; inside the bar the colour gets brighter towards the end, and the percentage is white. For a backup the percentage covers the whole operation (preparing 0-5 %, snapshot 5-20 %, saving 20-95 % by bytes, cleanup up to 100 %); the pause and the snapshot are short and show the start of their band. When it ends the bar turns green ("Done") for a moment, or red if the backup failed. An upload runs beside the backups, so while both work you see the bar of the backup followed by `Upload 63%`; an upload or a download with nothing else running has its own bar (by the bytes of the backup being sent), and a failed upload is told in red.

The bar is meant to look fluid without costing anything: the number glides towards the real one a part of the way at each refresh and never goes back, so a big file does not make it jump; the edge of the bar is a cell that is only partly lit, so it moves by less than a cell; and a message is sent only when what you would see has changed (and anyway every two seconds, because the action bar fades). It only reads counters on the server thread, with no disk or network access.

### Examples

These numbers come from a synthetic world on an SSD, **without reflink** (a normal copy). They show the shape, not a promise: your disk decides the absolute values.

| World | Backup | Autosave paused | Total time | Added to the store |
|---|---|---|---|---|
| 1.5 GB (30 regions of 50 MB) | the first | 2.9 s | 17 s | 1.5 GB |
| | nothing changed | 0.2 s | 0.4 s | 0 |
| | one region and 20 player files changed | 0.3 s | 0.7 s | 50 MB |
| 10 GB (60 regions of 170 MB) | the first | 25 s | 60 s | 10 GB |
| | nothing changed | 0.2 s | 0.4 s | 0 |
| | one region and 20 player files changed | 0.4 s | 1.1 s | 170 MB |

The **first** backup of a world has to copy everything, so without a reflink autosave stays off for as long as a full copy takes (the same as a plain copy of the world). Every later backup is fast. A backup that reads every file again (`/backup full`, see [Full backups](#full-backups)) costs the same as the first one in pause time.

A region file is stored whole the first time. When it changes, it is stored again **in pieces** cut along its chunks, and from then on a change to one chunk adds only the pieces around that chunk, not the whole 170 MB region (see [Region files](#region-files)).

## Where the backups are

Each world has its own folder, named after the world's folder, inside `backupDir`:

```
backups/world/
├── manifests/         one small file per backup: the list of its files and their checksums
│   ├── 2026-10-03_12-00-00.json
│   └── 2026-10-03_13-00-00_before_the_big_update.json
├── objects/           the content of the files, once for each distinct content
│   ├── 0a/0a3f…       a file, or a piece of a region file, stored as it is
│   ├── 0b/0b91….z     a file, stored deflated
│   └── 0d/0d52….zst   a file, stored with zstd (see compressionMethod; all forms are read)
├── recipes/           how a region stored in pieces is put back together
│   └── 0c/0c47….r
├── exports/           standalone .zip files made with /backup export
└── tmp/               files being written (removed automatically)
```

- A **backup** is its manifest. It lists every file of the world with its size, modification time and SHA-256 checksum. The manifest is written **last**, atomically: a backup exists if and only if its manifest does, so an interrupted backup never leaves a half backup.
- The **objects** are named after the SHA-256 of the file content. A file that did not change since an earlier backup is the same object, stored once, however many backups contain it. `session.lock` is not saved.
- Region files (`.mca`, `.mcc`) are stored **as they are**: Minecraft has already compressed their chunks, so compressing them again costs a lot of CPU for almost no space. Every other file is deflated (`compressionLevel`).

### Region files

Minecraft rewrites a chunk inside its region file, in place, when the chunk still fits. So between two backups most of a region is byte for byte the same, and storing the whole region again for one changed chunk would waste most of what it writes.

The first copy of a region is stored as one object: most regions of a big world never change again, and cutting them all would make the first backup of a 50 GB world hundreds of thousands of small files. When a region that is already in an earlier backup changes, it is stored in **pieces**: the header, and the chunks, each cut where a chunk ends (small chunks are grouped to about 8 KiB). Each piece is an object named after its checksum, and a small **recipe** in `recipes/` lists the pieces in order. From then on, a backup stores only the pieces that are new, plus the recipe.

Everything else works the same: a manifest names the file by the checksum of the whole file, and restoring, exporting and `/backup verify` put the pieces back together and check the checksum of the result. A recipe has a checksum of its own, so a damaged one is never used. The pieces of a region are deleted, like any other object, when no backup uses them.

Backups are named after the time they were taken, in UTC, optionally followed by your label (`/backup start before the big update`).

### Why you can trust the files

- Every object is written to a temporary file, flushed to disk and renamed in one atomic step: it appears in the store only complete.
- Right after, the mod **reads it back** and compares it with its checksum. A damaged object is deleted and the backup fails: no later backup will ever rely on it.
- The manifest is written only after every object it lists is in the store.
- `/backup restore` checks every file of the backup again before it schedules anything, and the restore checks every file as it writes it.
- `/backup verify` reads **all** the stored files and checks them against their checksums, whenever you want a complete check (bit rot, a disk problem, a file deleted by hand).

### How "unchanged" is decided

Each file is compared with the previous backup by **size and modification time**, like rsync does by default. An unchanged file is not read at all: the backup just records the checksum the previous backup already knew.

That check can be fooled: a file rewritten without changing its size, on a filesystem that stores the time with a resolution of seconds (FAT, some network shares), could look unchanged. So:

- a file modified in the few seconds **before the previous backup started** is never trusted, and is read again;
- a file modified in the few seconds before it is copied is compared with its copy byte by byte, so a change while copying is caught;
- the content of every file the backup takes from the previous one must still be in the store; if it is not, the file is read again.

### Full backups

For the rare case that a file changed without changing its size or time, there is a way to read **every** file again:

- `/backup full [name]` takes a backup that hashes every file. Nothing already in the store is stored twice, so the space used does not grow; only the time does. Without a reflink, autosave stays off for as long as a copy of the whole world takes, so run it in a quiet moment.
- With `fullScanEvery` in the config (0 by default: never by itself), every N-th backup is a full one.

## Reflink or copy

How the snapshot is taken decides how long autosave stays paused.

- **Reflink (copy-on-write)**: the filesystem makes a "clone" that shares the same disk blocks as the original until one of them changes. It takes milliseconds and uses almost no extra space, whatever the size of the world.
- **Copy**: the files that changed are copied one by one (a few at a time), and each copy is checked to make sure the original did not change while it was being copied. Autosave stays paused for the whole copy, which is short when little changed.

Reflink needs a filesystem that supports it and works on Linux and macOS:

| System | Reflink support |
|---|---|
| Linux, Btrfs | yes |
| Linux, XFS (created with reflink enabled) | yes |
| Linux, OpenZFS 2.2 or newer | possible, if block cloning is enabled (also depends on your `cp` version) |
| Linux, ext4 | no, normal copy |
| macOS, APFS | yes |
| Windows | no, normal copy |

You do not need to configure anything. With `allowReflink` on (the default) the mod tries a reflink first and falls back to a normal copy by itself. The backup message says which one was used (`snapshot by reflink` or `snapshot by copy`).

The snapshot is created in a temporary folder `.tickless-staging` **next to your world folder**, because a reflink only works inside one filesystem. Keep your world on a filesystem that supports reflink if you want the fast path. `backupDir` can be on another disk.

A hard link is never used for snapshots. Minecraft modifies region files in place, so a hard-linked "backup" would change together with the world.

## Disk space

The mod checks that there is enough free space, with a 5% margin, and fails with a clear message if not (`Not enough free space for …`):

- for a normal copy, the filesystem of the world needs the size of the **files that changed** for the temporary snapshot (for the first backup, the size of the world). A reflink uses almost no space, so it is not checked (if the disk is really full the reflink fails and the normal copy, with its check, is used);
- the filesystem of `backupDir` needs the size of the files that changed, to store them (compression and files already in the store usually make the real need smaller);
- a restore needs the size of the backup's files, free next to the world. This is checked when you run `/backup restore` (the restore is not scheduled if the space is missing) and again at shutdown, before anything is extracted;
- `/backup export` needs the size of the backup's files, free in the `exports` folder.

If a reflink fails 3 times in a row on the same filesystem, the mod stops trying it there until the server restarts and uses the normal copy directly (the log says so). A single failure is not enough, because it can be temporary.

The temporary snapshot is deleted as soon as the files are stored, and after a failure too.

### What retention deletes

Retention runs after every successful backup and deletes the **manifests** of old backups (see [Configuration](configuration.md#retention)). Then it deletes the stored files that **no remaining backup uses any more**. A file that a newer backup still uses stays, so deleting an old backup frees only what was unique to it.

For safety:

- if a manifest cannot be read, **nothing is deleted from the store**: its files would look unused;
- a stored file younger than one hour that nobody uses is kept, in case something else is still writing to the same folder;
- the newest backup, and the backup a scheduled restore is waiting for, are never deleted, and neither are the files they need.

## Off-site copy

With [`upload`](configuration.md#upload) on, every backup is also copied to an S3-compatible storage or a WebDAV server. The remote has the same layout as the backup folder of the world (`manifests/`, `objects/`, `recipes/`), so it is a copy of it, and the same rule holds: a backup exists if and only if its manifest does, and the manifest is sent last. A transfer that is cut leaves unused files, never a backup that cannot be restored. Only what the remote lacks is sent, on a thread of its own after the backup has ended, so the upload never delays autosave or the next backup. `/backup fetch` goes the other way and checks every file against its hash before it is installed.

## Standalone zips

`/backup export <name>` writes a backup as a normal `.zip` in the `exports` folder. Any zip tool opens it, you can copy it anywhere, and you can restore it by hand without the mod (see [Restoring a backup](restoring.md#restoring-by-hand-without-the-mod)). The zip is checked after it is written. Exports are never deleted by retention.

## Consistency: what is guaranteed

- A backup is a snapshot of the world at one moment, taken while autosave is paused.
- With `saveBeforeBackup` on (the default) the server saves right before the snapshot, so the backup holds the world as it is *now*.
- Even with autosave paused, Minecraft still rewrites a few files now and then (`level.dat`, `data/`). If a file changes while it is being copied (or cloned), or a file the backup took from the previous one is modified meanwhile, the snapshot is retried (up to 3 times, still with autosave paused). A snapshot without `level.dat` is never accepted. If it cannot be made consistently, the backup **fails and says so**. It never silently stores an inconsistent world.
- Do not run `/save-on` or `/save-all` while a backup is running. It can make the backup fail.

## Autosave safety

- Autosave is turned back on right after the snapshot, and again at the end of every backup, whether it succeeded or failed.
- This holds even when the server is lagging: if the backup gives up waiting for the pause, a pause that arrives late is cancelled and never turns autosave off.
- If you had already turned autosave off yourself with `/save-off`, the mod does not turn it back on for you. Those dimensions are not saved by `saveBeforeBackup` either (that is what `/save-off` means): for them the backup holds the last save.
- When the server stops, the mod makes sure autosave is on before the server's final save, so that save is never skipped.

## One operation at a time

Backups, exports and checks run one at a time on the same background thread: they never overlap, so none of them can see the half-written state of another. `/backup status` says what is running. Listing backups and checking a restore run on a separate thread, so a long backup does not block `/backup list`.
