# Camera Assistant in Home Assistant: install it, see its events, watch its recordings

Status: draft plan for the Preview phase. Epic: 07. This is the technical companion of the epic on the project board. It is written so a developer or a coding agent can start on a step without asking what was meant.

Terms used here:

- **The server**: the Camera Assistant server (`camera-assistant/camera-assistant`), with its WebSocket and HTTP API.
- **The app**: the server packaged as a Home Assistant app, for aarch64 and x86-64.
- **The client library**: a Python package that speaks the server's API.
- **The integration**: the Home Assistant integration for Camera Assistant, in Home Assistant Core.

## Problem

Camera Assistant records, but a household lives in Home Assistant. Without an integration:

- the app has to be found, installed and started by hand;
- events from cameras cannot start automations;
- a recording can only be watched in the app's own interface;
- an automation cannot ask for a clip ("record 30 seconds when the doorbell rings").

Home Assistant has a `camera.record` action today, but it records inside Home Assistant's own process: at most 3600 seconds, one recording per camera at a time, no way to stop it, and only to a local file path. Playback of recordings from other systems also runs through Home Assistant Core: the Frigate integration proxies HLS, UniFi Protect streams MP4, and the Reolink integration downloads and converts recordings in Core (core#192887). Camera Assistant should not add to that.

## What exists

**The "dedicated server, thin integration" pattern in Home Assistant.** Several integrations connect to a server that runs in its own process or app:

| Server | Integration and client library | Transport | Install |
|---|---|---|---|
| Z-Wave JS Server | `zwave_js`, `zwave-js-server-python` | WebSocket JSON, schema version agreed on connect | `zwave_js/addon.py` uses `AddonManager` |
| Matter Server | `matter`, `matter-python-client` | WebSocket JSON with a schema version | `matter/addon.py` uses `AddonManager` |
| Music Assistant | `music_assistant`, `music-assistant-client` | WebSocket JSON, server schema version | `music_assistant/app.py` uses `AddonManager` |
| go2rtc | `go2rtc` | REST and WebSocket | binary bundled with Home Assistant; a repair when its version is too old |

What they share:

1. The heavy work runs in a separate process with a versioned WebSocket JSON API.
2. A thin Python client library on PyPI.
3. A config flow that finds the server through Supervisor discovery or zeroconf, and can install and start the app through `homeassistant.components.hassio.AddonManager`.
4. A version check on connect.

All three are in [home-assistant/core](https://github.com/home-assistant/core) under `homeassistant/components/`.

**How recordings show today.** Through the Media browser (`media_source`). An integration builds a tree of folders and items and resolves an item to a URL with a MIME type (`async_resolve_media` returns `PlayMedia`). The model is browse-only: no time range query, no filters by label, no event metadata. In practice every integration returns a URL on Home Assistant itself and passes the bytes through Core, for authentication, remote access and browser support. Supervisor ingress is no exception: `hassio/ingress.py` streams the app's response through Core.

**Entities for events.** The `event` entity platform fires an event with an event type and attributes; its state is the time of the last event. It is what Home Assistant uses for doorbells and buttons.

**Labs.** Home Assistant ships preview features as Labs features that a user switches on in the settings. An integration declares its preview features in its `manifest.json`, and checks them with `async_is_preview_feature_enabled` from the `labs` component (`homeassistant/components/labs/`). Two-way audio for Matter cameras is an example.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) has a draft HTTP API for Home Assistant (`docs/ha-api.md`): a camera list, an event list with filters and paging, a server-sent event stream for event start, update and end, clips served with HTTP Range, snapshots, and WHEP for live view. It never sends camera credentials. It is a useful reference for the shape of events and media; the integration for it was not written.

**Prior art for the API shape.** The Advanced Camera Card's engine interface ([engine.ts](https://github.com/dermotduffy/advanced-camera-card/blob/main/src/camera-manager/engine.ts)) queries events and recordings by camera, time window, label and zone, and has an event stream with new, update and end. The Music Assistant client ([music-assistant/client](https://github.com/music-assistant/client)) shows a client library with typed models shared with the server.

## Concept

```
Home Assistant                                        Camera Assistant (app)
  integration  --- WebSocket API (client library) --->  server
     |   event entity, last event per camera               |
     |   record_clip action                                |
     |   Media browser: tree from the API                  |
     |                                                     |
  browser / app  <---------- HLS or MP4, signed URL -------+  (direct when reachable)
```

The integration is a client of the server's API. It holds no video. The app never needs credentials for Home Assistant.

### The client library

- Package name: open (see the open questions). Python, typed, `asyncio`, built on `aiohttp`.
- Connects to the server's WebSocket, reads the server info (version, schema version), and refuses to work with a schema it does not support, with a clear error.
- Models for the API's objects: server info, camera, recording mode, event, recording span, storage location, health. The models follow the server's versioned schema; how they are kept in sync is part of step 1.
- Commands the preview needs: list cameras, get one camera, list events (camera, time window, type, page), get an event, start a clip, browse recordings by day, get a playback URL for an event or a time range.
- Subscriptions: event started, updated and ended; camera added, changed and removed; health changed.
- Reconnects with backoff and resubscribes. Raises typed errors for "not connected", "invalid schema" and command failures.
- Tested against a fake server in the package's own tests, without a running Camera Assistant.

### The integration

**Setup (config flow).**

- On Home Assistant OS or Supervised: the flow offers to install and start the app through `AddonManager`, the same way as `zwave_js`, `matter` and `music_assistant`. It then connects to the app's API address from Supervisor discovery.
- Elsewhere: the user enters the server's URL. Zeroconf discovery can follow if the server announces itself.
- The flow checks the schema version and stops with a clear message if the app is too old or too new.
- The server always requires authentication. How the integration gets its API token is open question 1 of the server foundation plan.
- One config entry per server.

**Devices.** In the preview, cameras are added in the app's own interface. The integration creates one device per Camera Assistant camera, with a `configuration_url` that opens that camera in the app's interface. The same camera may also exist as a device of its own integration (Matter, Reolink, ONVIF). Linking the two is part of the MVP hand-over (see "Later").

**Entities per camera.**

| Entity | Platform | What it shows |
|---|---|---|
| Events | `event` | Fires for each event of that camera. Event types: `motion`, `doorbell`, `clip_requested`, `recording_started`, `recording_stopped` (the final list follows the server's event model). Attributes: event id, zone, start and end, whether a clip exists. |
| Last event | `sensor` with device class `timestamp` | The start of the last event, with its type and zone as attributes. |

The server serves a still per event, made when it is first asked for (recorder plan). Whether the last event also gets an `image` entity with that still is an open question.

The integration fills the entities from the client library's subscription. After a reconnect it reads the last event per camera once, so the state is right even when events were missed.

**The `record_clip` action.**

- Domain action `camera_assistant.record_clip` (the domain name follows the project's final name).
- Target: one or more Camera Assistant camera devices.
- Fields: `duration` (seconds, required), `pre_roll` (seconds, optional; default the camera's pre-roll setting, capped at what the server's ring holds), `label` (optional text stored with the event).
- Response (optional): the id of the event the clip belongs to, so an automation can send it on.
- For an RTSP camera the server cuts the clip from its ring and the live stream. For a Matter camera it triggers the camera's upload (`ManuallyTriggerTransport`).
- Errors are raised as `HomeAssistantError` with a translated message: camera unknown, camera not recording, duration out of range.

**Media browser.**

- A `media_source` platform with this tree: Camera Assistant, then cameras, then days, then events and continuous footage per hour.
- Each event item has its title (type, zone, time) and a thumbnail when the server has one.
- Resolving an item asks the server for a playback URL: HLS over the stored CMAF segments, or MP4 for a single clip. The URL is short-lived and signed by the server.
- Video path. When the client can reach the app directly, the URL points at the app and no video passes through Core. When it cannot, for example away from home, the preview needs a way that works. Options are in the open questions; the integration does not convert or store video in any case.
- When the recording provider platform lands in Home Assistant Core (MVP), the integration becomes its first provider and the Media browser tree comes from that platform. The client library calls stay the same.

**Labs.** In the preview the integration is a Labs feature: it declares a preview feature in its `manifest.json`, a household switches it on in Home Assistant's settings, and only then can the integration be set up. The exact gate (the config flow refuses to start, or the integration loads but stays idle) is agreed with the Core maintainers.

**Settings live in the app.** In the preview the integration has no options flow for recording modes, retention or storage. The device's `configuration_url` and a link in the integration's page open the app's interface.

### Data shapes

The event as the integration sees it, from the client library:

```
Event {
  id, camera_id,
  type: motion | doorbell | clip_requested | continuous | ...,
  zone,                               // optional
  started_at, ended_at,               // ended_at empty while the event runs
  has_clip, has_still,
  label,                              // optional, from record_clip
}
```

The playback answer:

```
Playback {
  kind: hls | mp4,
  url,                                // signed, short-lived
  expires_at,
  mime_type,
}
```

## Decisions

1. **Follow the pattern of the Matter Server, Z-Wave JS and Music Assistant.** A thin Python client library, a WebSocket API with a schema version, and a config flow that installs the app. Maintainers of Home Assistant know this pattern, and it keeps all video work in the app.
2. **The integration holds no video and never converts it.** Video goes from the app to the viewer. Core only passes on a URL.
3. **The app never gets credentials for Home Assistant.** The integration connects to the app, not the other way round.
4. **In the preview, settings live in the app's own interface.** That avoids building them twice while the design settles. Home Assistant shows status and links to the app.
5. **The preview uses `media_source` directly.** The recording provider platform is proposed through Home Assistant's architecture process and comes later. The client library is designed so the switch does not change it.
6. **The integration is meant for Home Assistant Core, not as a custom integration.** It is a Labs feature in the preview.

## Steps

1. **[hass-integration] Client library: connection, schema and models.** Package skeleton, WebSocket connection, server info, schema check, reconnect, typed errors, the models. Decide how the models follow the server's schema.
2. **[hass-integration] Client library: commands and subscriptions.** The commands and events of "The client library". Tests against a fake server. First release on PyPI.
3. **[hass-integration] Integration: config flow that installs and starts the app.** `AddonManager` on Supervisor installs, a URL elsewhere, schema check, one entry per server, translations, tests at Home Assistant's standard.
4. **[hass-integration] Integration: devices and event entities.** One device per camera with `configuration_url`, the `event` entity and the last-event sensor, state restored after a reconnect.
5. **[hass-integration] Integration: the `record_clip` action.** Action schema, translated errors, optional response with the event id.
6. **[hass-integration] Integration: recordings in the Media browser.** The tree, the thumbnails, resolving to a signed URL. The video path from the decision on the open question.
7. **[hass-integration] Integration: Labs feature.** Shown as a preview in Home Assistant, in the way agreed with the Core maintainers.
8. **[documentation] Docs.** The integration page on home-assistant.io, and a page in the project docs on using Camera Assistant with Home Assistant.

## Order

- Step 1 needs the first version of the server's API schema. Step 2 needs step 1.
- Steps 3 to 6 need step 2. Steps 4, 5 and 6 can run in parallel after step 3.
- Step 5 needs `record_clip` in the server. Step 6 needs the server's playback URLs.
- Step 7 can be discussed with the Core maintainers at once, and is built after step 3.
- Step 8 comes with the pull request to Home Assistant Core.

## Open questions

1. **Remote playback in the preview.** Away from home the client cannot reach the app directly. Options: (a) a plain pass-through in the integration, with no conversion and no temporary files, as the last resort; (b) WebRTC playback from the app, signalled through Home Assistant, which needs work in the server and in Home Assistant's frontend; (c) the Media browser works at home only in the preview. Proposal: (a) for the preview, (b) with the provider platform.
2. **Where the client library lives and what it is called.** Options: its own repository in the `camera-assistant` organisation, or a folder in the server's repository. A repository of its own matches Music Assistant's client.
3. **Keeping the models in sync with the server.** Options: generate them from a schema file the server publishes, or write them by hand with a test that checks them against the server's schema.
4. **The integration's domain and name.** Camera Assistant is a working title. Inside Home Assistant it may show as "Cameras" or "Recordings".
5. **An image entity for the last event.** The recorder plan makes a still per event on demand, so the Media browser can show thumbnails. Should the last event also get an `image` entity, which asks for a still on every event?
6. **How a whole integration is gated by Labs.** Options: the config flow refuses to start until the preview feature is on; or the integration can be set up but does nothing until it is on. A new integration in Core also goes through the usual review, so the maintainers' view decides.
7. **The API address on Supervisor.** Is the API reached on the app's host name inside the Supervisor network, or through a port on the host? The app's packaging decides.

## Later, and what this must not block

**In the MVP:**

- **Cameras handed over from Home Assistant.** The integrations that already know a camera (Reolink, ONVIF, Matter, generic) pass its stream addresses, credentials and trigger entities to the app through its management API. The camera keeps one owner: its own integration. Camera Assistant's entities then attach to that camera's device, and a camera is never a device twice.
- **A recording switch per camera** in Home Assistant. All other settings stay in the app's interface.
- **Repairs**: a full or missing disk, an app that is too old for the integration, a camera that stopped recording.
- **Recording follows the camera's own events** from its integration (motion, person, doorbell) without an automation.
- **The app's interface as a panel** in Home Assistant.
- **The integration as the first provider** of the recording provider platform in Core.
- **A clip in a notification.**
- **Live view for RTSP cameras** from the app, through Home Assistant's WebRTC provider hook.

**After the MVP:** dashboard cards that reuse the app's player and timeline, and the optional parts of the provider platform where Home Assistant's own views need them.

Nothing in the preview should block these: the client library keeps camera ids stable, the event model is the one the provider platform uses, and no setting is stored in Home Assistant that later has to move.
