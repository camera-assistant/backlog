# Matter cameras record to Camera Assistant

Status: draft plan for the Preview phase. Epic: 06. This is the technical companion of the epic on the project board. It is written so a developer or a coding agent can start on a step without asking what was meant.

Terms used here:

- **Matter Server**: the Matter controller that Home Assistant uses ([matter-js/matterjs-server](https://github.com/matter-js/matterjs-server)). Camera Assistant talks to it over its WebSocket API, next to Home Assistant.
- **Push AV**: the Push AV Stream Transport cluster. The camera uploads its video as CMAF segments over HTTPS to an address the controller sets.
- **Upload endpoint**: the HTTPS server inside Camera Assistant that receives those uploads.
- **Data model**: the public XML description of every Matter cluster in the Matter SDK ([connectedhomeip/data_model](https://github.com/project-chip/connectedhomeip/tree/master/data_model)). The full specification text is not public, so facts below come from the data model and the SDK code.

## Problem

A Matter camera has no recorder in the home today.

- Matter defines one thing for recording: the camera can be told to push its video to an HTTPS address. Whoever owns that address stores, indexes and plays back. Matter has no recorder device type, no clip catalogue, no timeline and no retention rules.
- Home Assistant shows Matter cameras with live view through the Matter Server, but it records nothing from them. The Matter Server has no support for Push AV.
- The one platform that records Matter cameras today records to its own cloud. A household that wants its video to stay at home has no option.

Camera Assistant becomes the place a Matter camera uploads to. The camera then records to local and network storage, on its own motion events, continuously, or on command, without a cloud.

## What exists

**In Matter (data model 1.6.1, with the 1.7 draft noted).**

| Part | What it gives | Source |
|---|---|---|
| Camera device type | Push AV Stream Transport is **optional** on a camera. A certified camera may offer only WebRTC live view and no upload at all. | `data_model/1.6.1/device_types/` |
| Push AV Stream Transport (0x0555) | `AllocatePushTransport`, `ModifyPushTransport`, `DeallocatePushTransport`, `SetTransportStatus`, `ManuallyTriggerTransport`, `FindTransport`. Events `PushTransportBegin` and `PushTransportEnd`. The configuration is fabric-scoped. Allocation needs Manage privilege. | [data model](https://github.com/project-chip/connectedhomeip/tree/master/data_model) |
| TLS Certificate Management (0x0801) | The controller installs trusted root certificates on the camera (`ProvisionRootCertificate`, at most 3000 bytes DER). The camera generates a key and a signing request (`ClientCSR`). The controller signs it and installs the result (`ProvisionClientCertificate`). Needs Admin privilege. | data model |
| TLS Client Management (0x0802) | A "TLS endpoint": host name (4 to 253 bytes), port, which root to trust and which client certificate to present. Needs Admin privilege. | data model |
| Zone Management (0x0550) | Zones with motion triggers and sensitivity. Events `ZoneTriggered` and `ZoneStopped`. Optional on a camera. | data model |
| Camera AV Stream Management (0x0551) | Streams per usage (recording, live view, analysis), encoder limits (`MaxConcurrentEncoders`, `MaxEncodedPixelRate`, `MaxNetworkBandwidth`, `MaxContentBufferSize`), privacy switches. | data model |

Facts from the data model and the SDK code that shape the upload endpoint:

- **Transport options.** Stream usage and lists of video and audio streams, a TLS endpoint id, a URL of 13 to 2000 characters, trigger options, the ingest method (only CMAF ingest) and container options (only CMAF, with a segment duration of 500 to 65500 ms and an optional chunk duration of at most half a segment). An optional expiry time.
- **URL rules** (SDK `PushAVStreamTransportLogic::ValidateUrl`, [source](https://github.com/project-chip/connectedhomeip/blob/master/src/app/clusters/push-av-stream-transport-server/PushAVStreamTransportLogic.cpp)): `https` only, a host is required, the path must end with `/`, no query string and no fragment. There is no room for a token in the URL: the camera is known by its client certificate.
- **Mutual TLS is mandatory.** The SDK refuses a push transport whose TLS endpoint carries no client certificate.
- **The camera is the HTTP client.** The SDK camera uploads with libcurl, HTTP/2 and PUT, and verifies the server's certificate and host name ([pushav-uploader.cpp](https://github.com/project-chip/connectedhomeip/blob/master/examples/camera-app/linux/src/uploader/pushav-uploader.cpp)).
- **Path layout** under the URL, as the SDK's reference ingest server validates it ([validation.py](https://github.com/project-chip/connectedhomeip/blob/master/src/tools/push_av_server/src/validation.py)):
  - `session_<N>/index.mpd` or `session_<N>/index.m3u8`
  - `session_<N>/<track>/index.m3u8`
  - `session_<N>/<track>/<name>.init`
  - `session_<N>/<track>/segment_<M>.m4s`
  - A dynamic MPD marks the start of a session. A static MPD marks its end.
- **Triggers.** `Continuous` (streams while the transport is active), `Motion` (camera motion, optionally limited to zones, sensitivity 1 to 10) and `Command` (only through `ManuallyTriggerTransport`, with a reason: user, automation, emergency or doorbell press). The 1.7 draft adds a provisional trigger on detected context.
- **Clip timing** for motion: initial length, extension on new motion, maximum length, and a cool-down after a clip. Pre-roll (`MaxPreRollLen`) is required for motion and command triggers. A pre-roll that conflicts with the key-frame interval is refused.
- **One trigger is one session** (`session_<N>`). `PushTransportBegin` and `PushTransportEnd` carry the connection id, trigger type, activation reason and, since revision 2, the CMAF session number. That lets a controller match uploads to triggers.
- **Codecs.** Video: H.264 is mandatory, H.265 and AV1 are optional. Audio: Opus is mandatory, AAC is optional. Streams arrive as the camera encodes them.
- **Time.** A status `InvalidTime` exists: the camera needs a set UTC time before TLS works.

**In the Matter SDK.** A reference ingest server ([push_av_server](https://github.com/project-chip/connectedhomeip/tree/master/src/tools/push_av_server)) in Python runs on a local address with its own certificate authority. It signs camera requests, accepts the uploads and validates them. It is a test tool, not a product. The certification tests (`TC_PAVST_*` in [python_testing](https://github.com/project-chip/connectedhomeip/tree/master/src/python_testing)) show the exact order of setup: root certificate, signing request, TLS endpoint, allocation. A Linux reference camera ([camera-app](https://github.com/project-chip/connectedhomeip/tree/master/examples/camera-app)) implements the uploader, pre-roll and zones. No production-grade open-source upload server was found.

**In the Matter Server.** The WebSocket API ([websockets_api.md](https://github.com/matter-js/matterjs-server/blob/main/docs/websockets_api.md)) has:

- generic commands: `device_command` and `batched_device_commands` for any cluster command, `read_attribute`, `write_attribute`;
- `node_event`, which forwards Matter events from devices to every listening client;
- a camera API for live view: `camera_get_capabilities`, `camera_start_stream`, `camera_stop_stream`, `camera_snapshot`, `camera_release_stream`, and the events `camera_session_ended` and `camera_stream_evicted`.

The server manages a camera's limited encoders for its clients. When a camera is full it reuses streams, asks for smaller ones, and can free idle streams of other controllers. `allow_eviction: false` turns that off for one request ([CAMERA_LIMITATIONS.md](https://github.com/matter-js/matterjs-server/blob/main/docs/CAMERA_LIMITATIONS.md)). It already pushes the time to a camera that refuses a TLS-based ICE server because its clock is not set. It has no Push AV support and no commands for the TLS clusters. The matter.js library has generated client types for Push AV but no ingest server ([matter.js](https://github.com/matter-js/matter.js)).

**In other projects.** The UGREEN ID500 Pro sends its recordings to a local hub; whether that is Push AV is not confirmed ([Matter Alpha](https://www.matteralpha.com/news/ugreen-matter-camera-id500-pro-opens-for-preorder-with-homeagent-hub)). A SmartThings community thread describes Push AV uploads with the same session layout to its cloud, and bugs in motion-triggered clips ([SmartThings community](https://community.smartthings.com/t/matter-cameras-device-initiated-recording-never-produces-a-clip-the-upload-url-only-accepts-one-session/310641)).

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) has no Matter support. Its event clips and SQLite index are a reference for the recorder, not for this epic.

## Concept

```
            Matter fabric (Home Assistant's)
  camera  <---------------------------------  Matter Server  <--- WebSocket ---  Camera Assistant
    |                                                                               |
    |  HTTPS PUT, mutual TLS, CMAF segments                                         |
    +------------------------------------------------------------------>  upload endpoint
                                                                                    |
                                                                     recorder, ring, event logic
```

Control runs from Camera Assistant through the Matter Server to the camera. Video runs from the camera straight to Camera Assistant. Home Assistant is not in either path.

### Components in the server

**1. Certificate authority (`matter::ca`).**

- One root key and root certificate per Camera Assistant installation, created on first start and kept in the app's data folder. The private key never leaves the app.
- The root certificate must fit in 3000 bytes DER. An ECDSA P-256 key keeps it small.
- A server certificate for the upload endpoint, signed by the root, with subject alternative names for every host the app hands to cameras (an IP address, a local DNS name, or both; see the open questions).
- Signing a camera's request: input is the DER request from `ClientCSR`, output is a client certificate for that camera. The app stores the certificate's fingerprint with the camera id. That record is how the upload endpoint knows which camera is uploading.
- Rotation: the server certificate can be reissued without touching cameras, because cameras trust the root. A new root means a new setup on every camera.
- Libraries worth evaluating: `rcgen` for certificates and signing, `rustls` for the TLS server with client certificate checks.

**2. Upload endpoint (`ingest::push_av`).**

- An HTTPS server on its own port, separate from the API port, so its TLS settings (client certificates required) never mix with the API's.
- HTTP/2 and HTTP/1.1. The SDK camera uses HTTP/2; whether real cameras also use HTTP/1.1 is for the spike.
- Mutual TLS: a client certificate signed by the app's root is required. The fingerprint maps to a camera. An unknown certificate is refused at the TLS layer.
- The URL given to a camera is `https://<host>:<port>/push/<camera id>/`. The path must end with `/`. The camera id in the path must match the camera of the certificate; a mismatch is refused with 403.
- Accepted: `PUT` of `.init`, `.m4s`, `.mpd` and `.m3u8` under `session_<N>/` and `session_<N>/<track>/`, following the path layout above. Anything else gets 404 or 405. A size limit per request protects the disk.
- Each init segment and media segment is handed to the recorder as it arrives (`intake_init`, `intake_segment` in the recorder plan). The recorder owns the files, the index and retention; this component only checks and hands over.
- The stored segments are CMAF already, so playback can serve them as HLS without any conversion.

**3. Session tracking (`ingest::push_av::session`).**

- A session (`session_<N>`) with a motion or command trigger is one event with its clip: its camera, start, end, tracks, codecs and trigger.
- A session with the continuous trigger is continuous footage (recording spans in the index), not an event. Events on top of it come from the camera's own zone and motion events (component 7).
- Start: the first upload of a session, or a dynamic MPD. End: a static MPD, the `PushTransportEnd` event, or a timeout when neither arrives (a camera that reboots mid-session).
- `PushTransportBegin` and `PushTransportEnd` give the trigger type and the reason (motion, command with its reason, continuous). The CMAF session number links the event to the uploads.
- For continuous recording the sessions are long. The recorder decides how they map to its segments on disk.

**4. Matter Server client (`matter::server`).**

- A WebSocket client for the Matter Server's API: connect, check the schema version, `start_listening`, keep the node list current, reconnect with backoff.
- Finds camera endpoints by device type (Camera, Video Doorbell, Floodlight Camera; Audio Doorbell and Snapshot Camera are not video recorders) and reads which clusters each has. A camera without Push AV is shown as "live view only" or falls back to WebRTC (component 7).
- Receives `node_event` for the clusters it cares about: Push AV, Zone Management, the doorbell's switch.
- The address of the Matter Server comes from settings. With Home Assistant on Home Assistant OS, the integration can hand it over later; in the preview the user enters it or the app finds the Matter Server app on the same host (see the open questions).

**5. Camera setup (`matter::setup`).** The order from the certification tests, done through the Matter Server:

1. Read the camera's TLS and Push AV attributes. Check its time is set.
2. Install the app's root certificate (`ProvisionRootCertificate`), unless it is there already. Keep the returned id.
3. Ask the camera for a signing request (`ClientCSR`), sign it, install the result (`ProvisionClientCertificate`). Keep the id.
4. Create the TLS endpoint (`ProvisionEndpoint`) with the upload host, port, root id and client certificate id.
5. Pick or allocate the camera's video and audio streams for recording, through the Matter Server, so that it knows these streams are in use (see "What the Matter Server must gain").
6. Allocate the push transport (`AllocatePushTransport`) with the URL, the trigger options of the recording mode, the segment duration and the pre-roll.
7. Store the connection id and everything needed to undo the setup.

Teardown, when the camera is removed or recording is switched off: deallocate the transport, remove the endpoint and certificates where the cluster allows it, forget the fingerprint. On every start the app compares what it stored with what the camera reports (`FindTransport`) and repairs differences.

**6. Triggers and recording modes (`matter::triggers`).**

| Recording mode in Camera Assistant | Push AV trigger |
|---|---|
| Off | No transport, or the transport set inactive (`SetTransportStatus`) |
| On events | `Motion`, with the camera's zones when it has them, a pre-roll and clip timing |
| Continuous | `Continuous` |
| A clip on request (`record_clip` from Home Assistant, a doorbell press) | `ManuallyTriggerTransport` with the reason `Automation`, `UserInitiated` or `DoorbellPressed` on a `Command` transport |

Whether one camera accepts a `Command` transport next to a `Motion` or `Continuous` transport, given its few encoders, is for the spike. If it does not, a clip on request on a continuous camera is cut from the continuous footage instead.

Pre-roll is the camera's one per-camera setting (`recording.pre_roll`, server foundation plan). The camera does the buffering, so the server sends that value as `MaxPreRollLen` when it allocates the transport, clamped to what the camera supports. Proposed defaults, to confirm in the spike: a segment duration of 4 seconds (cameras tested with the Matter Server only accepted a 4000 ms key-frame interval, and a segment should start on a key frame), and clip timing from the camera's own defaults where it reports them.

**7. Event following (`matter::events`).**

- `ZoneTriggered` and `ZoneStopped` become motion events with the zone, handed to the event logic, which links them to the recording of the same time.
- `PushTransportBegin` and `PushTransportEnd` open and close recordings (component 3).
- A doorbell press (the switch cluster of a Video Doorbell) becomes a doorbell event and can start a clip.
- The event model follows the labels and per-zone triggers of Matter, so the later AV Analysis Node role fits without a new model.

**8. WebRTC fallback, for cameras without Push AV (MVP).**

- Camera Assistant holds a WebRTC session to the camera through `camera_start_stream` with the usage for recording and `allow_eviction: false`.
- It uses a receive-only WebRTC peer, built in this epic on the WebRTC library chosen in the live view plan, and turns the RTP stream into fMP4 fragments for the ring and the recorder.
- A non-browser peer can offer a higher H.264 level than a browser and so get 1080p or more.
- It costs an encoder slot all the time, so it is only used for a camera without Push AV, and only in the continuous or events mode the user picks.

### What the Matter Server must gain

The Matter Server has no Push AV support today. Camera Assistant needs:

1. **Commands for the TLS clusters**, so a client can install a root certificate, get a signing request, install a client certificate and create, list and remove TLS endpoints on a camera. Either as typed `camera_*` commands, or by confirming that `device_command` covers these cluster commands and their responses.
2. **Commands for Push AV**: allocate, modify, deallocate, set status, trigger manually, find. As with live view, the server should pick or allocate the streams a push transport uses, so it does not free them for another client.
3. **Protection of recording streams.** A stream that a push transport uses must not be freed for a live view. Today the server can free another controller's idle streams. A stream allocated by a client through `device_command` may look like such a stream.
4. **Events**: Push AV, Zone Management and doorbell events reach Camera Assistant through `node_event`. This exists; the Push AV events must be decoded.
5. **Clock**: the server keeps the camera's time set before TLS setup and after a camera restart, as it does for TURN today.

This work lands in [matter-js/matterjs-server](https://github.com/matter-js/matterjs-server) as pull requests there. The interface is agreed with its maintainers first, in an issue in that repository.

### Data shapes

A Matter camera in Camera Assistant's registry:

```
MatterCameraSource {
  node_id, endpoint_id,               // in the Matter Server
  device_type,                        // camera, video doorbell, floodlight camera
  has_push_av, has_zones, has_doorbell,
  video_codecs, audio_codecs,         // as the camera states them
  push: {                             // empty until set up
    root_cert_id, client_cert_id, client_cert_fingerprint,
    tls_endpoint_id, connection_id,
    trigger, segment_ms, pre_roll_ms,
  },
  state: not_set_up | setting_up | recording | idle | error(reason)
}
```

An upload session, handed to the recorder:

```
PushSession {
  camera_id, session_number, connection_id,
  trigger, activation_reason,         // from PushTransportBegin
  tracks: [{ name, kind: video|audio, codec, init_path }],
  started_at, ended_at,               // ended_at empty while open
  segments: [{ track, number, path, size, duration }]
}
```

## Decisions

1. **Camera Assistant is the upload endpoint, and it runs the certificate authority.** The spec leaves recording to whoever owns the upload address. Only then does video stay in the home and need no cloud.
2. **Camera Assistant works through the Matter Server, not as a Matter controller of its own.** The Matter Server already holds the camera on Home Assistant's fabric and manages its streams. A second fabric would need a second commissioning and would compete for the same few streams without coordination.
3. **Push AV first, WebRTC as the fallback.** Push AV delivers finished CMAF segments, so recording needs no media pipeline. WebRTC costs an encoder slot all the time and a full WebRTC stack. The fallback comes in the MVP.
4. **Segments are stored as they arrive.** No conversion. HLS playback serves the CMAF segments directly.
5. **The camera is known by its client certificate.** The URL cannot carry a token, and the spec requires mutual TLS.
6. **A spike with real cameras comes first.** Three points are unproven, and they decide the design of the endpoint (see the open questions).
7. **The changes to the Matter Server are made there, in the open,** with an agreed interface before the code.

## Steps

1. **[matter] Spike: Push AV from real cameras to a local endpoint.** Run the SDK's reference ingest server, or a small prototype, on the local network. Set up Push AV on as many certified cameras as can be found, through a Matter controller that can send the cluster commands (the SDK's camera controller or the Matter Server's `device_command`). Answer, per camera: is Push AV there at all; is an IP address accepted as host; is the host name checked against the certificate; how is the clock set; which triggers, segment durations and pre-roll values are accepted; HTTP/2 or HTTP/1.1; can a command transport live next to a motion or continuous one; how events and uploads line up in time. Write the results in `docs/research/matter-push-av-spike.md` in the backlog.
2. **[server] Certificate authority.** Root key and certificate on first start, server certificate with the right names, signing of a camera's request, the fingerprint record, server certificate renewal. Unit tests with requests made by `openssl` and by the SDK's tools.
3. **[server] Upload endpoint.** HTTPS with mutual TLS on its own port, HTTP/2 and HTTP/1.1, the path layout, PUT only, the camera check, size limits, hand-over to the recorder. Tests with a client that uses a signed client certificate, and with requests the SDK server's validation would refuse.
4. **[server] Upload sessions become recordings.** Start and end of a session from manifests, events and timeouts. Tracks and codecs read from the init segments. A motion or command session opens and closes one event in the recorder; a continuous session is footage without an event. Segment files closed at the end.
5. **[matter] Matter Server: TLS cluster commands.** Agree the interface in an issue in matter-js/matterjs-server, then build it there: root certificate, signing request, client certificate, TLS endpoints.
6. **[matter] Matter Server: Push AV commands and protected recording streams.** Allocate, modify, deallocate, set status, trigger, find. The server picks the streams and does not free them for live view. Push AV events are decoded in `node_event`.
7. **[server] Matter Server client in Camera Assistant.** WebSocket connection, schema check, node list, camera endpoints and their clusters, events. Works against a recorded set of Matter Server messages in tests.
8. **[server] Set up and tear down a camera.** The setup order of component 5, stored state, compare and repair on start, teardown. State and errors per camera for the health view.
9. **[server] Recording modes and triggers.** The mapping table of component 6. A clip on request through `ManuallyTriggerTransport`. Defaults from the spike.
10. **[server] Follow the camera's events.** Zone, Push AV and doorbell events into the event logic and the index.
11. **[server] End-to-end test with the SDK's Linux camera.** Build the SDK camera app in a container, set it up through a Matter Server, and check that a clip lands in the index. Run it in CI if the build time allows, otherwise on demand.
12. **[server] WebRTC fallback for cameras without Push AV (MVP).** Hold a recording session through `camera_start_stream`, RTP to fMP4, into the ring and the recorder.
13. **[documentation] Docs: recording a Matter camera.** What a household needs, which cameras were tested and what works on each, what to do when a camera has no Push AV.

## Order

- Step 1 comes first and decides details of steps 2, 3 and 9. Steps 2 and 3 can start in parallel with it on the parts the spike does not touch.
- Step 3 needs step 2. Step 4 needs step 3 and the recorder's index.
- Steps 5 and 6 run in the Matter Server's repository and can start once the interface is agreed. Step 8 needs them, or `device_command` if the spike shows it is enough for a first version.
- Step 7 can start at once. Steps 8, 9 and 10 need step 7.
- Step 11 needs steps 3 to 9.
- Step 12 builds the receiving WebRTC peer on the library chosen in step 1 of the live view plan, and needs the ring. It belongs to the MVP.
- Step 13 comes last and is updated with every camera tested.

## Open questions

1. **IP address or DNS name as upload host.** The host is a free field of 4 to 253 bytes, and libcurl checks the server certificate's names against it. Whether cameras accept an IP literal, and whether the specification requires a DNS name, is not verified. Options: an IP address with an IP entry in the certificate; a local DNS name (mDNS `.local`) that every camera can resolve; both in the certificate. The spike decides.
2. **Clock on cameras.** TLS needs the camera's UTC time. How each camera keeps it (Matter time sync, NTP, the controller pushing it) is not verified. Options: the Matter Server pushes the time before setup and after a restart; Camera Assistant checks the time it reads and refuses setup with a clear message.
3. **How complete vendors' Push AV support is.** Push AV is optional, and only uploads to one cloud are documented. Which shipping cameras support it to another endpoint, and with which triggers, is not verified. The answer decides how soon the WebRTC fallback is needed.
4. **The Matter Server interface.** Typed `camera_*` commands for Push AV and TLS, or the generic `device_command`? Typed commands let the server keep track of the streams; `device_command` needs no change to start. Proposal: typed commands, with `device_command` for the spike.
5. **Where the Matter Server address comes from** without Home Assistant handing it over. Options: a setting; discovery of the Matter Server app through the Supervisor; mDNS.
6. **Content encryption.** No field for per-segment encryption was found in the data model. Whether the specification requires it is not verified.
7. **Battery cameras.** Continuous recording is not realistic on a battery camera. Does the app offer only "on events" for them, and how does it know a camera runs on a battery?
8. **Certificate lifetimes.** How long are client and server certificates valid, and how does the app renew a camera's client certificate without a gap in recording?
9. **Several Camera Assistant installations, or another recorder on the same camera.** Push AV transports are per fabric. Two recorders on one fabric would both need streams from the camera. Out of scope for the preview, but the setup should detect a transport it did not create.
10. **Pre-roll limits.** The data model has no fixed maximum for `MaxPreRollLen`. A value that conflicts with the key-frame interval is refused, and the camera's buffer size is `MaxContentBufferSize`. How to turn these into the clamp for the per-camera setting is not clear from the research; the spike measures what cameras accept.

## Later, and what this must not block

- **Matter live view through Camera Assistant** (after the MVP). One WebRTC session from the camera, passed on to every viewer. The Matter Server client and the WebRTC peer of this epic are the base.
- **Near-live from a continuous upload.** A few seconds behind, from the ring.
- **The AV Analysis Node role** once Matter 1.7 is final: Camera Assistant reports its detections as standard Matter events. That makes the app a Matter node of its own, not only a client of the Matter Server. The event model of this epic follows Matter's labels and zones so it fits.
- **The camera's own analysis as a detection source,** when cameras report person, pet or vehicle (provisional in 1.6, on cameras from 1.7 at the earliest).
- **Detected context as a trigger** (`AmbientContext`, 1.7 draft).
- **Hand-over from Home Assistant** (MVP). In the preview a Matter camera is picked in the app's own interface; later Home Assistant hands it over with the other cameras. The registry entry above stays the same.
