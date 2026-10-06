# Configuration

The configuration file is `config/tickless-backups.json`, inside your server folder. It is created with default values the first time the server starts with the mod.

Edit it with the server running or stopped, then apply it with `/backup reload` (or restart the server). New settings apply from the **next** backup.

## The configuration screen

You can change the settings, and start backups, from a screen in the game instead of editing the file. The screen is **another way to reach the same configuration**: it shows what is in `config/tickless-backups.json` and writes the same file, so a change made in the screen can be seen in the file and the other way round.

Open it with:

- the **Backups** button at the top left of the pause menu;
- `/backup gui`;
- a key, in Controls under *Tickless Backups* (not assigned by default).

It has six tabs: **Backups** (start a backup or a full backup, with a label; the progress shows above the hotbar), **Schedule**, **Retention**, **General** (including the compression method), **Alerts** (which events send a notification, and the free-space limit) and **Checks and upload** (the automatic check of the backups, and whether uploads are on). If the tabs do not fit in one row they go on two; no text is ever cut. **Save** sends the values to the server; a wrong value (a time that is not `HH:mm`, a number that is not a number) is refused with a message and nothing is changed. Changes apply from the next backup, as with `/backup reload` (which the screen does for you).

How it works with a server: the screen runs in your game and asks the server for the configuration, so it also works for an operator connected to a **dedicated server**, as long as their game has the mod too. The server checks that you may use `/backup` (the same permission as the commands) before it sends or changes anything. `backupDir` cannot be changed from the screen: it decides where files are written on the server, so it stays something only the file can change. The same holds for everything that reaches outside the server or runs a program: the **webhook address**, the **upload remote and its credentials** and the **hooks** are never sent to the screen (it only learns whether an address or a secret is set) and an edit from the screen cannot change them. The screen can only turn uploads on or off, and a request to turn them on without a remote in the file is refused.

The server builds what it sends from an **allow-list**: only the settings the screen shows or edits are copied, so a setting added in a later version is not sent until someone adds it on purpose. A player can ask for the screen once every half second and save or start a backup once a second (faster requests are dropped), the size of every message is limited, and a save is refused, writing nothing, if the file was edited by hand at the same moment.

## Default file

```json
{
  "compressionLevel": 3,
  "compressionMethod": "deflate",
  "backupDir": "backups",
  "allowReflink": true,
  "drainTimeoutSeconds": 60,
  "saveBeforeBackup": true,
  "fullScanEvery": 0,
  "retention": {
    "maxCount": 20,
    "maxAgeDays": 30,
    "maxTotalBytes": 0
  },
  "schedule": {
    "enabled": false,
    "intervalMinutes": 60,
    "times": [],
    "timeZone": "",
    "onlyWithPlayers": true,
    "announce": true,
    "label": "auto"
  },
  "verify": { "enabled": false, "intervalHours": 24, "latestOnly": false },
  "upload": {
    "enabled": false,
    "type": "s3",
    "endpoint": "",
    "bucket": "",
    "region": "us-east-1",
    "pathStyle": true,
    "prefix": "tickless-backups",
    "accessKey": "",
    "secretKey": "",
    "username": "",
    "password": "",
    "allowInsecureHttp": false,
    "afterEachBackup": true,
    "timeoutSeconds": 120,
    "attempts": 4,
    "multipartThresholdMiB": 64,
    "partSizeMiB": 16,
    "threads": 2,
    "remoteRetention": { "enabled": false, "keepAtLeast": 10, "maxDeletesPerRun": 5, "graceDays": 7 }
  },
  "hooks": {
    "before": { "command": [], "timeoutSeconds": 0, "onFailure": "continue" },
    "duringPause": { "command": [], "timeoutSeconds": 0, "onFailure": "continue" },
    "after": { "command": [], "timeoutSeconds": 0, "onFailure": "continue" }
  },
  "notifications": {
    "url": "",
    "format": "discord",
    "serverName": "",
    "events": { "backupFailed": true, "backupCompleted": false, "verifyProblems": true, "lowSpace": true },
    "lowSpaceGiB": 5,
    "timeoutSeconds": 10,
    "allowInsecureHttp": false
  }
}
```

If you leave an option out, its default value is used. This is also why a config written by an older version of the mod keeps working after an update. A section or a text written as `null` counts as left out.

When the mod reads a valid file that lacks some options (a file from an older version), it writes the file again with **every option**, keeping the values you had, so you can see and edit all of them. A file with an error is never rewritten. If the file cannot be written (read-only), the defaults are simply used.

The file can hold passwords and secret addresses. Where the system has file permissions (Linux, macOS), a file that the mod writes is readable only by its owner, and a file that already existed keeps the permissions you gave it. On Windows, keep the folder `config` out of reach of the people you do not trust.

## Options

### `compressionLevel`

| Default | Allowed |
|---|---|
| `3` | `0` to `9` |

How much the stored files are compressed (deflate). `0` stores them as they are (fastest, biggest). `9` is the smallest and slowest. Values outside the range are clamped.

It applies to everything **except region files** (`.mca`, `.mcc`): Minecraft has already compressed their chunks, so they are always stored as they are, because compressing them again costs a lot of CPU for almost no space. It also applies to the zips made by `/backup export`. Compression runs on background threads, never on the server thread.

A removed option you may still have in an older file: `format` is no longer read (there is only one format now) and is ignored.

### `compressionMethod`

| Default | Allowed |
|---|---|
| `"deflate"` | `"deflate"` or `"zstd"` |

How **new** files (those that are not region files) are compressed in the store.

- `deflate` is the classic method, understood everywhere.
- `zstd` (Zstandard) is faster than deflate for a similar size. It is written by a pure-Java library that is bundled inside the mod jar: there is no native code and nothing to install. The library has a single speed, so with `zstd` the `compressionLevel` only decides whether to compress (`0` = no compression, anything else = compress).

You can change it at any time. Files already in the store keep working whatever the setting is (they are named `<sha256>`, `<sha256>.z` for deflate and `<sha256>.zst` for zstd, and all three are read), and a file that is already stored is never stored again because the method changed. Region files are not compressed in any case. The zips made by `/backup export` are always deflate: a zip that every tool opens, which is the point of an export.

Note: on recent Java versions the zstd library makes the JVM print a one-time warning about `sun.misc.Unsafe` in the log. It is harmless today. A future Java version that removes that access would stop `zstd` from working; `deflate` is not affected, and objects already written in zstd would then need a Java version that still allows it to be read.

### `backupDir`

| Default | Allowed |
|---|---|
| `"backups"` | A relative or absolute path |

Where backups are stored. Each world has its own subfolder, named after the world folder: with the default value the backups of `world` are in `backups/world/`, with `manifests`, `objects` and `exports` folders inside (see [How it works](how-it-works.md#where-the-backups-are)).

- A relative path is relative to the server folder.
- An absolute path can point anywhere, for example another disk.
- It **cannot be inside the world folder**. If it is, backups fail with `The backup folder cannot be inside the world folder`.
- It **cannot be inside `.tickless-staging`** (the temporary folder next to the world, wiped at every backup).
- On Windows, write backslashes doubled in JSON: `"D:\\Minecraft\\backups"`. Forward slashes also work: `"D:/Minecraft/backups"`.
- An empty value uses `"backups"`.

```json
"backupDir": "/mnt/storage/minecraft-backups"
```

### `allowReflink`

| Default | Allowed |
|---|---|
| `true` | `true` or `false` |

If `true`, the mod first tries a copy-on-write snapshot (reflink), which takes milliseconds and almost no extra disk space. If the filesystem does not support it, the mod automatically falls back to a normal copy. You never need to turn this off for it to work.

Set it to `false` only if you want to force the normal copy. See [How it works](how-it-works.md#reflink-or-copy) for the supported filesystems.

### `drainTimeoutSeconds`

| Default | Allowed |
|---|---|
| `60` | `1` or more |

Before taking the snapshot, the mod waits for Minecraft to finish writing pending data to disk. This is the longest it will wait. If the time runs out the backup fails and autosave is turned back on. Increase it on slow disks or very busy servers.

### `saveBeforeBackup`

| Default | Allowed |
|---|---|
| `true` | `true` or `false` |

If `true`, the server does a normal world save (like `/save-all`, without blocking waits) right before the backup starts. The backup then contains the world **as it is now**.

The cost is a short spike on the server thread, similar to an autosave. The time is logged: `World saved before the backup in N ms (server thread)`.

If `false`, there is no extra save. The backup contains what was last saved to disk by the server's regular autosave, so changes made since then (up to a few minutes) may be missing. Use this only if you want to avoid even that one spike.

### `fullScanEvery`

| Default | Allowed |
|---|---|
| `0` (never by itself) | `0` or more |

Every this many backups, the backup reads and hashes **every** file instead of trusting size and modification time (see [How "unchanged" is decided](how-it-works.md#how-unchanged-is-decided)). With `3`, the pattern is: full, normal, normal, full, normal, normal, …

It protects against the rare case of a file that changed without changing its size or time. It costs time, not space: files already in the store are not stored again. But without a reflink, autosave stays off for as long as a copy of the whole world takes, on every N-th backup. That is why it is off by default; you can also take a full backup on request with [`/backup full`](commands.md#backup-full-name) in a quiet moment.

### `schedule`

Automatic backups, made without anyone typing `/backup start`. **Off by default**: set `enabled` to `true` to turn it on.

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Whether automatic backups are made. |
| `intervalMinutes` | `60` | A backup every this many minutes after the **last backup of any kind**, so a backup you made by hand postpones the next one. `0` = not by interval. At least `1`. |
| `times` | `[]` | Also a backup at these times of the day, as `"HH:mm"`: `["04:30", "16:30"]`. A time that passes while the server is off is not made up for. |
| `timeZone` | `""` | The time zone of `times`, such as `Europe/Rome` or `UTC`. Empty = the time zone of the server. |
| `onlyWithPlayers` | `true` | Skip a due backup if nobody has been online since the previous automatic one: nothing that matters changed. In singleplayer you are always online. |
| `announce` | `true` | Tell the operators who are online when an automatic backup finishes. A failure is always told. |
| `label` | `"auto"` | The label that ends the name of automatic backups (`2026-10-05_04-30-00_auto`), so they are easy to tell apart. |

You can use the interval, the times, or both. Example: a backup every 30 minutes, plus one at 04:30 Rome time:

```json
"schedule": { "enabled": true, "intervalMinutes": 30, "times": ["04:30"], "timeZone": "Europe/Rome" }
```

An automatic backup is an ordinary backup: it pauses autosave only for the snapshot, shows the progress bar, and goes through retention. If another backup, export or check is running when one is due, it starts as soon as that ends. A wrong time or time zone makes `/backup reload` (or the start of the server) fail with a message that says which one; the file is never overwritten. `/backup status` shows when the next automatic backup is due.

The schedule counts from the last backup and does not trust the clock blindly: if the clock of the machine is set back (a correction of the time), the schedule goes on from the present instead of waiting for the clock to catch up with the old moment, and if it is set forward, a backup that is due is made once. The same goes for the automatic check below.

### `verify`

An automatic check of the stored backups, started by itself like [`/backup verify`](commands.md#backup-verify-name). **Off by default.**

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Whether the check is made by itself. |
| `intervalHours` | `24` | Check every this many hours after the last check (or after the start of the server), `1` to `8760`. |
| `latestOnly` | `false` | Check only the newest backup, which reads far less. With `false` every backup is checked, each stored file once. |

The check reads files, so on a big store it uses the disk for a while; it runs in the background, never while a backup is running (it waits and starts when the manager is free), and never touches the world. When it ends:

- the result is written in the log and shown by [`/backup status`](commands.md#backup-status) (`Last scheduled check: …`, and when the next one is due);
- if it found damaged or missing files, or could not run, the operators who are online are told, and a [notification](#notifications) is sent (`events.verifyProblems`). A check that finds everything fine is only logged.

The time is counted in memory: after a restart the first check is due `intervalHours` after the server started.

### `upload`

Copies every backup to a place away from this machine, so that losing the machine does not lose the backups. **Off by default.** Two kinds of remote are supported, both written in this mod on top of the HTTP client of Java (no SDK and no other library): an **S3-compatible object storage** (AWS S3, MinIO, Backblaze B2, Cloudflare R2, ...) and a **WebDAV server** (Nextcloud, ownCloud, Apache, nginx with DAV, ...). SFTP is not supported.

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Whether backups are uploaded. |
| `type` | `"s3"` | `"s3"` or `"webdav"`. |
| `endpoint` | `""` | S3: the address of the service, such as `https://s3.eu-west-1.amazonaws.com` or `https://minio.example.com:9000`. WebDAV: the folder that holds the files, such as `https://cloud.example.com/remote.php/dav/files/me`. |
| `bucket` | `""` | S3: the bucket. It must exist. |
| `region` | `"us-east-1"` | S3: the region used to sign requests, such as `eu-west-1`. MinIO and most services accept any. |
| `pathStyle` | `true` | S3: `https://host/bucket/key` (every service accepts it) or, with `false`, `https://bucket.host/key`. |
| `prefix` | `"tickless-backups"` | The folder for the backups of this server. Each world has its own subfolder (`<prefix>/<world folder>/`). Use another prefix for another server in the same bucket. |
| `accessKey`, `secretKey` | `""` | S3: the key pair. **Secrets.** |
| `username`, `password` | `""` | WebDAV: the login (empty for none). **Secrets.** |
| `allowInsecureHttp` | `false` | Plain `http` is accepted only for this machine (`localhost`, `127.0.0.1`, `::1`). For another machine you need `https`, or this option set to `true` if the network is yours. |
| `afterEachBackup` | `true` | Upload after every successful backup. With `false`, only `/backup upload now` uploads. |
| `timeoutSeconds` | `120` | The time a request may take, `5` to `3600`. A big file gets more time (at least 64 KiB/s are assumed). |
| `attempts` | `4` | How many times each operation is tried (with pauses of 1 s, 2 s, 4 s, ... up to 30 s), `1` to `10`. |
| `multipartThresholdMiB` | `64` | S3: files larger than this are sent in parts, `5` to `5000`. |
| `partSizeMiB` | `16` | S3: the size of a part, `5` to `128`. A part is held in memory while it is sent, and `threads` files are sent at once. |
| `threads` | `2` | How many files are sent at once, `1` to `8`. |
| `remoteRetention` | off | Deleting old backups from the remote, see below. |

```json
"upload": {
  "enabled": true, "type": "s3", "endpoint": "https://minio.example.com:9000",
  "bucket": "minecraft", "accessKey": "…", "secretKey": "…"
}
```

How it works:

- After a successful backup, an upload starts **on a thread of its own**. It does not use the backup worker, so it never delays the autosave resume or the next backup. Requests that arrive while an upload runs become one more run. At server start, and after a failure, the next run sends whatever the remote lacks, newest backup first, so the remote catches up by itself.
- Only what the remote lacks is sent: a stored file (or a piece of a region file, or a recipe) that the remote already has, in any of its forms (`<sha256>`, `.z`, `.zst`), is not sent again. The remote has the same layout as the backup folder of the world: `manifests/`, `objects/`, `recipes/`.
- **A backup exists on the remote if and only if its manifest does.** The order is: stored files and pieces, then recipes, then the **manifest last**. A transfer that is cut leaves files that no manifest uses yet, never a backup that cannot be restored. S3 makes a file appear only when it is complete (also for the parts of a big file, which are aborted on a failure); for WebDAV a file is sent as `<name>.part` and renamed with `MOVE` when it is complete (a server without `MOVE` cannot be used).
- Every request is signed (S3: AWS Signature Version 4, which covers the SHA-256 of the body, so the server refuses a damaged body), has a timeout and is tried again for a network error, a server error or "too many requests". A wrong login or a missing bucket is not repeated.
- **The remote is not trusted.** What it sends is checked before it is used: an answer is read only up to a limit (64 KiB for an answer to a command, 8 MiB for a page of an S3 listing, 16 MiB for the listing of a WebDAV folder) and only for a limited time, a download stops at the size it must have (a manifest at most 64 MiB, a recipe 16 MiB, a stored file a little more than its size), a listing is refused if it repeats its pages, goes in circles or has more than 5 million files, a name in a listing that could lead outside the store (`..`, an empty part, a backslash, a control character) is ignored, and the XML of the answers is read by a parser that refuses DOCTYPE and external entities. A stored file is installed only after its content has been read the way it is stored and has the size and SHA-256 it is named after. A remote that is hacked, or an address typed wrong that reaches another server, can make a download fail; it cannot fill the disk, use the memory, or put into the store a file that does not match its hash.
- Credentials go only to `endpoint`: redirects are never followed. They are never written in the log or in an error message, and never sent to the configuration screen.
- A failed upload is logged, shown to the operators who are online, and kept for `/backup upload status` and `/backup status`. A backup that is being uploaded is not deleted by retention meanwhile; if a file it needs is gone from the local store, that backup is skipped (and logged) and the manifest is not sent.
- `/backup upload status` shows how it is going, `/backup upload now` uploads now, and `/backup fetch <name>` brings a backup from the remote into the local store, verified by hash (see [Commands](commands.md#backup-fetch-name)).

#### `remoteRetention`

Deleting from the remote is **off by default and written to delete as little as possible**, because the remote copy is the one that survives the loss of the machine.

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `false` | Whether anything is deleted from the remote. Cannot be turned on from the screen. |
| `keepAtLeast` | `10` | The remote always keeps at least this many backups (at least `1`). |
| `maxDeletesPerRun` | `5` | At most this many backups are deleted from the remote in one run, `1` to `1000`. |
| `graceDays` | `7` | Stored files younger than this are never deleted from the remote, `1` to `3650`. |

When it is on, the remote **follows the local retention**: a backup is deleted from the remote only if this machine uploaded it **and** it is no longer here (the local retention deleted it). A backup that someone else put on the remote is never touched. After deleting manifests, the stored files that no remaining backup uses, and that are older than `graceDays`, are deleted too. Nothing is deleted if there are no local backups, if the newest local backup is not on the remote yet, if a local manifest cannot be read, or if no backup would remain; and no stored file is deleted if a remaining manifest cannot be read, or if the remote does not say when the file was written (without a date, the protection of the young files cannot work). The cleanup also runs when there is nothing new to upload. What this machine remembers of its uploads is in `upload-state.json` in the folder of the backups of the world, and it is tied to one remote: pointing the configuration elsewhere makes it start again from nothing (which only means less is deleted).

A wrong value (no endpoint, no bucket or keys for S3, an address that is not `http` or `https`, plain `http` to another machine, a prefix with `..`) makes `/backup reload` or the start of the server fail with a message that says which option and does not repeat the address or a secret. The file is never overwritten.

### `hooks`

Commands to run around a backup, for example to flush a database or tell a plugin to save. **None by default.** There are three places:

| Hook | Runs | Default timeout | Longest allowed |
|---|---|---|---|
| `before` | Before anything is touched. Autosave is still on. | 30 s | 3600 s |
| `duringPause` | While autosave is **paused**, right before the snapshot. Every second it takes is a second of autosave off: keep it very short. | 10 s | 120 s |
| `after` | In the background, after the backup has ended, whether it went well or not. It never delays the next backup. | 60 s | 3600 s |

Each hook has these options:

| Option | Default | Meaning |
|---|---|---|
| `command` | `[]` | The program and its arguments, **as a list**: `["/usr/local/bin/flush.sh", "--fast"]`. Empty = no hook. |
| `timeoutSeconds` | `0` | How long the command may run before it is stopped (with everything it started). `0` = the default of the hook (see the table). |
| `onFailure` | `"continue"` | `"continue"` logs a command that fails or runs out of time and goes on with the backup. `"abort"` cancels the backup. Ignored for `after`, which cannot cancel a backup that has happened. |

```json
"hooks": {
  "duringPause": { "command": ["/usr/local/bin/flush-db.sh"], "timeoutSeconds": 5 },
  "after": { "command": ["/usr/local/bin/copy-backup.sh"] }
}
```

How it behaves:

- **No shell.** The command is a list of arguments that are passed to the program exactly as written. Nothing is split, expanded or interpreted (`*`, `$HOME`, `;` and `|` are just characters). If you need a shell, make that explicit: `["/bin/sh", "-c", "…"]` or a script.
- **A hook can never leave autosave off or break a backup**, unless you ask it to with `"abort"`. A command that fails, cannot be started or runs out of time is logged and the backup goes on. The timeout of `duringPause` is at most 120 seconds, and autosave is resumed in every case, also when the hook makes the backup abort.
- The command gets **no input** (reading it ends at once). What it prints (output and errors) goes to the server log as `[hook <name>] …`, up to 200 lines per run.
- Variables the command can read, besides those of the server: `TICKLESS_PHASE` (`before`, `during_pause` or `after`), `TICKLESS_BACKUP_NAME`, `TICKLESS_WORLD_DIR`, `TICKLESS_BACKUP_DIR` (this world's backup folder), `TICKLESS_SNAPSHOT_DIR` (the temporary folder where the snapshot is taken; it may not exist yet in `before`), and, for `after` only, `TICKLESS_RESULT` (`success` or `failed`), `TICKLESS_MANIFEST` (the manifest, if it succeeded) and `TICKLESS_ERROR` (if it failed).
- When the server stops, a command still running is stopped.
- **Only the file can change hooks.** Whoever can edit the file can make the server run programs, so the configuration screen neither receives nor changes this section (it is hidden from it, and an edit from the screen leaves it as it is). Keep the file writable only by the people you trust, and do not put passwords in the arguments if you can avoid it: they would be readable in the file and in the process list.
- A wrong command (a blank program, an argument with a NUL character, more than 64 arguments) or a wrong `onFailure` makes `/backup reload` fail with a message that says which hook and does not repeat the command.

### `notifications`

A message to a **webhook** when something happens, so you know even when nobody is online. **Off by default**: it is on as soon as `url` is set.

| Option | Default | Meaning |
|---|---|---|
| `url` | `""` | The address of the webhook (`http` or `https`). Empty = no notifications. **A secret**: see below. |
| `format` | `"discord"` | `"discord"` for a Discord webhook (a message with a coloured embed), `"json"` for a JSON object that any other service can read. |
| `serverName` | `""` | The name that tells which server sent the message. Empty = the name of the world folder. |
| `events.backupFailed` | `true` | A backup failed. |
| `events.backupCompleted` | `false` | A backup was made. Off by default: with a backup every hour it is a lot of messages. |
| `events.verifyProblems` | `true` | A check (`/backup verify`, or the [automatic one](#verify)) found damaged or missing files, or could not run. |
| `events.lowSpace` | `true` | The disk of the backups has less than `lowSpaceGiB` free. Looked at after every backup and every 15 minutes; repeated at most every 6 hours. |
| `lowSpaceGiB` | `5` | The free-space limit for `lowSpace`, in GiB. `0` = never. |
| `timeoutSeconds` | `10` | The longest to wait for the webhook to answer, `1` to `60`. |
| `allowInsecureHttp` | `false` | Plain `http` is accepted only for this machine (`localhost`, `127.0.0.1`, `::1`). For another machine you need `https`, or this option set to `true` if the network is yours: the address is a secret and would cross the network in the clear. An address with a login in it (`https://user:pass@host`) is refused. |

The generic JSON looks like this (`event` is `backup_failed`, `backup_completed`, `verify_problems`, `low_space` or `test`; `severity` is `error`, `warning`, `success` or `info`):

```json
{"event":"backup_failed","severity":"error","title":"Backup failed","message":"...","server":"survival",
 "timestamp":"2026-10-05T12:00:00Z","fields":{"Backup":"2026-10-05_12-00-00","Duration":"1.2 s"}}
```

How it behaves:

- Messages are sent on a thread of their own, with timeouts. A webhook that is slow, down or wrong **never delays or fails a backup**. A message that fails with a network error, a server error or "too many requests" is tried up to 3 times; the others (a wrong address, for example) are not repeated. If many messages pile up, the new ones are dropped.
- **The address is never written in the log** (a failure is logged with the HTTP status or the type of the error only) and redirects are not followed.
- **The address is not sent to the configuration screen** and cannot be changed from it: the screen shows only whether it is set. The same goes for `format`, `serverName`, `timeoutSeconds` and `allowInsecureHttp`. The screen can turn the events on and off and change `lowSpaceGiB`. An operator who can type `/backup` cannot point the messages to another place.
- A wrong address or format makes `/backup reload` (or the start of the server) fail with a message that does not repeat the address; the file is never overwritten.
- Anyone who can edit the file can make the server post to any address, including one inside your network (the answer is never read or shown, only whether it was accepted). Keep the file writable only by the people you trust.
- The text of the messages can have file names and error texts of the server. In Discord, a message never turns text into a mention (`@everyone`, roles).
- `/backup notify test` sends a test message, to try the address.

### `retention`

Automatic deletion of old backups. It runs after every **successful** backup. Set any value to `0` to turn that rule off.

| Option | Default | Meaning |
|---|---|---|
| `maxCount` | `20` | Keep at most this many backups |
| `maxAgeDays` | `30` | Delete backups older than this many days |
| `maxTotalBytes` | `0` (off) | Delete the oldest backups until the space taken by the store is under this many bytes |

Deleting a backup deletes its manifest and then the stored files that **no remaining backup uses**. Because backups share their files, deleting an old backup frees only what was unique to it, and `maxTotalBytes` counts what deleting a backup really frees (a file shared with newer backups does not count). With `maxTotalBytes`, the mod keeps the list of files of every backup in memory while it runs: on a very large store with many backups, prefer `maxCount` and `maxAgeDays`.

How the rules combine:

- A backup is deleted if **at least one** rule says it should go.
- **The newest backup is never deleted**, whatever the rules say, even if it alone exceeds `maxTotalBytes` or is older than `maxAgeDays`.
- **A backup with a scheduled restore is never deleted** either. Like the newest one, it still takes one of the `maxCount` slots and counts towards `maxTotalBytes`. The files it needs stay in the store.
- Rules apply to each world separately: only the backups in that world's subfolder of `backupDir` are considered. Other files are left alone.
- If a manifest in the `manifests` folder cannot be read, **no stored file is deleted** until you fix or remove it: the files it needs would look unused.
- A failure while deleting never invalidates the backup that was just made.

#### Examples

Keep the last 10 backups and nothing else:

```json
"retention": { "maxCount": 10, "maxAgeDays": 0, "maxTotalBytes": 0 }
```

Keep one week of backups, at most 50 of them:

```json
"retention": { "maxCount": 50, "maxAgeDays": 7, "maxTotalBytes": 0 }
```

Keep backups under 10 GiB in total (`10 × 1024 × 1024 × 1024 = 10737418240` bytes):

```json
"retention": { "maxCount": 0, "maxAgeDays": 0, "maxTotalBytes": 10737418240 }
```

Never delete anything automatically:

```json
"retention": { "maxCount": 0, "maxAgeDays": 0, "maxTotalBytes": 0 }
```

## If the file is invalid

If `tickless-backups.json` is not valid JSON (a missing comma, for example), the mod does **not** overwrite or reset it. Instead:

- At server start, the mod logs `Tickless Backups could not start: Invalid JSON in tickless-backups.json: …` and the `/backup` commands answer `Tickless Backups is not running: …`.
- Fix the file, then run `/backup reload`. The mod starts without restarting the server.

If you delete the file, a new one with the defaults is created at the next start (or reload).

## Other files the mod uses

| File | Purpose |
|---|---|
| `config/tickless-backups/pending-restore/<world>.json` | Exists only while a restore is scheduled for that world. Created by `/backup restore`, removed once the restore has been applied or cancelled. See [Restoring a backup](restoring.md). |
| `config/tickless-backups/pending-restore/<world>.json.invalid` | A restore request the mod could not read, or one written for a different world with the same folder name. Kept so you can inspect it. Safe to delete. |
| `.tickless-staging/` (next to the world folder) | Temporary snapshot (only the files that changed) while a backup runs. Removed automatically, even after a failure. |
| `<backupDir>/<world>/tmp/` | Stored files being written. Cleaned automatically at the start of every backup. |
