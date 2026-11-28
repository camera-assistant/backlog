# Server foundation: one process that keeps every camera running and answers one API

This is the technical companion of the epic "Server foundation" on the project board. It is
written so a developer or a coding agent can pick up a step and start. The server lives in
`camera-assistant/camera-assistant` and is written in Rust.

Related plans, which build on this one:

- [RTSP ingest and the ring](rtsp-ingest-and-ring.md) (ingest, ring)
- [Recorder and playback](recorder-and-playback.md) (recorder, index, playback)
- [Storage locations](storage-locations.md) (storage manager)

## Problem

Camera Assistant has to run for months without attention, on small boxes such as a Home
Assistant Green or a Raspberry Pi. It has to keep recording while Home Assistant restarts,
while a camera drops off the network, and while one part of the server has a problem.

Every other part of the server needs the same base:

- a way to start, watch and restart work per camera, so one bad camera never stops the others;
- one place that knows which cameras exist, how to reach them and what to record;
- configuration and state on disk that survive a restart and an update;
- one API that Home Assistant, the web interface and other clients all use, with a version
  so clients and server can tell whether they fit together;
- authentication, health reporting and logs that help a user or a developer find a problem.

None of this exists yet. This plan builds it before the media parts land on top of it.

## What exists

**In Home Assistant.** Home Assistant already talks to several dedicated servers in the same
way: the Matter Server, Z-Wave JS and Music Assistant. Each one is a separate process with its
own registry and a WebSocket API with JSON messages and a schema version that is checked on
connect. A thin Python client library on PyPI wraps the API, and the integration installs and
starts the app through the Home Assistant Supervisor. Camera Assistant follows the same pattern.

- Z-Wave JS Server: [zwave-js/zwave-js-server](https://github.com/zwave-js/zwave-js-server),
  client [home-assistant-libs/zwave-js-server-python](https://github.com/home-assistant-libs/zwave-js-server-python).
- Matter Server: [matter-js/matterjs-server](https://github.com/matter-js/matterjs-server).
- Music Assistant: [music-assistant/server](https://github.com/music-assistant/server), client
  [music-assistant/client](https://github.com/music-assistant/client).

Home Assistant apps can declare a `watchdog` address that the Home Assistant Supervisor polls, can exclude
folders from backups with `backup_exclude`, and can be shown as a panel through ingress. All
three are used below.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) has an HTTP API for Home
Assistant (`docs/ha-api.md` in its repository): a camera inventory that never returns the
stream address, a list of events with filters and paging, media served with HTTP range
requests, and a stream of server-sent events (event started, updated and ended; clip and
snapshot ready; camera status; disk alert). It also keeps a health check that tells a stalled
pipeline from a busy one. These are useful as prior art for the API shape and for health.

**In other projects.** [Moonfire NVR](https://github.com/scottlamb/moonfire-nvr) (Rust) has its
own sessions with CSRF protection and a JSON API; its README states that the API is not stable
before version 1.0. [Frigate](https://github.com/blakeblackshear/frigate) has users with roles
and a separate internal port without authentication.

## Concept

### Parts of the server

The server is one process. These names are used in every plan:

| Part | What it does | Plan |
|---|---|---|
| **Ingest** | Takes video into the server: RTSP client, HTTPS endpoint for Matter Push AV uploads, and later a WebRTC peer | RTSP ingest and the ring; the Matter plan for Push AV |
| **Ring** | Per camera stream, the last seconds of video in memory as fMP4 fragments | RTSP ingest and the ring |
| **Recorder** | Writes segment files, keeps the event list and the index | Recorder and playback |
| **Storage manager** | Locations, checks, background copies | Storage locations |
| **Live and playback** | HLS and MP4 for recordings now; WebRTC live view later | Recorder and playback (recordings) |
| **Event logic** | Turns triggers into events | Recorder and playback (the small Preview part) |
| **Detection modules** | Separate processes, compressed substream in, detections out | Later plans |
| **API** | WebSocket and HTTP, versioned schema | This plan |
| **Camera registry** | Cameras, stream sources, credentials, recording mode | This plan |
| **Task supervisor** | Starts, watches and restarts tasks and helper processes | This plan |

The task supervisor is part of the server. "Supervisor" alone, in every plan, means the Home
Assistant Supervisor.

### Process model and supervision

- One async runtime runs the whole server. Tokio is the expected choice (the language research
  assumes it); step 1 confirms it.
- The **server task tree** has a root with long-lived services (API, storage manager, recorder
  services such as retention) and one **camera task** per camera.
- A **camera task** owns everything that belongs to that camera: its ingest sessions, its ring
  buffers and its recorder writer. When the camera is changed or removed in the registry, the
  camera task is stopped and started again with the new settings. Nothing else in the server
  holds a camera's connections.
- Each task runs under the task supervisor. When a task ends with an error or panics, it
  logs it, reports it to health and starts it again after a backoff that grows with each failure
  and resets after a stable run. The backoff values are a constant in code, not a setting.
- **Watchdog.** A task reports progress through a small handle (`watchdog.feed()`). The
  task supervisor restarts a task that stops reporting for longer than its own limit. Ingest uses
  this for streams that stall without an error (see the ingest plan for the limits). The watchdog
  runs on a thread of its own, outside the async runtime, and reads progress counters, so it also
  notices when the runtime itself is blocked. In that case it ends the process, and the
  container's restart policy, or the app watchdog on Home Assistant OS, starts the server again.
- **Demand.** A camera task starts the ingest of a stream only when a consumer needs it. In
  the Preview the recorder is the only consumer, so a camera with recording off holds no
  connection to the camera at all. This keeps scarce camera connections free.
- **Helper processes.** Later parts run as separate processes: detection modules, and perhaps
  a thumbnail helper (see the recorder plan). The task supervisor starts them, watches them and
  restarts them with the same rules. A crash in a helper costs that helper's work, never a
  recording.
- **Shutdown.** On a stop signal the server stops accepting API calls, asks every camera task
  to finish, waits a short time for open segment files to be closed cleanly, and exits. A hard
  kill is safe as well: the recorder recovers open files at the next start.

### What fails, and what recovers it

| Failure | What is lost | What recovers it |
|---|---|---|
| A camera drops off the network | Footage during the outage | Ingest reconnects with a growing back-off (ingest plan). |
| A stream stalls without an error | Footage until the stall is noticed | The watchdog sees no progress; ingest tears the session down and reconnects (ingest plan). |
| A camera task panics | That camera, for a moment | The task supervisor starts it again after a back-off. Other cameras are untouched. |
| A detection module crashes | Detections | The server restarts it with a back-off. Recording and the camera's own events carry on (detection plan). |
| A detection module hangs | Detections | Its health reports a hung call or stops arriving, and the server kills and restarts it (detection plan). |
| The async runtime is blocked | Footage until the restart | The watchdog thread ends the process; the container's restart policy or the app watchdog starts it again. |
| A disk fills up | The oldest unprotected footage. New footage on that location, when nothing more can be deleted. | Emergency cleanup deletes the oldest unprotected footage. When only protected or open files are left, the recorder stops writing to that location and the rule for an unavailable primary applies. From the MVP, Home Assistant shows a repair (recorder plan, storage plan). |
| A copy location goes offline | Nothing, unless retention on the primary deletes a file before it is copied. That copy is then missed. | Its copy queue waits until it is back. Health reports a missed copy (storage plan). |
| Home Assistant restarts | Running recordings: nothing. Triggers that Home Assistant would have sent in that time, and the event clips they would have started. | The server does not depend on Home Assistant and keeps recording. The integration reconnects when Home Assistant is back. |
| The server crashes | Footage until the restart | The container's restart policy or the app watchdog starts it again. The recorder recovers open files, and the index is rebuilt from the files if it is lost (recorder plan). |

### Camera registry

The registry is the list of cameras. It is part of the configuration (below) and is changed
only through the API.

A camera record:

| Field | Meaning |
|---|---|
| `id` | Stable id, created by the server. Used in paths on disk, so it never changes. |
| `name` | Shown to people. Can change at any time. |
| `owner` | `home_assistant` or `local`. A camera has one owner. A camera handed over by Home Assistant is changed by Home Assistant; a camera added in the app's own interface is changed there. |
| `source` | `rtsp` or `matter`. |
| `streams` | For `rtsp`: a `main` stream and an optional `sub` stream, each with its address. For `matter`: the node id on the Matter fabric (the Matter plan fills in the rest). |
| `credentials` | User name and password for the camera, stored apart from the address (see secrets). |
| `recording.mode` | `off`, `events` or `continuous`. |
| `recording.streams` | Which streams are recorded: `main`, or `main` and `sub`. |
| `recording.pre_roll` | Seconds before an event to keep, within a maximum the ring sets (ingest plan). A Matter camera gets it as the pre-roll of its push transport. |
| `enabled` | False stops the camera task without removing the camera. |
| `external_ids` | Ids that other systems use for this camera, such as the Home Assistant device or entity id. Opaque to the server. |

Rules:

- A stream address never contains the password in the stored record. The registry strips
  credentials from an address on input and stores them in `credentials`.
- The API never returns a password or an address with a password in it, also not in errors or
  logs.
- A camera with `owner: home_assistant` refuses changes from other clients, except for the
  settings that the app's own interface manages in the Preview (recording mode, streams and
  pre-roll).
  The exact split is an open question.
- Each change emits an event (`camera_added`, `camera_updated`, `camera_removed`).

The ingest plan adds the probe that tests a stream address. The Matter plan adds what a
Matter camera needs.

### Configuration and state on disk

The server has one **data folder**. On Home Assistant OS this is the app's `/data`; in a plain
container it is a volume; in development it is a folder given on the command line.

| File | What it holds | Can it be rebuilt? |
|---|---|---|
| `config.json` | Cameras, storage locations and their rules, users, settings. Has a `schema_version`. | No. This is the source of truth. |
| `secrets.json` | Camera passwords, API token hashes, the key that signs media links. File mode 0600. | No. |
| `index.db` | The SQLite index of recordings and events (recorder plan). | Yes, from the files on the storage locations. |
| `logs/` | Optional log files (see logging). | Not needed. |

Rules:

- Every write of `config.json` and `secrets.json` goes to a temporary file in the same folder,
  is flushed, and is then renamed over the old file. A crash never leaves half a file.
- At start the server reads `schema_version` and runs migrations in order. A config from a newer
  server is refused with a clear log line; the server does not start on a config it does not
  understand.
- The index is kept apart from the config on purpose: it can be deleted and rebuilt, the config
  cannot. On Home Assistant OS, `index.db` is excluded from backups with `backup_exclude`; a
  restore rebuilds it.
- The index is always in the data folder, on local storage. SQLite should not run over a
  network file system, because file locking there is not reliable
  ([SQLite: use over a network](https://www.sqlite.org/useovernet.html)).

### The API

One API serves Home Assistant, the web interface and any other client. It follows the
pattern of the Matter Server, Z-Wave JS and Music Assistant.

**WebSocket** at `/ws`, JSON text frames, for commands and events.

- On connect the server sends a **server info** message before anything else:

  ```json
  {"server_id": "…", "server_version": "0.1.0", "schema_version": 1, "min_supported_schema_version": 1}
  ```

- The client then authenticates (see authentication) and sends commands:

  ```json
  {"message_id": "7", "command": "cameras/list", "args": {}}
  ```

- The server answers each command once:

  ```json
  {"message_id": "7", "result": […]}
  {"message_id": "7", "error_code": "not_found", "details": "No camera with id …"}
  ```

- Events are pushed after a client subscribes with `subscribe_events`:

  ```json
  {"event": "camera_updated", "data": {…}}
  ```

- Commands are named `<area>/<verb>`. Error codes are a fixed list of strings in the schema.
- `schema_version` is a whole number. It goes up when a command, a field or an event changes in
  a way an older client could not handle. Adding an optional field does not raise it. A client
  that needs a newer schema than the server offers tells the user to update the server; the
  Home Assistant integration raises a repair for that, as it does for the other servers.

**HTTP** on the same port, for things that are bytes rather than messages:

- media for playback (HLS playlists, segments, MP4, thumbnails; recorder plan);
- `/health` for the Home Assistant app watchdog (see health);
- the web interface's static files.

The Push AV upload endpoint for Matter cameras is a separate HTTPS listener with mutual TLS. It
is owned by the Matter plan, not by this API.

**Schema as code.** The message types are Rust types with serde. The schema is exported from
them (JSON Schema, or TypeScript types; candidates to evaluate are `schemars` and `ts-rs`), so
the Python client and the frontend do not copy types by hand. The export runs in CI and fails
when the committed schema is out of date.

**Who owns which commands.** This plan builds the framework and these commands:

| Area | Commands | Events |
|---|---|---|
| server | `server/info`, `subscribe_events` | |
| auth | `auth/login`, `auth/token_create`, `auth/token_revoke`, `auth/tokens_list` | |
| cameras | `cameras/list`, `cameras/get`, `cameras/add`, `cameras/update`, `cameras/remove` | `camera_added`, `camera_updated`, `camera_removed` |
| health | `health/get` | `health_changed` |
| logs | `logs/get` | |

The ingest plan adds `cameras/probe` and stream status. The recorder plan adds events,
recordings, playback and `record_clip`. The storage plan adds locations. Each plan defines its
own commands with the same framework and the same rules.

### Authentication

The API and the web interface always require authentication. The server never needs, and
never stores, credentials for Home Assistant.

- **Users.** The server has its own users with a password (hashed with Argon2 or a similar
  password hash; to confirm in step 6). The first user is created on first start in the web
  interface.
- **Tokens.** A user can create long-lived API tokens. The Home Assistant integration uses one.
  The server stores only a hash of each token.
- **Ingress.** On Home Assistant OS the web interface is shown as a panel through Supervisor
  ingress. Requests that come from the Supervisor's ingress proxy carry the Home Assistant
  user; the server trusts them only from the ingress source address, and maps them to an
  administrator for the Preview.
- **Media links.** A video player cannot always send a header. Playback links therefore carry
  a short-lived signature (HMAC over the path and an expiry time, with the key in
  `secrets.json`). The recorder plan says which paths use it.
- The WebSocket accepts the token in the first message after server info (`auth/login` with a
  token or a user and password). Nothing else is accepted before that.

### Health reporting

Health answers "is everything working, and if not, what is wrong" for a person and for Home
Assistant.

- Each part reports health items with a `key`, a `state` (`ok`, `warning`, `error`), a short
  message for people and optional numbers. Examples: a camera stream that is reconnecting, a
  location that is almost full, a helper process that keeps crashing.
- The health service keeps the current items in memory, sends `health_changed` when one
  changes, and answers `health/get`.
- Per camera: connected or not, the last time a frame arrived, the measured bitrate, the number
  of restarts. Ingest and the recorder fill these in.
- Per location: available, writable, free space, copy queue. The storage plan fills these in.
- Server: version, uptime, memory use of the process, CPU use.
- `/health` over HTTP answers 200 while the server's own loop runs, for the Home Assistant app
  watchdog. It does not fail because a camera is down; that is a health item, not a reason to
  restart the whole server.

### Logging

- Structured logs through the `tracing` crate (to confirm in step 8), written to standard
  output, where the Home Assistant Supervisor and Docker collect them.
- A log level setting (`info` by default) and a per-part override for debugging, changed at
  run time through the API.
- Every log line about a camera carries the camera id.
- A redaction layer removes passwords from stream addresses and other secrets before a line is
  written. This is tested.
- `logs/get` returns the last lines from a ring buffer in memory, so the web interface can show
  them and a user can attach them to an issue.

## Decisions

Settled by the project:

1. **Rust, one process.** The server is one Rust process; detection modules are separate
   processes. Reason: one language for the engine, the smallest footprint, and crash isolation
   for code from hardware vendors.
2. **Each camera in its own supervised task with a watchdog.** Reason: one camera that stalls
   or crashes must never stop recording on the others.
3. **Standalone engine.** The server never calls Home Assistant and holds no credentials for
   it. Cameras, modes and storage arrive through its API. Reason: Home Assistant is one client;
   the server keeps recording while Home Assistant restarts.
4. **WebSocket and HTTP with a versioned schema**, in the style of the Matter Server, Z-Wave JS
   and Music Assistant. Reason: the integration can follow a known pattern.
5. **Authentication is always on.** Reason: the server holds video of people's homes.
6. **A camera has one owner.** Reason: no duplicate cameras between Home Assistant and the app.
7. **The SQLite index can be rebuilt from the files.** Reason: losing the index must never mean
   losing recordings.

Proposed in this plan (to confirm in review):

8. **Config in a JSON file, index in SQLite, kept apart.** Reason: the config is small, must be
   backed up and must never be lost; the index is large and can be rebuilt.
9. **A camera with recording off holds no connection.** Reason: many cameras allow only one or
   two connections.
10. **The task supervisor restarts tasks; `/health` stays up while one camera is down.** Reason: a
    restart of the whole server for one camera would stop the others.

## Steps

1. **[server] Crates and process skeleton.** On the workspace of the project setup plan (which
   creates `ca-server` and the shared lints, `unsafe` forbidden included), add the library
   crates of this plan (`ca-supervisor`, `ca-registry`, `ca-api`) and empty crates for the parts
   that later plans fill (`ca-ingest`, `ca-ring`, `ca-recorder`, `ca-storage`, `ca-media`).
   Two rules for dependencies: no crate except `ca-server` depends on `ca-api`; the server's
   crates depend on detection only through `ca-detect-contract`, so FFmpeg and inference runtimes
   are never linked into the server. Command line and environment variables for the data folder
   and the port. Start, and a graceful stop on SIGTERM and SIGINT. Confirm the async runtime.
2. **[server] Task supervisor and watchdog.** The task tree, restart with backoff, the watchdog
   handle and its thread outside the async runtime, the stop order, and the same rules for helper
   processes. Tests with tasks that fail, panic and hang.
3. **[server] Data folder, config store and secrets.** `config.json` with schema version and
   migrations, `secrets.json` with mode 0600, atomic writes, refusal of a newer config.
4. **[server] Camera registry.** The camera record, validation, credentials split from the
   address, the owner rule, change events, and starting and stopping camera tasks on change.
5. **[server] API framework.** WebSocket with server info, commands, results, errors and
   events; HTTP router on the same port; the commands of this plan for server, cameras and
   logs; schema export checked in CI.
6. **[server] Authentication.** Users, password hashing, API tokens, ingress trust, signed media
   links, and the first-start flow.
7. **[server] Health reporting.** The health model and service, `health/get`,
   `health_changed`, process numbers, and `/health` for the app watchdog.
8. **[server] Logging.** Structured logs, levels at run time, redaction of secrets, and
   `logs/get`.
9. **[documentation] API reference.** A page generated from the schema export, with an example
   session (connect, authenticate, list cameras, subscribe).

## Order

Step 1 needs the workspace of the project setup plan. Steps 1 and 2 come first. Step 3 follows step 1. Step 4 needs steps 2 and 3. Step 5 needs
step 1 and can run next to steps 3 and 4; the camera commands land once step 4 is in. Step 6
needs step 5. Steps 7 and 8 need only step 1 and can start at any time. Step 9 needs step 5.

The other plans can start their own work as soon as steps 1 and 2 are in: ingest and the
ring need the task supervisor; the recorder and the storage manager need the config store and the
API framework to expose their commands.

## Open questions

1. **How does the Home Assistant integration get its API token?** Options: (a) the user creates
   a token in the app's interface and pastes it in the config flow; (b) the app hands a token to
   the integration through Home Assistant Supervisor's discovery when the integration installs it; (c) on Home
   Assistant OS, calls from the integration over the internal network are trusted. (b) gives
   the simplest setup; (c) weakens "authentication is always on".
2. **Which settings may the app's interface change on a camera that Home Assistant owns?** The
   Preview sets everything in the app's interface; the MVP moves setup to Home Assistant.
   Proposal: Home Assistant owns the source and credentials; the recording settings can be
   changed from both, and the last change wins.
3. **Are camera passwords encrypted at rest?** A key stored next to the file protects little
   against someone who can read the data folder, but it does keep passwords out of a copied
   config.
4. **Do users have roles in the Preview?** Proposal: one role (administrator) in the Preview;
   roles such as viewer come later.
5. **One port or two?** The web interface and the API on one port is simplest. A separate port
   for media could help when the interface goes through ingress (see the recorder plan,
   question on how a player reaches the media).

## Later, and what this must not block

- **Setup from Home Assistant (MVP).** Cameras handed over by their integration with trigger
  entities. The `owner` and `external_ids` fields are there for this.
- **Detection modules.** Helper processes under the same task supervisor.
- **Users and roles, per-camera access.** The auth model must allow roles without a change of
  the token format.
- **Other hosts.** The data folder and the API do not assume Home Assistant OS.
- **A full product without Home Assistant.** The registry and the API already work without it.
