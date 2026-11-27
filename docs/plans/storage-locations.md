# Storage locations: recordings go where the household wants them, and are safe when a disk goes away

This is the technical companion of the epic "Storage locations" on the project board. It is
written so a developer or a coding agent can pick up a step and start.

Related plans:

- [Server foundation](server-foundation.md): config, secrets, API, health.
- [Recorder and playback](recorder-and-playback.md): owns the segment file, the layout of
  files on a location, the index (including the `placements` table this plan writes to),
  retention and emergency cleanup.
- [RTSP ingest and the ring](rtsp-ingest-and-ring.md): reports the bitrates this plan uses to
  judge whether a location is fast and large enough.

This plan owns the **location interface**, the **storage manager** (finding, checking and
configuring locations, the primary location and background copies) and the **cloud connector
interface**.

## Problem

People keep recordings in different places: the disk of the box itself, a USB drive, a NAS.
Some want a second copy somewhere else, and later some want a copy in the cloud. Each of these
can be slow, full, or gone for a while: a NAS reboots, a USB drive is unplugged, a network share
fails to mount.

Recording must not depend on all of that going well:

- A location that is missing must never be mistaken for an empty folder. When a network share
  is not mounted, its mount point is an empty folder on the system disk; writing there fills
  the disk that Home Assistant runs from.
- A slow or offline second location must never slow down recording to the first.
- A person should know, before choosing, whether a location is fast enough and how many days of
  footage fit. A 1080p camera at 2 to 4 Mbit/s fills 20 to 45 GB a day when recording
  continuously; a Home Assistant Green has 32 GB of eMMC storage and USB 2.0 ports.

## What exists

**In Home Assistant.** On Home Assistant OS, disks and network shares (SMB and NFS) are set up
once in Home Assistant's settings and managed by the Home Assistant Supervisor. A network
share added with the usage "Media" appears to apps that map the media folder, below `/media`.
Backup agents (`homeassistant/components/backup/agent.py`) move whole backup files through
Home Assistant: no appending, no range reads, no listing by time. That model fits a nightly
backup and is the model for the experience of choosing a location, not for the data path.

**In Music Assistant.** Music Assistant's storage controller
([`music_assistant/controllers/storage`](https://github.com/music-assistant/server/tree/dev/music_assistant/controllers/storage))
already finds and manages locations for the same two environments:

- It reads the mount table of its own process (`/proc/self/mountinfo`). Inside a container that
  table holds the volumes and bind mounts given to it; under the Home Assistant Supervisor it
  also holds every network share and drive mounted below `/media`. It skips pseudo file systems
  and system paths, and tells network, removable and local file systems apart
  (`backends/mountinfo.py`).
- It notes that the Supervisor mounts a network share on first access: until then the table only
  holds an automount trigger (`autofs`) on its mount point.
- It can add network shares as mounts through the Supervisor's API (`backends/supervisor.py`),
  or mount them itself outside Home Assistant OS (`backends/local_mount.py`).
- Each location reports whether it is available, read-only, and its free, used and total space
  (`models.py`).

Camera Assistant uses the same model.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) keeps everything in one local
data folder and deletes the oldest events when free space drops below a set minimum.

**In other projects.** [Viseron](https://github.com/roflcoopter/viseron) moves segments between
storage tiers by age (for example from an SSD to a NAS). [`object_store`](https://crates.io/crates/object_store) and
[`opendal`](https://crates.io/crates/opendal) are Rust crates that put many storage services
behind one interface; how well they fit here is not checked yet.

## Concept

### Words

- **Location**: one place where the server can keep files. It has an id, a name and a root.
- **Folder location**: a location in the file system the server is given (a folder, a volume,
  a mounted drive or share).
- **Cloud location**: a location reached through a cloud connector. Designed in this plan, built
  later.
- **Primary location**: the one location the recorder writes to.
- **Copy location**: a location that receives copies in the background.
- **Storage manager**: the part of the server that owns all of the above.

### The location interface

A small Rust trait. The recorder, the copier and playback use only this; they never touch a
path directly.

```rust
trait Location: Send + Sync {
    fn id(&self) -> &LocationId;
    fn capabilities(&self) -> Capabilities; // can_grow, can_read_range

    /// Write a file. `Growing`: readable while it is written, appended to fragment by
    /// fragment (segment files). `Whole`: visible only once complete (event files, stills,
    /// copies).
    async fn write(&self, path: &RelPath, mode: WriteMode) -> Result<Box<dyn LocationWriter>>;

    async fn read_range(&self, path: &RelPath, range: Range<u64>) -> Result<ByteStream>;
    async fn list(&self, prefix: &RelPath) -> Result<EntryStream>; // path, size, modified
    async fn delete(&self, path: &RelPath) -> Result<()>;
    async fn space(&self) -> Result<Space>; // free and total bytes
}

trait LocationWriter: Send {
    async fn append(&mut self, bytes: Bytes) -> Result<()>;
    async fn sync(&mut self) -> Result<()>;
    async fn finish(self: Box<Self>) -> Result<()>;
    async fn abort(self: Box<Self>) -> Result<()>;
}
```

- Paths are relative to the location's root and use `/`. They are checked: no `..`, no absolute
  paths.
- A folder location supports both write modes. `Whole` writes to a temporary name in the same
  folder and renames on `finish`, so a half-written file is never seen as complete.
- A cloud location supports `Whole` only (`can_grow` is false). That is why a cloud location can
  never be the primary location.
- The names above are a starting point; step 1 settles them.

### Where folder locations come from

The server does not mount anything itself in the Preview. It uses the file system it is given.

| Environment | How a location reaches the server |
|---|---|
| Plain container (Docker) | Volumes and bind mounts. A NAS share is mounted on the host and passed in as a bind mount. |
| Home Assistant OS | The app maps the media folder. Network shares added in Home Assistant's settings with the usage "Media" appear below `/media/<name>`. The app's own data folder is also available. |
| Development | Any folder given on the command line. |

**Finding candidates.** The storage manager reads `/proc/self/mountinfo` the way Music
Assistant does, and offers as candidates the mounts with a real file system outside system
paths, each with its kind (local, removable, network), its free and total space, and whether
it is writable. A network share behind an automount trigger is offered as well; the first
access mounts it. The user picks a candidate, or types a path, and the server checks it before
saving.

### Identity and the mount guard

- When a folder becomes a location, the server writes a marker file `.camera-assistant` at its
  root with the location id and the server id.
- Before the recorder opens a new segment file, and at every periodic check, the storage
  manager confirms that the marker is there and has the right id. For a location that was a
  separate mount when it was added, it also confirms that the path is still a mount (it is
  listed in the mount table, or its device differs from its parent's).
- If either check fails, the location is **missing**. Nothing is written to it. This is what
  keeps an unmounted share from filling the system disk.
- The marker also lets a rebuild of the index tell which location a folder is, and warns when two
  servers write to the same folder.

### Checks

Each location is checked when it is added, at server start, and while it is in use.

| Check | How | Result |
|---|---|---|
| Write access | Create, write, sync, read back and delete a small probe file | `writable` or the error |
| Speed | When added: write and sync a probe of some megabytes and measure. While in use: the time the recorder's real writes take, measured by the writer, not by extra writes. | Measured MB/s, compared with what the recorded cameras need (sum of their bitrates from ingest, with a margin) |
| Free space | `statvfs` on the root, every few seconds (cheap) | Free and total bytes |
| Fit | Free space and the current recording rate | Estimated days of footage that fit, shown before a choice is made |

**Status** of a location, reported through health and the API:

| Status | Meaning | What happens |
|---|---|---|
| `ok` | All checks pass | |
| `slow` | Writes are slower than recording needs | Health warning |
| `low_space` | Free space below the low threshold | The recorder runs emergency cleanup (recorder plan) |
| `full` | Below the threshold and nothing more can be deleted | Nothing more is written; see "When the primary location is not available" |
| `missing` | Marker or mount gone | Nothing is written or deleted; copies wait |
| `read_only` | Write check fails | Nothing is written |

The low threshold is the larger of a fixed amount and a share of the disk, and is higher on the
disk that Home Assistant itself uses (open question 1). The thresholds are constants in code,
not settings.

### Primary location and background copies

- **One primary location** for all cameras in the Preview. It must be a folder location. The
  recorder writes every segment file, event file and still there.
- **Copy locations** get copies of **closed** files only, by a rule per copy location:
  - `events`: the segment files of events, plus the event files and the stills;
  - `all`: everything.
- **The copier** is one background task in the storage manager. It copies one file at a time
  per copy location: `read_range` from the source, `write` in `Whole` mode to the target, then
  compares the size and adds a row to `placements` in the index.
- **The queue is derived, not stored.** The files that a copy location should have and does not
  have follow from the index (`segments`, `events` and `placements`). So the queue survives a
  restart, and a rebuild of the index restores it. Events go first, then the oldest footage.
- **A copy location that is offline** (`missing`) is skipped; its queue waits. Health shows the
  queue length and the age of the oldest waiting file. When the location is back, copying
  resumes on its own.
- **A copy never slows recording.** The copier has a low priority on disk and network, and it
  reads from the primary through the same interface as playback.
- **Retention runs per location** (recorder plan). A copy location can keep footage longer or
  shorter than the primary. When retention on the primary deletes a file that was not copied
  yet, that file is not copied any more; health reports it.
- **Playback** reads from the primary location when the file is there and from a copy when it is
  not.

### When the primary location is not available

When the primary location is `missing`, `read_only` or `full`, the recorder must keep going if
it can. Proposal (open question 2): the recorder writes to the first copy location that is a
folder location and is `ok`, and health raises an error that says so. The footage stays where
it was written and stays in the index. When the primary is back, new footage goes there again,
and the copier treats the primary as a copy target for the footage written meanwhile, by the
`all` rule. If no folder location is available, recording stops for all cameras and health
raises an error. The server's own data folder is never used as a fallback, because on Home
Assistant OS it shares the disk with Home Assistant.

### Settings and API

A location in `config.json`:

| Field | Meaning |
|---|---|
| `id`, `name` | Stable id; name shown to people |
| `kind` | `folder` or `cloud` |
| `path` | For a folder location: the root in the server's file system |
| `source` | How it was found: `container_volume`, `home_assistant_media`, `home_assistant_share`, `local_disk`, `manual` |
| `role` | `primary` or `copy` |
| `copy_rule` | For a copy location: `events` or `all` |
| `keep_days`, `max_gb` | Retention limits, used by the recorder |
| `enabled` | False pauses it without removing it |

Rules: exactly one `primary`; the Preview allows one primary and one copy location (the
"one local and one network location" of the Preview), and nothing in the design stops more
later. Removing a location removes it from the config and its rows from the index; **it never
deletes files**. Changing the primary does not move footage.

| Commands | Events |
|---|---|
| `locations/candidates`, `locations/list`, `locations/get`, `locations/add`, `locations/update`, `locations/remove`, `locations/test`, `locations/set_primary` | `location_updated` (status, space, queue) |

`locations/test` runs the checks on a path that is not a location yet, so the web interface can
show the result before saving.

### The cloud connector interface (designed now, built later)

A cloud copy of events is in the MVP, through an open connector interface. This plan designs
the interface so nothing built now has to change for it.

- **A connector implements `Location`** with `can_grow` false, plus:
  - `config_schema()`: the fields a user fills in (endpoint, bucket, key, and so on), so the web
    interface can draw the form without knowing the connector;
  - `test(config)`: checks the account and the permissions before saving.
- **A cloud location is always a copy location**, never the primary, because it cannot take a
  growing file.
- **Encryption before upload.** The storage manager encrypts every file before it reaches a
  connector, so a connector never sees video, and the provider stores only ciphertext. The
  format has to allow range reads (open question 4), so playback and partial restores work
  without downloading whole files. The key lives in `secrets.json`; the user is shown a
  recovery key, since without it a cloud copy cannot be read after the box is lost.
- **Credentials** for a cloud location are stored in `secrets.json`. The location interface keeps
  the source of a location replaceable, so a change in where credentials live stays inside the
  storage manager.
- **How connectors are added** is open question 3.

## Decisions

Settled by the project:

1. **Locations come from the file system the app is given, or from pluggable cloud
   connectors.** Reason: Docker and Home Assistant OS already know how to give an app a disk or
   a share; the app does not mount things itself.
2. **A location is a small interface: write, read a range, list, delete, report free space,
   with a source that can change.** Reason: the recorder and playback never depend on where the
   bytes are.
3. **One primary location, copies in the background, a queue while a location is offline.**
   Reason: recording never waits for a second location.
4. **Each location is checked for write access, speed and free space when it is added and while
   it is in use.** Reason: a problem shows before footage is lost.
5. **Cloud uploads are encrypted before they leave the house.** Reason: video of a home is
   private.

Proposed in this plan (to confirm in review):

6. **A marker file and a mount check guard every location.** Reason: an unmounted share must
   never become an empty folder on the system disk.
7. **The primary location is always a folder location.** Reason: segment files grow while they
   are written; object storage cannot do that.
8. **The copy queue is derived from the index.** Reason: no second state to keep in step, and a
   rebuild restores it.
9. **Removing a location never deletes files.** Reason: footage is the user's; deleting it is a
   separate, explicit action.
10. **No mounting by the server in the Preview.** Reason: mounting in a container needs extra
    rights; Home Assistant OS and the host already do it.

## Steps

1. **[server] Location interface and folder location.** The trait, the folder implementation
   with both write modes, `read_range`, `list`, `delete`, `space`, path checks, and tests on a
   temporary folder.
2. **[server] Finding candidate folders.** Parse `/proc/self/mountinfo`, skip pseudo file
   systems and system paths, classify local, removable and network mounts, include automount
   triggers. Unit tests with sample mount tables from Docker, Home Assistant OS and a desktop.
3. **[server] Identity and the mount guard.** The marker file, the mount check, the `missing`
   status, and tests that unmount or remove the marker while writing.
4. **[server] Location checks.** Write access, speed (probe at add, measured from real writes in
   use), free space, the thresholds, the status, the estimate of days that fit, and the health
   items.
5. **[server] Location settings and API.** The config fields, the rules for one primary and one
   copy location, the commands and the `location_updated` event.
6. **[server] Background copies.** The copier, the rules `events` and `all`, the queue derived
   from the index, whole-file writes with a size check, offline handling, and health for the
   queue.
7. **[server] When the primary location is not available.** The fallback described above, the
   return to the primary, and the health errors.
8. **[hass-app] Storage on Home Assistant OS.** The app definition comes from the project
   setup plan and maps the storage that the Supervisor grants to apps. This step adds what
   storage needs on top: exclude the index (and any recordings kept in the data folder) from
   backups with `backup_exclude`, choose the default primary location on first start, and test
   with a network share added in Home Assistant's settings.
9. **[documentation] Storage guide.** How to give the server a folder in Docker (volume, bind
   mount, a NAS mounted on the host), how to add a network share on Home Assistant OS, and how
   much space footage takes.
10. **[server] Cloud connector interface.** A design note in the repository (the trait additions,
    the encryption format, the recovery key, how connectors are added), and a test connector
    that behaves like object storage (whole files only) on a local folder, so copies are tested
    against a location that cannot grow.

## Order

Step 1 comes first; the recorder's segment writer needs it. Steps 2 and 3 need step 1 and can
run in parallel. Step 4 needs step 3, and the bitrates from ingest for the speed comparison.
Step 5 needs step 4 and the config store and API of the server foundation plan. Step 6 needs
step 5 and the index of the recorder plan. Step 7 needs steps 5 and 6. Step 8 needs step 5.
Step 9 can start once step 5 is clear. Step 10 needs step 6.

## Open questions

1. **What are the free-space thresholds?** Proposal: low is the larger of a few GB and a few
   percent of the disk, and higher on the disk Home Assistant uses. To decide with
   measurements on a Green, where the whole disk is 32 GB.
2. **Is a fallback to the copy location right?** It keeps recording going, but it fills the
   copy location with footage it was not meant to hold, and on a NAS it may be slow. The
   alternative is to stop recording and raise an error. Proposal: fall back, with a clear
   error.
3. **How are cloud connectors added?** (a) Rust crates compiled into the server behind the
   trait: simple and reviewed, but every connector is a server release. (b) Separate processes
   that speak a small protocol: any language and isolated, but a protocol to design and
   maintain. (c) One generic connector on top of a crate that covers many services, such as
   `object_store` or `opendal`, plus (a) for services it does not cover. Proposal: decide with
   the first real connector, in the MVP.
4. **Which encryption format?** It needs authenticated encryption in chunks, so a range of a
   file can be read and checked without the whole file. Options: an existing chunked format such
   as age (whether its Rust library supports reading a range has to be checked), or a small
   format of our own with one nonce and tag per fixed-size chunk. File and folder names leak
   camera ids and times; whether to encrypt names as well is part of the question.
5. **Where is the default primary location on Home Assistant OS?** The media folder is easy to
   find and to browse, but Home Assistant's own backups can include the media folder, so
   recordings could end up in backups. The app's data folder with `backup_exclude` keeps them
   out of backups but out of sight. Both are on the same disk as Home Assistant.
6. **How does a USB drive reach the app on Home Assistant OS?** To check what Home Assistant OS
   offers today for drives that are not the data disk.
7. **Does the server add network shares itself through the Supervisor's API, as Music Assistant
   does?** Proposal: not in the Preview; the user adds the share in Home Assistant's settings
   and picks it in Camera Assistant.

## Later, and what this must not block

- **Cloud copy of events (MVP)** through the connector interface.
- **Repairs in Home Assistant for a full or missing location (MVP).** The status and health
  items are the source.
- **Several locations with full rules (after the MVP):** what goes where and for how long, per
  camera and per kind of footage; a primary location per camera; continuous footage to the
  cloud. The config already has a role and a rule per location.
- **More cloud connectors (after the MVP),** for example S3-compatible storage, added by the
  community.
