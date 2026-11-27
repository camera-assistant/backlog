# RTSP ingest and the ring: each camera stream taken once and kept in memory for everyone

This is the technical companion of the epic "RTSP ingest and the ring" on the project board. It
is written so a developer or a coding agent can pick up a step and start.

Related plans:

- [Server foundation](server-foundation.md): the task supervisor, the camera registry, the API and
  health that this plan builds on.
- [Recorder and playback](recorder-and-playback.md): the first consumer of the ring; it owns
  the segment files on disk.
- [Storage locations](storage-locations.md).

This plan owns the **fragment format**: the init segment and the fMP4 fragment that every
part of the server passes around. The recorder plan owns the **segment file**, which is built
from these fragments. No other plan describes either format again.

## Problem

Most cameras in homes today are RTSP cameras. They send a main stream (full resolution) and
often a substream (small, for previews and detection). Video is H.264 or H.265; audio, when
there is any, is AAC, Opus or G.711.

Many of these cameras allow only one or two connections at a time. Each viewer, recorder or
detector that opens its own connection takes one, and the next one fails. Cheap cameras also
stall without closing the connection, send odd timestamps, and drop off the network.

Camera Assistant takes each stream once and passes the compressed video to everything that
needs it, without converting it. It also keeps the last seconds of every stream in memory, so
a recording can start a few seconds before the moment that triggered it (pre-roll), and so a
viewer can start without waiting for the next keyframe.

## What exists

**In Home Assistant.** Live view runs through the bundled go2rtc, which opens its own
connection to the camera. The `stream` integration decodes and muxes inside Home Assistant's
own process; `camera.record` writes from its buffer and is limited to 3600 seconds, one
recording at a time per camera. In the Preview, Home Assistant's live path stays as it is, so
go2rtc keeps its own connection to the camera next to Camera Assistant's.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) keeps a per-camera ring of fMP4
(CMAF) fragments in memory, `pre_window_seconds` long, with an atomic "drain and subscribe": a
new consumer gets the buffered fragments and then the live ones, with no gap and no duplicate
(`docs/architecture.md`, section "The ring buffer"). It detects on the substream and records
from the main stream. Its ingest is an FFmpeg process as an RTSP to MPEG-TS bridge by default,
with a native RTSP client per camera as an option. These ideas carry over; the code is Elixir
and is not reused.

**In other projects.**

- [Retina](https://github.com/scottlamb/retina) is an RTSP client library in Rust, in
  production use in Moonfire NVR. Its README lists: digest and basic authentication; RTP over
  TCP (interleaved) and over UDP (experimental, out-of-order packets are dropped because there
  is no reorder buffer); depacketizing of H.264, H.265, AAC, Opus and G.711; receiving RTCP
  sender reports. It also lists what is missing, among them "multiple SPS/PPS" and "periodic
  intra refresh" for H.264, a stable API, and good test coverage. It has one main maintainer.
- [Moonfire NVR](https://github.com/scottlamb/moonfire-nvr) records without decoding and,
  according to its README, handles six 1080p streams at 30 frames a second on a Raspberry Pi 2
  at less than 10% CPU. Its design notes (`design/time.md`) describe how it handles the drift
  between a camera's clock and the server's clock.
- [MediaMTX](https://github.com/bluenviron/mediamtx) is a Go media server with a widely used
  RTSP stack (gortsplib). It is useful here as a test source: FFmpeg publishes looped sample
  files to it, and the server under test reads them as RTSP streams.
- [go2rtc](https://github.com/AlexxIT/go2rtc) knows the quirks and address formats of many
  consumer cameras. Camera Assistant reuses that knowledge as a reference, not its code.

## Concept

### Pieces

Per camera, the camera task (server foundation plan) holds, for each stream it opens:

```
RTSP session ──► depacketizer ──► fragmenter ──► ring ──► consumers
 (one per stream)  (access units)   (init + fMP4)  (memory)  (recorder now; live view
                                                            and detection later)
```

- **RTSP session**: one connection per stream (`main`, `sub`), RTP over TCP by default.
- **Depacketizer**: turns RTP packets into whole frames (access units) with their timestamps.
- **Fragmenter**: turns frames into init segments and fMP4 fragments, one track each.
- **Ring**: keeps recent fragments and hands them to consumers.

The ring's input does not know about RTSP. Later, the Matter WebRTC path and a near-live
feed from Push AV can put fragments into the same ring.

### The fragment format

This is the one description of the in-memory format. The recorder writes these bytes to disk
unchanged.

**Track set.** Each stream of a camera has one video track and zero or one audio track.

| Track | Codecs | Sample entry | Timescale |
|---|---|---|---|
| Video | H.264, H.265 | `avc1` with `avcC`; `hvc1` with `hvcC` (parameter sets in the sample entry) | 90000 |
| Audio | AAC | `mp4a` with `esds` | the sample rate |
| Audio | Opus | `Opus` with `dOps` | 48000 |
| Audio | G.711 (A-law, µ-law) | converted before the fragmenter (open question 2) | — |

`hvc1` (parameter sets in the sample entry, not in the stream) is chosen because Apple's
players require it for H.265 in fMP4. New parameter sets therefore always mean a new init id.

**One track per fragment.** Video and audio are kept as separate CMAF tracks: each has its own
init segment and its own fragments. This is how CMAF defines a track, and it is how Matter
cameras upload (one folder per track), so their segments can be stored as they arrive (see the
Matter plan). The audio fragments are cut at the same moments as the video fragments, so the
two tracks always line up.

**Init segment.** `ftyp` + `moov`, with one `trak`. Each stream has an **init id** (a counter
per stream) that covers the init segments of all its tracks. A new init id is made when the
codec, the resolution or the parameter sets (SPS, PPS, VPS) change, or when an audio track
appears or goes away.

**Fragment.** `moof` + `mdat`, one track.

- A video fragment always **starts with a keyframe** (an IDR frame for H.264, an IRAP frame for
  H.265) and runs to the frame before the next keyframe. So a fragment is one group of
  pictures. Any fragment can be the first one a consumer gets.
- The audio fragment of the same span holds the audio samples from the start of that video
  fragment to the start of the next one.
- A video fragment and its audio fragment together are a **fragment pair**. The ring, the
  recorder and playback work in fragment pairs, so a cut always falls on the same moment in both
  tracks. A stream without audio has pairs with only a video fragment.
- `mfhd` carries a sequence number that goes up by one per fragment pair of the stream; both
  fragments of a pair carry the same number.
- `tfdt` carries the decode time of the fragment's first sample, in the track's timescale,
  counted from the start of the init id.
- A `prft` box (producer reference time) carries the wall-clock time of the first sample.
  Together with `tfdt` it makes every fragment say when it was recorded, which the recorder
  needs to rebuild its index from files.
- Composition time offsets are written when a camera sends B-frames.

**Fragment metadata.** Next to the bytes, the ring keeps for each fragment pair: camera id,
stream, init id, sequence number, wall-clock start, duration, byte size per track, and whether
a gap came before it (a lost connection or a timestamp jump).

**Gaps.** After a reconnect or a timestamp jump the fragmenter starts a new init id, even if
the codec settings are the same. A consumer then sees a new init segment and knows that time
did not run on without a break.

**Long groups of pictures.** Some cameras can be set to send a keyframe only every 10 seconds or more. A
fragment is then that long. This costs memory, not correctness. A live viewer with low delay
needs smaller pieces; that is for the live view plan (see "Later").

### Timing

- The video timestamp comes from the RTP timestamp (90 kHz for video).
- The wall clock is anchored at the server's clock when the first frame of a session arrives
  and then runs on with the RTP timestamps. When the two clocks drift apart by more than a
  limit, the fragmenter re-anchors and marks a gap. The limit and the method are open question
  4; Moonfire's `design/time.md` is the reference to read first.
- The camera's own clock is not trusted for the wall clock in the Preview.
- Audio and video are aligned by their RTP timestamps through the RTCP sender reports where the
  camera sends them, and by arrival time where it does not.

### The ring

One ring per opened stream (so a camera with main and sub stream has two).

- It keeps whole fragment pairs, newest last, plus the init segments of each init id still in
  it.
- **Size.** It keeps at least the configured pre-roll back to a keyframe: the oldest pair it
  keeps starts at or before "now minus pre-roll". Because a pair starts with a keyframe,
  pre-roll always starts with a keyframe. A hard memory limit per ring guards against a camera
  with a very high bitrate or a very long group of pictures; when it is reached, the oldest
  pair goes first, and health reports it.
- **Memory.** Roughly bitrate × seconds kept. For example, 8 cameras at 4 Mbit/s with 10
  seconds kept need about 40 MB. The substream adds little.
- **Subscribe.** A consumer calls `subscribe(start)`, where `start` is `Now` (from the next
  pair) or `Back(duration)` (from the newest pair that starts at or before now minus the
  duration). It gets, in one atomic step, the init segments, the buffered pairs from `start`,
  and a receiver for every pair after them. No gap, no duplicate.
- **Consumers never slow ingest down.** Each consumer has a bounded queue. When a consumer's
  queue is full:
  - a live viewer (later) skips ahead to the next pair;
  - the recorder gets a "lagged" signal with the number of pairs it lost; it closes its
    current segment files at the gap, reports it to health, and continues from the next pair.
    The recorder plan handles this.
- Fragments are shared, not copied (reference-counted byte buffers), so ten consumers cost
  the same memory as one.

### Session lifecycle, reconnects and stalls

- The camera task starts a stream's session when a consumer needs it (server foundation plan,
  "Demand") and stops it a short time after the last consumer leaves.
- **Connect**: OPTIONS, DESCRIBE, SETUP per track, PLAY, with digest or basic authentication.
  The session keeps itself alive within the timeout the camera gives (keepalive requests).
- **Reconnect** after any error with a backoff that grows from about one second to about thirty
  seconds and resets after a stable minute. The exact values are constants in code.
- **Stall detection** (the watchdog of the server foundation plan): a stream counts as stalled
  when no frame has arrived for longer than a limit that follows the stream's own rhythm (for
  example three times the measured keyframe interval, with a floor and a ceiling). A stalled
  session is torn down and reconnected, even though the TCP connection is still open.
- **Timestamp problems**: a timestamp that goes backwards or jumps far ahead starts a new init
  id (a gap) instead of passing on a broken timeline.
- **Health**: per stream: state (`connecting`, `streaming`, `reconnecting`, `refused`), codec,
  resolution, measured bitrate and frame rate, keyframe interval, last frame time, reconnects
  in the last hour, and the last error in words a person understands ("wrong password", "the
  camera refused another connection").

### Cameras that allow only one or two connections

Camera Assistant opens at most one connection per stream it uses. What it can still not
control is who else connects to the camera: in the Preview, Home Assistant's own live view
(go2rtc) keeps its connection.

- **Open only what is used.** The recorder needs the main stream. The substream is opened only
  when the camera is set to record it (registry field `recording.streams`), and later when
  detection needs it. In the Preview nothing else asks for it.
- **Recognise a refusal.** A camera that is full can answer in different ways: an RTSP error
  (for example 453 "Not Enough Bandwidth" or 503), a closed connection right after SETUP or PLAY, or
  a stream that never sends a frame. The session classifies these as `refused` and reports
  "the camera refused another connection; it may allow only a few at a time".
- **Give way in a fixed order.** When the substream is refused while the main stream works,
  the substream is given up and retried at a slow pace; the main stream is never given up for
  the substream.
- **Say it once at setup.** The probe (below) opens the streams the way the recorder will and
  warns when a second connection fails.
- **What solves it for good** comes later: when Camera Assistant serves live view (MVP), Home
  Assistant no longer needs its own connection. Until then, a restream from Camera Assistant
  for go2rtc is an option (open question 5).

### Probe

`cameras/probe` (an API command of this plan) takes a stream address and credentials, opens
the stream once and answers with what it found: codecs, resolution, frame rate, keyframe
interval, audio, and warnings. Warnings include: H.265 on every stream (it does not play in
every browser), no keyframe within a few seconds, G.711 audio (converted), a refused second
connection. The web interface calls it when a camera is added, and the user sees the result
before saving.

### Libraries worth evaluating

None of these is settled.

- **RTSP client**: Retina first, because it is Rust, async with tokio, used in production by
  an NVR, and covers the codecs above. Points to check against real cameras: H.264 cameras
  that change SPS/PPS during a stream (not supported according to its README), periodic intra
  refresh (not supported), and UDP (experimental). Fallback options: an FFmpeg process as a
  bridge for odd cameras (as Cairn does by default), or contributing what is missing to Retina.
- **fMP4 writing**: a small writer of our own for the boxes listed above, or an existing MP4
  crate. The format is small and fixed, so a writer of our own is realistic.
- **G.711 conversion**: decoding G.711 is a lookup table and needs no library. Encoding needs an
  AAC or Opus encoder (open question 2).

## Decisions

Settled by the project:

1. **Each stream is taken once and passed on compressed.** Nothing is transcoded to record.
   Reason: recording then runs on every box, and the camera's few connections are used once.
2. **The ring holds the last seconds per camera in memory as fMP4 fragments.** Reason: pre-roll
   without writing to disk all the time, and one place for every consumer.
3. **H.264 and H.265 video; AAC and Opus audio recorded as they arrive; G.711 converted.**
   Reason: these are what home cameras send; G.711 does not play in browsers or in fMP4
   players, and converting audio costs little.

Proposed in this plan (to confirm in review):

4. **A fragment starts with a keyframe and spans one group of pictures.** Reason: every
   fragment can start a recording or a viewer, and pre-roll is a list of whole fragments.
5. **One track per fragment, with audio cut at the same moments as video.** Reason: this is a
   CMAF track, Matter cameras upload the same way so their segments are stored as they arrive,
   and aligned cuts keep both tracks in step for recording and playback. The price is a merge
   step for an MP4 download and an audio rendition in HLS, both cheap.
6. **`prft` carries the wall clock in every fragment.** Reason: the files then say when they
   were recorded, so the index can be rebuilt from them.
7. **A gap always starts a new init id.** Reason: one simple rule for consumers, whatever the
   cause of the gap.
8. **Consumers never block ingest.** Reason: a slow disk or a slow viewer must not stall the
   camera for everyone else.
9. **RTP over TCP by default.** Reason: no packet loss and no reordering on a busy Wi-Fi
   network; UDP is experimental in Retina.

## Steps

1. **[server] Sample streams for tests.** A set of short sample recordings (H.264 and H.265;
   AAC, Opus and G.711 audio; one without audio; one with a long keyframe interval; one with a
   resolution change) and a script that serves them as RTSP through MediaMTX with FFmpeg, for
   local development and for CI. A README that says how each file was made and under which
   licence.
2. **[server] RTSP client evaluation.** Connect Retina to the sample streams and to as many
   real cameras as contributors can test. Write down per camera what works and what does not,
   and decide on the client. The result goes into this plan.
3. **[server] RTSP session.** One session per stream on top of the chosen client:
   authentication, TCP transport, keepalive, clean teardown, and frames with timestamps out.
   Credentials come from the registry and never reach a log line.
4. **[server] H.264 and H.265 to fMP4.** The video side of the fragmenter: parameter sets into
   `avcC` and `hvcC`, keyframe detection, new init id on a change, `mfhd`, `tfdt`, `prft`,
   composition offsets. Tests that feed the sample streams and check the boxes.
5. **[server] Audio into fMP4.** AAC and Opus as their own track, cut at the same moments as
   the video into fragment pairs. Tests with each sample stream.
6. **[server] G.711 conversion.** Decode G.711 and encode it to the codec chosen in open
   question 2. Tests on a sample stream.
7. **[server] The ring.** Per stream buffer with the size rule and the memory limit,
   `subscribe(start)` with the atomic hand-over, shared buffers, bounded consumer queues and
   the lagged signal. Unit tests with fake fragments, including slow and fast consumers.
8. **[server] Reconnects and stall detection.** Backoff, the stall rule with the watchdog,
   timestamp checks, gaps as new init ids, demand-based start and stop. Tests with a test
   server that stalls, closes and jumps.
9. **[server] Stream health.** The health items per stream and the `stream_status` event, so
   the web interface and Home Assistant can show them.
10. **[server] Connection limits.** Classify refusals, give way in the fixed order, and the
    health message. Tested with a MediaMTX setup that limits readers, and on real cameras.
11. **[server] Probe command.** `cameras/probe` with the result and the warnings described
    above.
12. **[documentation] Camera notes.** A page where users and contributors list cameras they
    tested, with the stream addresses and quirks they found.

## Order

Step 1 comes first; it feeds every test. Step 2 needs step 1 and decides the base of step 3.
Steps 4 and 7 can start in parallel with step 3, against recorded frames from step 1. Step 5
needs step 4. Step 6 needs step 5 and open question 2. Step 8 needs steps 3 and 7 and the
task supervisor of the server foundation plan. Steps 9 and 10 need step 8. Step 11 needs step 3.
Step 12 can start at any time.

The recorder plan can start writing segment files as soon as steps 4 and 7 are in, with
fragments from the sample streams.

## Open questions

1. **Do Matter cameras cut video and audio at the same moments?** Fragment pairs assume that
   they do. If a camera's audio segments do not line up with its video segments, the recorder
   stores them as they arrive and pairs them by time instead of by number. The Matter spike
   answers this.
2. **G.711 converted to what?** AAC plays everywhere, HLS included; an encoder in Rust has to be
   found or bound (FFmpeg's AAC encoder, or another). Opus has a mature open encoder
   (libopus), but support for Opus in fMP4 differs between players and has to be tested
   in Safari. Recommendation: decide after a test of both in the browsers that the web
   interface supports.
3. **How long is the ring?** It has to cover the camera's pre-roll (registry field
   `recording.pre_roll`, a setting in the web interface), so the maximum pre-roll decides the
   ring's memory. Proposal: a maximum of some tens of seconds, a default of a few seconds, and a
   ring sized from the camera's own setting plus one group of pictures.
4. **Wall clock and drift.** How far may the server's clock and the RTP clock drift apart
   before a re-anchor? Do we use RTCP sender reports when the camera's clock is set by NTP?
   Read Moonfire's `design/time.md` and decide.
5. **A restream for go2rtc before the MVP?** Camera Assistant could offer an RTSP stream from
   the ring so Home Assistant's live view stops taking its own connection. That needs an RTSP
   server in Rust, which Retina does not have. Proposal: not in the Preview; the MVP replaces
   it with live view from Camera Assistant.
6. **Which substream setting by default?** Recording only the main stream in the Preview keeps
   connections free. Recording the substream as well makes thumbnails cheaper and helps
   playback on weak devices later. Proposal: main only by default; substream on request.

## Later, and what this must not block

- **Live view from the ring (MVP).** WebRTC for RTSP cameras, and fMP4 over a WebSocket. A
  live viewer cannot wait for a whole fragment, so the live view plan adds a subscription to
  each access unit as it arrives, and converts AAC to Opus for WebRTC. The rule
  "a fragment starts with a keyframe" stays.
- **Detection modules (MVP).** They subscribe to the substream ring like any consumer and get
  compressed fragments.
- **Matter live view through the server, and a near-live feed from continuous Push AV.** Both
  put fragments into the same ring.
- **Cameras that only offer WebRTC.** A WebRTC peer as another ingest.
- **ONVIF discovery and two-way audio.** Not part of ingest in the Preview.
