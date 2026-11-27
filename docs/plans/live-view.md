# Live view through the server, from one connection per camera

This is the technical companion of the epic "Live view through the server" on the project board.
It is written so that a developer or a coding agent can start on a step without asking what was
meant. Phase: MVP.

## Problem

Today a camera in Home Assistant is opened again for each job. go2rtc opens it for live view, the
recorder opens it to record, and detection would open it a third time. Many cheap RTSP cameras
and Matter cameras allow only one or two streams at a time. A camera that is full drops a stream
or refuses a new one, and the recording is the one that suffers.

Camera Assistant already holds one connection per camera for recording. Live view should come
from that same connection: one connection per camera serves every viewer, the recorder and
detection. Video is passed on as the camera sends it. Converting it is the last resort.

## What exists

**In Home Assistant** (`home-assistant/core`, `homeassistant/components/camera/`).

- **The WebRTC provider hook.** `CameraWebRTCProvider` in `camera/webrtc.py` is the interface a
  media server implements to serve live view for camera entities:
  - `domain`, and `async_is_supported(stream_source)`, which answers for a camera's stream source
    (a URL string);
  - `async_handle_async_webrtc_offer(camera, offer_sdp, session_id, send_message)`, which returns
    the answer and ICE candidates through `send_message`;
  - `async_on_webrtc_candidate(session_id, candidate)` and `async_close_session(session_id)`;
  - optional: `async_get_image(camera, width, height)`, `async_register_camera`,
    `async_unregister_camera`, `async_on_camera_prefs_update`.
- A provider is registered with `async_register_webrtc_provider(hass, provider)`, which returns a
  function to remove it. Each camera entity asks the registered providers when providers change
  (`async_refresh_providers` in `camera/entity.py`). It takes the first one that says it supports
  the stream source. The hook has no way to prefer one provider over another.
- A camera entity that does WebRTC itself (`_supports_native_async_webrtc`, as Matter cameras do)
  does not use a provider.
- Signalling runs over Home Assistant's WebSocket (`camera/webrtc/offer`,
  `camera/webrtc/candidate`, `camera/webrtc/get_client_config`). The media flows between the
  viewer and the provider directly, not through Home Assistant's Python process. This keeps working
  away from home with the ICE servers that Home Assistant hands out
  (`camera.async_get_webrtc_client_configuration()`).
- **go2rtc** (`homeassistant/components/go2rtc`, https://www.home-assistant.io/integrations/go2rtc/) is the
  registered provider today. Home Assistant starts the bundled binary in container installs or
  uses a configured address. Its `async_is_supported` accepts every URL scheme go2rtc knows. It
  passes the camera's ICE servers along with the offer, and also serves snapshots
  (`async_get_image`).
- The `stream` integration still serves the HLS fallback and `camera.record`. The maintainers'
  direction is to move all of it to go2rtc and then retire `stream`
  (home-assistant/architecture#1417).
- Other open proposals on the camera entity: several stream sources per camera
  (home-assistant/architecture#1417), separate video, audio and talk-back sources (#1573) and
  WebRTC-native sources (#1589).

**Matter cameras.** Live view of a Matter camera is WebRTC, signalled over Matter. The Matter
Server sets up the session and the media goes straight between the camera and the browser
(https://github.com/matter-js/matterjs-server/blob/main/docs/CAMERA_LIMITATIONS.md). From that
document:

- Browser live view is capped at 1280×720, because browsers offer H.264 level 3.1. A client that
  is not a browser can offer a higher level.
- When a camera is full, the server reuses streams whatever their purpose, and may evict another
  controller's idle streams (`camera_stream_evicted`).
- H.264 and Opus are mandatory on every Matter camera; H.265 and AAC are optional.

**In Cairn** (https://github.com/bbangert/cairn-nvr). Live view through WebRTC (`ex_webrtc`), with
a hub that replays the last group of pictures so the first frame shows at once; MSE over a
WebSocket channel; and HLS as a fallback. Home Assistant reaches it with WHEP
(`POST /api/cameras/:id/webrtc`), without trickle ICE. The ideas carry over; the code is Elixir.

**Rust WebRTC libraries.**

| Library | Notes |
|---|---|
| webrtc-rs (https://github.com/webrtc-rs/webrtc) | About 5.2k stars, 158 contributors, version 0.17. |
| str0m (https://crates.io/crates/str0m) | Sans-IO: the library does no networking itself. About 670 stars, 83 contributors, versions as git tags only. |

Both are before 1.0. go2rtc and Pion (Go) are the most used open stacks, but a Rust server cannot
link Go code.

## Concept

### One connection, many viewers

```
camera ──RTSP──▶ ingest ──▶ ring ──▶ recorder
                              │──▶ detection module (substream)
                              │──▶ WebRTC session per viewer
                              └──▶ fMP4 over WebSocket per viewer
```

The ring holds the last seconds of each stream as fMP4 fragments. A live viewer cannot wait for a
whole fragment, so the ring also lets a subscriber take each access unit as it arrives. A new
viewer starts at the last keyframe in the ring, so the first picture shows at once.

### WebRTC from the ring

- One `RTCPeerConnection` per viewer, receive-only on the viewer's side.
- Video: the server repacketises the camera's H.264 (RFC 6184) or H.265 (RFC 7798) access units
  into RTP. It never decodes or encodes video.
- Audio: WebRTC audio is Opus. The ring holds no G.711 (ingest converts it). Opus passes as it
  is; AAC, from the camera or converted from G.711, is converted to Opus for that viewer (see the
  ladder).
- A viewer that falls behind loses frames up to the next keyframe. It never slows down the ring or
  other viewers.
- A keyframe request from the viewer (PLI) is answered from the ring's last keyframe, without
  asking the camera.

### Signalling

- **In Home Assistant:** through the provider hook (below). The browser uses Home Assistant's own
  player and signalling, unchanged.
- **In Camera Assistant's own interface and API:** WebSocket commands next to the rest of the API:
  `live/offer`, `live/candidate`, `live/close`. Trickle ICE. The same session code serves both.

### The provider in Home Assistant

The Camera Assistant integration registers a `CameraWebRTCProvider` with domain `camera_assistant`.

- `async_is_supported(stream_source)` returns true for the stream sources of cameras that were
  handed to Camera Assistant. The integration keeps that set from the app's camera list.
- `async_handle_async_webrtc_offer` passes the offer to the app with the camera's ICE servers, as
  the go2rtc provider does, and passes the answer and candidates back through `send_message`.
- When the connection to the app is lost, the integration removes the provider. Home Assistant
  then asks the providers again, and go2rtc serves the camera until the app is back.
- Cameras that were not handed to Camera Assistant stay with go2rtc.

### fMP4 over a WebSocket, the fallback

For codecs and audio that WebRTC cannot carry to a viewer, Camera Assistant's player plays the
stream with Media Source Extensions (MSE):

- The server sends the init segment, then the media fragments from the ring, as binary WebSocket
  messages. They are fMP4 already, so nothing is converted except where the ladder says so.
- Small fragments keep the delay low. Their size is an open question.
- This path exists in Camera Assistant's own player only. Home Assistant's player uses WebRTC.

### Native first, transcoding last

The server sends the camera's own stream whenever the viewer can play it, and goes down a short
ladder when it cannot:

1. **The native stream,** when the viewer reports that it can decode it.
2. **Another native stream.** The substream, which many cameras send as H.264. It also serves
   small tiles and slow mobile connections.
3. **The same video with converted audio or another container.** AAC to Opus for WebRTC; WebRTC to fMP4 over a WebSocket when the codec is the problem. This costs
   little and is always available.
4. **Transcoded video,** for one viewer and only while they watch. This needs a hardware encoder
   and is not part of the MVP.

The viewer reports what it can decode when it asks for a stream (from the browser's
`RTCRtpReceiver.getCapabilities()` and `MediaSource.isTypeSupported()`), and the server picks the
highest step that fits. To keep step 4 rare, the setup check flags a camera that sends H.265 on
every stream.

### Matter cameras

- **In the MVP,** a Matter camera keeps its own WebRTC session, straight from the camera to the
  viewer, as Home Assistant does today. Camera Assistant's player shows it in the same place as
  the other cameras, using Home Assistant's camera signalling for that entity.
- **Later,** Camera Assistant holds one WebRTC session with the camera and passes it on to every
  viewer, as it does for RTSP. That saves the camera's few encoder slots.

### Relation to go2rtc

- Without Camera Assistant, nothing changes: go2rtc stays the default media server.
- With Camera Assistant, it serves live view for the cameras that are handed to it. go2rtc stays
  for everything else: cameras not handed over, and the many sources and protocols Camera
  Assistant does not handle.
- Camera Assistant reuses go2rtc's knowledge of camera quirks and its behaviour as a reference, not
  its code.

## Decisions

1. **Live view of RTSP cameras comes from the ring.** Reason: one connection per camera serves
   every viewer, the recorder and detection, and cameras with few stream slots keep recording.
2. **WebRTC is the first transport.** Reason: it has the lowest delay, and it works away from home
   with Home Assistant's signalling and ICE servers, without video passing through Home
   Assistant.
3. **In Home Assistant, Camera Assistant plugs into the existing WebRTC provider hook.** Reason:
   Home Assistant's own player, cards and remote access work unchanged, and Home Assistant needs
   no second live path.
4. **fMP4 over a WebSocket is the fallback,** in Camera Assistant's own player. Reason: it carries
   codecs and audio that WebRTC cannot, and the ring holds fMP4 already.
5. **Native first, transcoding last.** Video is never transcoded in the MVP. Reason: the boxes
   this runs on often have no usable hardware encoder, and the camera's own streams cover most
   viewers.
6. **Matter cameras keep their own WebRTC session in the MVP.** Reason: it works today, and passing
   a Matter session on needs work in the Matter Server and in the server's WebRTC client.
7. **go2rtc stays.** It is the default without Camera Assistant and the fallback for sources the
   server does not handle. Reason: it supports far more camera protocols than Camera Assistant
   will at first.

## Steps

1. **[server] Choose the WebRTC library.** Build the smallest send-only session with webrtc-rs and
   with str0m: H.264 from a file to a browser on the LAN and through a TURN server. Measure memory
   and CPU per session on a Green. Write down the choice and why.
2. **[server] Live subscription on the ring.** A subscriber gets each access unit as it arrives,
   starting at the last keyframe, with a bounded queue that drops to the next keyframe when the
   subscriber is slow.
3. **[server] WebRTC sessions from the ring.** One session per viewer: H.264 and H.265 into RTP,
   Opus audio as it is and AAC converted to Opus, keyframe requests answered from the ring, sessions closed when
   the camera stream ends or the viewer leaves.
4. **[server] Live view commands in the API.** `live/offer`, `live/candidate` and `live/close`
   over the WebSocket API, with trickle ICE and the ICE servers given by the client. Errors with a
   code and a reason.
5. **[hass-integration] WebRTC provider in Home Assistant.** A `CameraWebRTCProvider` in the
   integration and a matching call in the Python client library: supported for the stream sources
   of handed-over cameras, offers forwarded with the camera's ICE servers, candidates and close
   forwarded, the provider removed when the app is not reachable.
6. **[hass-integration] Prefer Camera Assistant for its cameras.** The hook takes the first
   provider that supports a stream source. Propose a way in Home Assistant Core to prefer one
   provider, so the result does not depend on the order of registration, and adopt it.
7. **[server] Pick the stream per viewer.** The viewer's decode capabilities come with the request;
   the server picks the native stream, or the substream, or the path with converted audio.
8. **[server] Audio conversion for live view.** AAC to Opus for WebRTC. G.711 never
   reaches live view: ingest converts it before the ring. Only for the audio track, only while someone watches.
9. **[server] fMP4 over a WebSocket.** Init segment and fragments from the ring as binary
   messages, starting at a keyframe, for the player's fallback.
10. **[frontend] The live player.** Live view in the player component of the web interface plan
    (`packages/components`): it asks for WebRTC first and falls back to fMP4 over a WebSocket, sends the browser's decode capabilities, shows the stream it plays, and
    reconnects after a drop.
11. **[frontend] Matter cameras in the same player.** For a Matter camera, the player uses Home
    Assistant's camera WebRTC signalling for that entity, so the camera's own session shows in the
    same place.
12. **[server] Live view health.** Per camera: the number of viewers, their transport and stream,
    and bitrate, in the health API.
13. **[documentation] Live view and go2rtc.** What changes when Camera Assistant is installed,
    which cameras it serves, and what stays with go2rtc.

## Order

- Step 1 comes first. Step 2 needs the ring. The WebRTC fallback for Matter cameras without Push
  AV (Matter cameras plan, step 12) builds its receiving peer on the library step 1 chooses.
- Step 3 needs steps 1 and 2. Step 4 needs step 3.
- Step 5 needs step 4 and the camera hand-over in the integration. Step 6 can start at once and is
  needed before the MVP release.
- Steps 7 and 8 need step 3. Step 9 needs step 2.
- Step 10 needs steps 4 and 9. Step 11 needs step 10 and the panel.
- Step 12 needs step 3. Step 13 comes last.

## Open questions

1. **WebRTC library.** webrtc-rs or str0m. Step 1 decides.
2. **Provider preference in Home Assistant.** Options: a priority on the provider, a provider
   chosen per camera, or a check that looks at the camera and not only its stream source. To work
   out with Home Assistant Core.
3. **Snapshots.** The provider hook can also serve stills (`async_get_image`). Options: return
   nothing and leave stills to the camera entity; take a decoded frame from the detection module;
   or decode one keyframe on request.
4. **H.265 over WebRTC.** Which browsers and Home Assistant apps accept H.265 over WebRTC, and
   which play H.265 through MSE, is to be measured. The answer decides how often the substream is
   used.
5. **Fragment size for the fallback.** Small fragments lower the delay and cost overhead. A
   starting value comes from measurement.
6. **Signalling for other clients.** Is WHEP (HTTP) worth adding next to the WebSocket commands,
   for clients other than Camera Assistant's own player?
7. **HLS for Home Assistant.** For a camera whose codec WebRTC cannot carry to a viewer, should the
   app also serve HLS to Home Assistant's player, or is the fallback in Camera Assistant's own
   player enough?
8. **Audio libraries.** Candidates to evaluate for AAC decoding and Opus encoding in Rust; the
   choice must fit Apache 2.0.
9. **Limits.** Is there a maximum number of viewers per camera on a Green? To be measured in step
   1.

## Later, and what this must not block

- **Matter live view through the server.** One WebRTC session from the camera to Camera Assistant,
  passed on to every viewer. Near-live from a continuous Push AV upload, a few seconds behind, as
  the fallback. The session code of step 3 must not assume that its source is RTSP.
- **Transcoded video on request,** for a viewer that cannot play any stream the camera offers. It
  needs a hardware encoder, such as Intel graphics. The ladder of step 7 has a place for it.
- **Two-way audio.** WebRTC sessions are receive-only for the viewer in the MVP; the session code
  must allow a send track later.
- **The media gateway role in place of go2rtc,** also for recordings of other recording systems.
- **WebRTC playback of recordings** (after the MVP), for clients that cannot reach the server
  directly. It has no plan yet; the session code of step 3 must not assume a live source.
- **Scrolling back from live into recordings in the same player.** Recordings use HLS over the
  stored segments; the player must be able to switch between the two.
- **Cloud cameras that only offer WebRTC** (Nest, Ring) as a source.
- **G.711 straight to WebRTC.** WebRTC can carry G.711, so passing the camera's G.711 on without
  a conversion is a possible saving. It would need the ring to keep G.711 next to the converted
  track.
