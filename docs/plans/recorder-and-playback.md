# Recorder and playback: recordings that are cheap to write, safe to keep and easy to watch back

This is the technical companion of the epic "Recorder and playback" on the project board. It
is written so a developer or a coding agent can pick up a step and start.

Related plans:

- [Server foundation](server-foundation.md): task supervisor, camera registry, config, API,
  authentication and signed media links, health.
- [RTSP ingest and the ring](rtsp-ingest-and-ring.md): owns the **fragment format** (init
  segment and fMP4 fragment). This plan does not describe it again.
- [Storage locations](storage-locations.md): owns the **location interface**, the checks of a
  location, background copies and the choice of the primary location.

This plan owns the **segment file**, its layout on a location, the **index**, recording modes,
events in the Preview, retention, and playback of recordings.

## Problem

A household wants three things from recordings: that they are there when something happened,
that they do not fill the disk, and that they can watch them back on any device.

- Recording must be cheap, so it runs on a Home Assistant Green or a Raspberry Pi. That means
  storing the video as the camera sends it, never converting it.
- A recording that starts when something happens is too late. It needs the seconds before the
  trigger (pre-roll).
- Disks fill up. Old footage has to go by itself, by age and by size, and a full disk must never
  stop recording or break the system it runs on. A clip that matters must be kept.
- An index makes recordings fast to find, but an index can get lost or damaged. The recordings
  on disk must be enough to rebuild it.
- Home Assistant today proxies or even remuxes recorded video inside its own process for
  Frigate, UniFi Protect and Reolink. Camera Assistant serves its recordings itself, in a form
  that browsers and phones can play.

## What exists

**In Home Assistant.** `camera.record` writes an MP4 from the `stream` integration's buffer,
for at most 3600 seconds, one recording at a time per camera, inside Home Assistant's own
process. The Media browser plays recordings through `async_resolve_media`, which in practice
returns an address inside Home Assistant that proxies the bytes. The Reolink integration
downloads a recording, remuxes it with FFmpeg into fMP4 and an HLS playlist in a temporary
folder, and serves that ([core#192887](https://github.com/home-assistant/core/pull/192887)).
Its description says this fixed playback in Safari on iOS, which needs HTTP range requests,
and fixed seeking in Chrome and Edge.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) records event clips. Each clip is
a file named `{event_id}_{camera}_{timestamp}.mp4`, starting at the first keyframe, cut from
the in-memory ring, and indexed in SQLite; the index can be rebuilt from the file names. Its
retention keeps clips per label and per camera for a number of days, and deletes the oldest
events when free space drops below `free_space_min_mb`. A still is cut from each finished clip.
Media is served with HTTP range requests (`docs/ha-api.md`).

**In other projects.**

- [Moonfire NVR](https://github.com/scottlamb/moonfire-nvr) stores video frames in files on a
  large disk and all other data in SQLite on flash, and builds `.mp4` files for any time range
  on the fly, served with HTTP range requests (`ref/api.md` and `design/schema.md` in its
  repository).
- [Frigate](https://github.com/blakeblackshear/frigate) records continuous segments of about
  10 seconds and serves HLS for any time range through nginx-vod-module. Since version 0.17
  its retention is tiered by kind of footage (continuous, motion, alerts, detections).
- [Viseron](https://github.com/roflcoopter/viseron) serves HLS by time and moves segments
  between storage tiers by age.
- HLS is specified in [RFC 8216](https://datatracker.ietf.org/doc/html/rfc8216). It supports
  fMP4 segments (`EXT-X-MAP` for the init segment), byte ranges within one file
  (`EXT-X-BYTERANGE`), wall-clock time (`EXT-X-PROGRAM-DATE-TIME`) and breaks
  (`EXT-X-DISCONTINUITY`).

## Concept

### The recorder in the server

Per camera, the camera task (server foundation plan) holds one **recorder writer** per recorded
stream. The writer subscribes to that stream's ring and writes segment files to the primary
location (storage plan). Long-lived **recorder services** run once in the server: the index, retention,
emergency cleanup, the maker of event stills and the playback endpoints.

### The segment file

A segment file is the on-disk unit. It holds fragments in the fragment format of the ingest
plan, unchanged.

- One file per **track** (video or audio) of a **stream** (`main` or `sub`) per stretch of
  time. The video file and the audio file of the same stretch start and end on the same
  fragment pair, so they always line up. Together they are one **segment**.
- A file starts with the track's **init segment** and is followed by consecutive
  **fragments** with the same init id. It is a valid fragmented MP4 file that common players
  open.
- A segment is **closed** (both files at once) when:
  - it reaches the target length (open question 1; proposal: about one minute);
  - the init id changes (a codec change or a gap; see the ingest plan);
  - recording of the stream stops (event ended in `events` mode, mode changed, camera stopped);
  - the writer lost fragments because it fell behind (the ring's lagged signal).
- Because every video fragment starts with a keyframe, every segment starts with a keyframe and
  can be played on its own.
- While a file is open, the writer appends one fragment at a time. It flushes after each
  fragment and syncs to disk when the file is closed and at a fixed interval while it is open
  (open question 2). After a crash the file is readable up to the last whole fragment.

### Layout on a location

All paths are relative to the root of a location (storage plan). Times in paths are UTC.

```
recordings/<camera_id>/<stream>/<YYYY>/<MM>/<DD>/<HH>/<start_ms>.video.mp4
recordings/<camera_id>/<stream>/<YYYY>/<MM>/<DD>/<HH>/<start_ms>.audio.mp4
events/<camera_id>/<YYYY>/<MM>/<DD>/<event_id>.json
events/<camera_id>/<YYYY>/<MM>/<DD>/<event_id>.jpg
```

- `<camera_id>` is the stable id from the registry, never the name.
- `<start_ms>` is the wall-clock start of the segment's first fragment pair, in milliseconds
  since the Unix epoch. The two files of a segment share it.
- A stream without audio has only the `.video.mp4` file.
- The folder per hour keeps folders small on network shares.
- `<event_id>.json` is the **event file**: id, camera id, start, end (empty while open),
  source, trigger details, labels, protected flag. It is written when the event opens and
  replaced (written to a temporary name, then renamed) when it changes.
- `<event_id>.jpg` is the event's still, once made.

**An event clip is not a separate file.** It is the span of segment files that covers the
event, plus its event file. In `events` mode, those are the only segment files that are
written. In `continuous` mode, the event file points into continuous footage that is there
anyway. One format, one playback path, no copy of video for an event.

### The index

The index is the SQLite file `index.db` in the server's data folder (server foundation plan).
It makes lookups fast. Everything in it can be rebuilt from the files.

| Table | Rows | Main columns |
|---|---|---|
| `segments` | One per segment (its video file and audio file) | id, camera id, stream, start, end, bytes per track, init size per track, codecs, gap before, state (`writing`, `closed`), fragment table |
| `placements` | One per copy of a segment or an event on a location | segment or event id, location id, path, state |
| `events` | One per event | id, camera id, start, end, source, trigger details, labels, protected, still state |
| `meta` | Key and value | schema version, last rebuild |

- The **fragment table** is a compact blob per segment: for each fragment pair its start
  (relative to the segment start), its duration, and per track the byte offset and size. With it, the server writes an HLS playlist
  without reading any video file.
- `placements` is shared with the storage manager, which adds and removes rows when it copies
  or deletes. Its rules for copies are in the storage plan.
- **Rebuild**: walk `recordings/` and `events/` on every available location. For a segment file,
  read the camera, stream and start from its path, then read only the box headers (`moof`,
  `tfdt`, `prft`, sizes) to fill in the fragment table and the end. For an event, read the event
  file. A file found on two locations becomes one row in `segments` or `events` and two rows in
  `placements`. The two files of a segment are found by their shared start. A rebuild runs when `index.db` is missing or its schema is unknown, and on
  request through the API.
- **Recovery at start**: every segment in state `writing` is checked: a trailing partial
  fragment is cut off, the end is set, and the state becomes `closed`. Every event without an
  end gets the end of its last footage.

### Recording modes

The registry's `recording.mode` per camera:

| Mode | Ingest | What is written |
|---|---|---|
| `off` | No connection (no demand from the recorder) | Nothing |
| `events` | Stream kept open so the ring has pre-roll | Segment files only while an event is open, from pre-roll to post-roll |
| `continuous` | Stream kept open | Segment files all the time; events mark time ranges in them |

- In `events` mode, when an event opens, the writer subscribes to the ring with
  `Back(pre_roll)` and writes until the event closes plus the post-roll. A new event that opens
  before the post-roll ends continues the same run of files. Pre-roll comes from the camera's
  registry field `recording.pre_roll`, which the web interface shows as a setting; post-roll is
  a fixed value in the Preview (open question 3).
- An event has a maximum length; the event logic closes it and opens a new one after it, so one
  stuck trigger cannot record forever.
- A change of mode takes effect at once; an open file is closed cleanly.

### Events in the Preview

The full event logic (combining camera triggers, motion and detection) is a later plan. The
Preview needs only this small part, which lives in the recorder:

- `open_event(camera, source, details) -> event_id`, `update_event(id, details)`,
  `close_event(id)`. These are internal functions, the same for every source.
- Sources in the Preview:
  - `record_clip` from the API (below);
  - a Matter camera's Push AV session with a motion or command trigger: one event with its clip;
    the Matter plan opens and closes it with the times the camera reports. A session with the
    continuous trigger is continuous footage (recording spans), not an event; events on top of
    it come from the camera's own zone and motion events.
- RTSP cameras in `events` mode record only on `record_clip` in the Preview, because no other
  trigger reaches the server yet. RTSP cameras in `continuous` mode record all the time.
- Events are pushed over the API as `event_started`, `event_updated` and `event_ended`.

### `record_clip`

API command `record_clip`, also the action that the Home Assistant integration offers.

| Argument | Meaning |
|---|---|
| `camera_id` | The camera. |
| `duration` | Seconds to record after now. Capped at a maximum. |
| `pre_roll` | Optional. Seconds before now to include. Default: the camera's `recording.pre_roll`. Capped at what the ring holds. |
| `label` | Optional text that the event carries, for example the automation's name. |

Behaviour:

- `events` mode: opens an event with source `request`, starting with the pre-roll, and closes
  it after `duration`.
- `continuous` mode: opens an event over the footage that is being written anyway.
- `off` mode: refused with error `recording_off` (open question 4).
- A second call for the same camera while a requested event is open extends it.
- For a Matter camera, the Matter plan also asks the camera to upload (a manual trigger of its
  push transport). The recorder does not know how; it calls the Matter side through one
  function.
- Answers with the event id.

### Uploaded segments (Matter Push AV)

Matter cameras do not go through the ring in the Preview. The Push AV endpoint (Matter plan)
receives the camera's CMAF uploads, one folder per track, and hands them to the recorder:

- `intake_init(camera, session, track, init)` when a session's init segment arrives: this
  starts a new segment for that track, as a new init id does for an RTSP stream;
- `intake_segment(camera, session, track, number, bytes, wall_clock_start)` for each uploaded
  segment: its `moof` and `mdat` boxes are appended unchanged to the track's open segment file
  (a Matter camera's recording stream is stored as stream `main`); nothing is converted;
- the end of the session (Matter plan) closes the segment files.

Uploaded segments may carry no `prft` box. The recorder then takes the wall clock from the
upload time and the session events, and the file name and `tfdt` keep the files
self-describing. Whether a camera's video and audio segments line up in time is open question 1
of the ingest plan. From there on, a Matter recording is the same as an RTSP recording: same
layout, same index, same retention, same playback.

### Retention

Retention runs in the recorder because only the recorder knows which footage belongs to
which event and which is protected. The storage manager supplies the free space and performs
the deletes.

Per location, the Preview has two limits:

- `keep_days`: footage older than this is deleted;
- `max_gb`: when the footage on the location is larger than this, the oldest is deleted
  first.

Rules:

- Retention works per segment file, oldest first, and runs every few minutes and after each
  closed file.
- It never deletes a file that is open, or that overlaps a protected event.
- An event row and its event file are deleted when no footage of it remains on any location,
  unless it is protected.
- A location where protected footage alone is above `max_gb` gets a health warning; protected
  footage is not deleted.

### Emergency cleanup

When the storage manager reports a location as `low_space` (its threshold is in the
storage plan), the recorder deletes the oldest unprotected footage on that location, whatever
its age, until the location is above the threshold plus a margin. Each emergency cleanup is
logged and reported to health, and Home Assistant shows it (a repair in the MVP).

When nothing more can be deleted (everything left is protected or open), the location is
**full**: the recorder stops writing to it, and the storage manager's rule for an unavailable
primary applies (storage plan). Recording never fills a disk to the last byte. This matters
most on Home Assistant OS, where the default location shares a disk with Home Assistant
itself.

### Protecting a clip

`events/protect` with `event_id` and `protected` (true or false).

- Sets the flag in the index and in the event file, so a rebuild keeps it.
- Protected footage is skipped by retention by age, by size, and by emergency cleanup.
- It counts towards `max_gb`.

### Playback

Playback serves what is on disk, as it is. Every media address carries a signed expiry (server
foundation plan, "Media links"), so a player can fetch it without a header.

**Playback handle.** `playback/get` with an event id, or a camera, a stream and a time range,
answers with the addresses a client can use and when they expire:

```json
{"hls": "/api/playback/….m3u8?exp=…&sig=…", "mp4": "/api/download/….mp4?exp=…&sig=…", "expires": …}
```

The Home Assistant integration turns this into a playback handle for the Media browser (its
own plan).

**HLS.** The server writes playlists from the index, for an event or for a time range: a
multivariant playlist that names the video media playlist and, when the stream has audio, an
audio rendition (`EXT-X-MEDIA` with `TYPE=AUDIO`) with its own media playlist. Each media
playlist covers one track:

- `EXT-X-PLAYLIST-TYPE:VOD` and `EXT-X-ENDLIST` for footage that is finished;
  `EXT-X-PLAYLIST-TYPE:EVENT` without `EXT-X-ENDLIST` for an event that is still open, so a
  player can follow it.
- For each segment file of the track: `EXT-X-MAP` with the file's address and the byte range
  of its init segment, and `EXT-X-PROGRAM-DATE-TIME` with its wall-clock start.
- For each fragment of the file: `EXTINF` with its duration and `EXT-X-BYTERANGE` with its size
  and offset from the fragment table. So each fragment is one HLS segment, a player can seek to
  any keyframe, and the video and audio playlists have the same segment boundaries.
- `EXT-X-DISCONTINUITY` between files whose init ids differ or with a gap between them.
- `EXT-X-TARGETDURATION` from the longest fragment, and the version number that these tags need
  according to RFC 8216.
- Files are served by a media endpoint with HTTP range requests. It reads through the location
  interface (`read a range`), from the primary location when the file is there and from a copy
  when it is not.

**MP4 for download.** One MP4 file for an event or a time range, with video and audio in it:
one init segment with both tracks, then per fragment pair one `moof` with a `traf` per track and
the samples of both, with the decode times rewritten so the file has one timeline. This moves
bytes and rewrites box headers; it never touches the video itself. A range that spans
two init ids is cut at the first change (rare: a codec change). Fragmented MP4 is chosen for
the Preview because it needs no second pass; whether a progressive MP4 (`moov` first) is
needed for phones and messaging apps is open question 5.

**A still per event.** One JPEG per event, from the first keyframe at or after the trigger
time, from the substream when it is recorded, else from the main stream. It is made when it is
first asked for and stored as `<event_id>.jpg`. Making it means decoding one frame, which the
server otherwise never does (open question 6 on where that runs). The Matter plan may supply a
still from the camera itself instead.

**Spans.** `recordings/spans` for a camera and a time range answers with the stretches of
continuous footage, merged across files. The Media browser uses it now; the timeline uses it
later.

H.265 recordings play where the browser or the device can decode H.265. The web interface
shows a clear message where it cannot. Converting video for a viewer is not part of the
Preview.

### API of this plan

| Commands | Events |
|---|---|
| `events/list` (cameras, time range, source, protected; paged with a cursor), `events/get`, `events/protect` | `event_started`, `event_updated`, `event_ended` |
| `recordings/spans`, `recordings/rebuild_index` | `recording_state` (per camera: recording or not, and why not) |
| `playback/get`, `record_clip` | |

HTTP: the HLS playlists, the media endpoint for segment files, the MP4 download and the still.

## Decisions

Settled by the project:

1. **No transcoding to record.** Reason: recording runs on every box.
2. **Recording modes off, on events, continuous, with pre-roll and `record_clip`.** Reason:
   these are the Preview's needs for Matter and RTSP cameras.
3. **An index in SQLite that can be rebuilt from the files.** Reason: losing the index must
   never mean losing recordings.
4. **HLS over the stored CMAF segments, MP4 for download.** Reason: nothing is converted, and
   seeking works as in any video player.
5. **The oldest footage goes first when a disk fills up.** Reason: recording keeps working and
   the system around it keeps working.

Proposed in this plan (to confirm in review):

6. **An event clip is a span of segment files plus an event file, not a separate file.**
   Reason: one format, one playback path, and no second copy of the video in `continuous` mode.
7. **Segment files are self-describing** (path, `prft`, box headers, event files). Reason: this
   is what makes the rebuild possible.
8. **One HLS segment per fragment, by byte range.** Reason: few files on disk, seeking to every
   keyframe, and no work to cut files for playback.
9. **Retention lives in the recorder; free space and deletes in the storage manager.** Reason:
   only the recorder knows events and protection; only the storage manager knows the locations.
10. **The still is made on demand.** Reason: no decoding work for events nobody looks at.

## Steps

1. **[server] Segment file writer.** Subscribe to the ring, write files in the layout above
   through the location interface, close them by the rules above, flush and sync. Tests with
   fragments from the sample streams of the ingest plan.
2. **[server] The index.** Schema with migrations, writes from the writer, queries for spans
   and for fragments of a time range.
3. **[server] Recovery and rebuild.** Repair of open files at start, and the full rebuild from
   the files on all locations. Tests that delete the index, cut files in half and put the same
   file on two locations.
4. **[server] Recording modes and the event lifecycle.** Off, events and continuous; demand on
   the ring; pre-roll, post-roll and the maximum event length; `open_event`, `update_event`,
   `close_event`; the event files; `recording_state`.
5. **[server] `record_clip`.** The command with its behaviour per mode, and the hook to the
   Matter side.
6. **[server] Intake of uploaded segments.** `intake_init` and `intake_segment` for the Matter plan, with a test
   that feeds segments in the layout that Matter cameras upload.
7. **[server] Retention by age and size.** Per location limits, the never-delete rules, removal
   of event rows, the warning for protected footage. Tests on a fake location with a clock.
8. **[server] Emergency cleanup.** React to "low on space" from the storage manager, delete
   until above the margin, mark the location full when nothing can go, health and logs.
9. **[server] Protecting a clip (MVP).** `events/protect`, the flag in the index and the event file,
   and the checks in retention and emergency cleanup.
10. **[server] HLS playback.** Playlists for events and time ranges, open events as `EVENT`
    playlists, the media endpoint with range requests and signed links. Tested in Safari and,
    through hls.js, in Chrome and Firefox.
11. **[server] MP4 download.** One file per event or time range with one timeline, served as a
    stream with a known length.
12. **[server] Stills per event.** The frame choice, the decode (per open question 6), JPEG,
    storage next to the event file, and a placeholder while it is made.
13. **[server] Events and recordings API.** `events/list`, `events/get`, `recordings/spans`,
    `playback/get`, `recordings/rebuild_index`, and the events, with paging.

## Order

Step 1 needs the ring (ingest plan, steps 4 and 7) and the file-system location (storage plan,
step 1). Step 2 runs next to step 1. Step 3 needs steps 1 and 2. Step 4 needs step 1 and the
camera task of the server foundation plan. Steps 5 and 6 need step 4. Step 7 needs steps 2 and
4; step 8 needs step 7 and the free-space check of the storage plan. Step 9 needs step 4 and is
then small. Steps 10 and 11 need step 2. Step 12 needs step 4 and the answer to open question
6. Step 13 can grow along with steps 2, 4 and 10.

## Open questions

1. **How long is a segment file?** Longer files mean fewer files on a network share; shorter
   files mean retention can delete in finer steps and less is lost when a file is damaged.
   Proposal: about one minute, always ending at a fragment boundary.
2. **How often to sync to disk?** Syncing after every fragment is safe but wears out eMMC and is
   slow on a network share. Proposal: sync when a file closes and every few seconds while it is
   open; measure on a Green.
3. **Defaults for pre-roll and post-roll?** Pre-roll is a per-camera setting (the web interface
   plan offers it); its default and maximum are still to be set, and the maximum decides the
   ring's memory. Proposal: a few seconds by default, and a post-roll of a few seconds that is
   not a setting. A Matter camera gets the pre-roll in its push transport.
4. **Does `record_clip` work when recording is off?** "Off" may mean "never record this
   camera", for privacy. Proposal: refuse, and say so in the error.
5. **Is a fragmented MP4 good enough for download and, in the MVP, for clips in notifications?**
   If phones or messaging apps play it badly, a progressive MP4 with `moov` first is built from
   the fragment tables; that costs a pass over the data but no transcoding.
6. **Where is the frame for a still decoded?** Options: (a) a short-lived FFmpeg process that
   gets one fragment and returns a JPEG; (b) the decoder of the detection engine that comes
   from Cairn, which decodes through FFmpeg as well; (c) a decoder library inside the server
   (H.264 only for some libraries). (a) keeps codec code out of the server process and works
   now; (b) fits once detection modules exist. The licence of the FFmpeg build has to be
   checked for every option.
7. **How does a player reach the media endpoint?** Directly on the server's port with a signed
   link is the shortest path. Through Supervisor ingress, video passes through Home
   Assistant's process. The integration plan decides what the Media browser uses; this plan
   only needs to serve both.
8. **What happens to footage of a removed camera?** Proposal: it stays until retention removes
   it, and the web interface lists it under the removed camera's name.

## Later, and what this must not block

- **Full storage rules** (after the MVP): what goes where and for how long, per kind of footage
  and per label. Retention per location is the first case of those rules.
- **The timeline with scrubbing** (after the MVP): spans, small stills along the time line, and
  a still at a past time. The fragment table and spans are the base.
- **Clips in notifications** (MVP): the MP4 download and the still.
- **Event logic with detection** (MVP): opens and closes events through the same functions.
- **WebRTC playback of recordings** (after the MVP), for clients that cannot reach the server
  directly. It has no plan yet; the fragment table must be enough to feed it.
- **Delete and export from the interface.**
- **Transcoded video for a viewer that cannot play the stream** (after the MVP): a separate
  path; the stored files stay as they are.
