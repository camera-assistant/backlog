# Recordings from every recording system in Home Assistant

Status: draft plan for the MVP phase. Epic: 11. This is the technical companion of the epic on the project board. The work itself happens in Home Assistant's repositories and goes through Home Assistant's architecture process; this plan is the starting point for that proposal, not a decision on Home Assistant's behalf.

Terms used here:

- **Recording system**: anything that records cameras and can show what it recorded: Camera Assistant, Frigate, a Reolink NVR or hub, UniFi Protect, a cloud camera service.
- **Provider**: a recording system as Home Assistant sees it, through its integration.
- **Media gateway**: a process outside Core that turns a video source a browser cannot play into one it can. Today go2rtc has that role for live view.
- **Core**: Home Assistant Core, the Python process.

## Problem

Home Assistant has no common way to show recordings.

- Each integration builds its own tree in the Media browser. The Media browser has no time range query, no filter by label or zone, no event metadata and no idea of continuous footage.
- The video of recordings runs through Core. The Frigate integration proxies HLS through Core. UniFi Protect streams MP4 through Core. The Reolink integration downloads recordings, converts them with ffmpeg and serves them from a temporary folder in Core ([core#192887](https://github.com/home-assistant/core/pull/192887)). Core's own `camera.record` writes MP4 files inside Core's process, for at most 3600 seconds, one recording per camera, with no way to stop it.
- Automations get events from each integration in its own shape.
- A card or a view that wants to show "what happened on this camera" has to know every recording system. The Advanced Camera Card does exactly that, in the frontend, because Core offers no such interface.

Camera Assistant needs a place in Home Assistant for its recordings. Building that place for Camera Assistant alone would repeat the problem. A generic platform serves Camera Assistant first and every other recording system after it.

## What exists

**In Home Assistant Core.**

| Part | What it does | What it lacks for recordings |
|---|---|---|
| `media_source` | Integrations offer a tree of folders and items; an item resolves to a URL and a MIME type. | No time range, no filters, no event metadata, no index. Every integration returns a URL on Home Assistant and passes the bytes through Core. No delete: [architecture#1282](https://github.com/home-assistant/architecture/discussions/1282) proposes one. |
| Backup agents (`backup/agent.py`) | A platform shape: integrations register agents through `async_get_backup_agents(hass)`; an agent lists, uploads, downloads and deletes whole files. | A good model for registering providers. Not a data path for video: whole files through Core, no range reads, no appends. |
| `camera` | Live view through `stream_source()`, go2rtc for WebRTC and `stream` for HLS. `CameraEntityFeature` has `ON_OFF` and `STREAM`. | Nothing about recorded history. [architecture#1528](https://github.com/home-assistant/architecture/discussions/1528) proposes a still image at a past time. |
| go2rtc | Bundled with Home Assistant, registered as a WebRTC provider. Signalling goes over Home Assistant's WebSocket; media goes between browser and go2rtc. | Used for live view only. Whether it can play a recorded source with seeking is not verified. |
| `stream` | HLS for live view, `camera.record`. | Maintainers want its functions moved to go2rtc so it can be shut down ([architecture#1417](https://github.com/home-assistant/architecture/discussions/1417)). |
| `event` entities | Fire an event with a type and attributes. | One per camera would serve automations; nothing creates them for recordings in a common way. |
| Supervisor ingress | Shows an app's interface inside Home Assistant. | Streams every response through Core, so it is not a way around Core for video. |

No architecture discussion proposes a generic recordings platform or a timeline. The open camera discussions are about sources per camera, go2rtc first, and a still at a past time.

A draft pull request, [core#192862](https://github.com/home-assistant/core/pull/192862), adds `camera.record` for WebRTC cameras inside Core. It shows the demand, and the limit of doing it in Core: a long recording occupies a worker of Core's shared executor.

**What recording systems offer.** A survey of about twenty systems (Frigate, Reolink, UniFi Protect, Synology Surveillance Station, Scrypted, Blue Iris, Agent DVR, Shinobi, ZoneMinder, Viseron, Moonfire NVR, Kerberos, ONVIF Profile G, TP-Link VIGI and Tapo, Ring, Nest, HomeKit Secure Video):

- *Almost all* can list recordings or events for a camera in a time window, play an item as some URL (MP4, HLS or RTSP replay), and give an event a start, a camera, a type and often a still.
- *Many, not all*: continuous footage on a timeline, seeking to any time, delete, protect, labels beyond motion, zones, a still at a past time, a summary per day.
- *Rare*: changing retention or storage through an API.
- *Playback formats differ*: Frigate, Moonfire and Viseron serve HLS or MP4 with ranges. Reolink, ONVIF Profile G and VIGI offer RTSP replay or proprietary downloads, which no browser plays. UniFi Protect exports MP4 through a private API.

**Prior art for the interface.**

- The Advanced Camera Card's engine interface ([engine.ts](https://github.com/dermotduffy/advanced-camera-card/blob/main/src/camera-manager/engine.ts)): queries for events, recordings and recording segments by camera, time, label and zone; capabilities such as `clips`, `recordings`, `seek`, `favorite-events`; an event stream with new, update and end. It abstracts Frigate, Reolink, motionEye, TP-Link and any camera entity, in the frontend.
- Scrypted's SDK ([types.py](https://github.com/koush/scrypted/blob/main/sdk/types/scrypted_python/scrypted_sdk/types.py)): `VideoRecorder.getRecordingStream(startTime)`, `getRecordingStreamThumbnail(time)`, `VideoClips.getVideoClips`, `EventRecorder.getRecordedEvents`, `VideoRecorderManagement.deleteRecordingStream`.
- Frigate's HTTP API: HLS for any time range, events, review items, previews for scrubbing.
- Moonfire NVR's API ([api.md](https://github.com/scottlamb/moonfire-nvr/blob/master/ref/api.md)): recordings as time ranges, any sub-range as MP4 with HTTP Range.

## Concept

Home Assistant gets a recording provider platform: one interface that any recording system implements, and a thin layer of user interface on top. Home Assistant stays a client. It does not record, store, convert or run detection.

```
          Home Assistant Core                                 recording systems
  ---------------------------------------                  ----------------------
  Media browser   event entities   cards (later)
         \             |             /
          recording provider platform  <-- provider -->  Camera Assistant
          (events, spans, playback handles) <-- provider -->  Frigate, Reolink, ...
                         |
                 playback handle (never video)
                         |
  browser / mobile app  <======== video =========  provider, or media gateway
```

### Concepts

| Concept | Meaning |
|---|---|
| **Provider** | A recording system, represented by its integration. One integration can register several (one per Frigate instance, one per NVR). |
| **Camera mapping** | Which Home Assistant camera, by device or entity, each of the provider's cameras belongs to. Recordings then show on the camera a person already has. A camera can have recordings in more than one provider. |
| **Event** | Something that happened on a camera: start, end, type and labels (motion, person, doorbell), zone, a still, a clip, protected or not. A stable id across updates. |
| **Recording span** | A stretch of continuous footage that exists for a camera. |
| **Playback handle** | Where and how to play an event or a time range: either formats the client can play, or a source for the media gateway. Never the video itself. |
| **Capabilities** | Flags that say which optional parts a provider supports. |

### The interface

| Part | Required | What it does |
|---|---|---|
| **List events** | Yes | Events for cameras in a time window, filtered by label, paged. |
| **Get playback** | Yes | A playback handle for an event or a time range. |
| **Get thumbnail** | Yes | A still for an event. |
| **Push events** | Optional | New and updated events as they happen, for automations and notifications. |
| **Timeline** | Optional | Recording spans, so a view can draw continuous footage. |
| **Seek and preview** | Optional | Jump to an absolute time; small stills while scrubbing; a still at a past time (in line with architecture#1528). |
| **Manage** | Optional | Delete, protect, export or download. |
| **Configure** | Optional | Recording mode, retention and storage rules, so Home Assistant's interface can manage the provider. |
| **Storage status** | Optional | Used and free space per location. |
| **Link to the provider's interface** | Optional | An address that opens the provider's own interface at a camera and a moment. |

The required set is what nearly every recording system can serve. Configuration over an API is rare, so it is optional.

A first sketch in Python, for the proposal. Names are placeholders.

```python
class RecordingProviderFeature(IntFlag):
    PUSH_EVENTS = 1
    TIMELINE = 2
    SEEK = 4
    PREVIEW = 8
    MANAGE = 16
    CONFIGURE = 32
    STORAGE_STATUS = 64
    PROVIDER_LINK = 128


class RecordingProvider(abc.ABC):
    domain: str
    name: str
    unique_id: str
    supported_features: RecordingProviderFeature

    async def async_get_cameras(self) -> list[ProviderCamera]: ...
    async def async_list_events(self, query: EventQuery) -> EventPage: ...
    async def async_get_playback(self, request: PlaybackRequest) -> PlaybackHandle: ...
    async def async_get_thumbnail(self, event_id: str) -> Thumbnail: ...
    # optional, by feature flag
    def async_subscribe_events(self, callback: Callable[[RecordingEvent], None]) -> CALLBACK_TYPE: ...
    async def async_list_spans(self, camera_id: str, start: datetime, end: datetime) -> list[RecordingSpan]: ...


# <integration>/recording.py, as backup agents do
async def async_get_recording_providers(hass: HomeAssistant) -> list[RecordingProvider]: ...
```

Data shapes:

```
ProviderCamera  { camera_id, name, device_id | entity_id }
EventQuery      { camera_ids, start, end, labels?, limit, cursor? }
RecordingEvent  { id, camera_id, start, end?, labels[], zones[], has_thumbnail, has_clip, protected, extra{} }
RecordingSpan   { camera_id, start, end }
PlaybackRequest { event_id } | { camera_id, start, end? }
PlaybackHandle  { options: [PlaybackOption] }   // in order of preference
PlaybackOption  { kind: hls | mp4 | webrtc, url?, expires_at, seekable, start, end }
                | { kind: gateway_source, source_url, source_kind: rtsp_replay | http_file | ... }
```

### Where the video flows

Core hands out a playback handle and stays out of the video path. Core never converts video and never writes it to a temporary file. There are two cases.

**1. The provider serves a format the client can play** (Camera Assistant, Frigate, Moonfire). The handle lists the ways to play, and the client takes the first one that works:

- **HLS or MP4 straight from the provider,** on a short-lived signed address, when the client can reach it. Seeking works as in any video player.
- **WebRTC from the provider,** signalled through Home Assistant like live view, when the client cannot reach the provider, for example away from home. Core carries only the signalling.
- **A plain pass-through in Core,** as the last resort: no conversion, no temporary files. Whether Core accepts this at all is a question for the proposal.

**2. The provider serves something the client cannot play** (RTSP replay from ONVIF Profile G or VIGI, a Reolink hub's own format). The handle carries a source, and a **media gateway** outside Core turns it into WebRTC or HLS. An integration then only hands over a source and never converts in Core.

| Situation | Media gateway |
|---|---|
| Camera Assistant is not installed | The bundled go2rtc, as for live view today. |
| Camera Assistant is installed | Camera Assistant, for live view and for recordings of other systems. It already holds the connection to its cameras. go2rtc stays as the fallback for sources Camera Assistant does not handle. |

The take-over goes in steps. In the MVP, Camera Assistant serves live view for its own cameras. Playback for other recording systems through a media gateway follows after the MVP. go2rtc is not removed, only used less.

Access stays with Home Assistant: a user gets a playback handle only for a camera they may see. The provider checks the signature or credential in the handle.

### What a person sees

- **Recordings in the Media browser,** for every recording system, by camera, day and event.
- **An event entity per camera** for automations, and notifications with a still or a clip.
- **A link from the camera** to the full interface of its recording system. For Camera Assistant that is its panel with the timeline.
- **Dashboard cards** with a camera and its latest events, after the MVP.
- **No second timeline in Home Assistant.** The timeline lives in the recording system's own interface. Later, where it proves its worth, a timeline in Home Assistant's camera view can reuse Camera Assistant's components.

## Decisions

1. **Home Assistant never stores or converts video.** It answers questions about recordings and passes on playback handles. The video goes from the recording system, or a media gateway, to the viewer.
2. **Recording systems and storage locations are two different things.** Frigate or a Reolink hub is a recording system that Home Assistant reads from: this platform. A NAS or a cloud bucket is a storage location that a recorder writes to: that stays inside the recorder. Backup agents are a model for the setup experience, not for the data path.
3. **The required set is the lowest common denominator,** so almost every system can be a provider. Everything else is optional and announced by flags.
4. **Configuration is optional.** Most systems cannot be configured over an API, and Camera Assistant keeps its settings in its own interface.
5. **No second timeline in Home Assistant.** Media browser, event entities and cards; the full interface belongs to the recording system.
6. **The platform has the shape of backup agents and media sources:** an integration platform file that registers providers.
7. **Camera Assistant is the first provider, as the reference.** Providers for other systems are for their integrations' maintainers and the community. The interface is checked against Frigate, Reolink and UniFi Protect while it is designed.
8. **The proposal goes through Home Assistant's architecture process.** Everything in this plan can change there.

## Steps

1. **[documentation] Check the interface against existing systems.** For Frigate, Reolink, UniFi Protect, Moonfire NVR, ONVIF Profile G and one cloud service: which part of the interface each can serve, and with which playback option. A table in `docs/research/recording-provider-fit.md` in the backlog.
2. **[documentation] Write the architecture proposal.** A discussion in [home-assistant/architecture](https://github.com/home-assistant/architecture/discussions) with the concepts, the interface, the video flows, the media gateway role and what a person sees. Answer the review there and update this plan.
3. **[hass-integration] Core: the platform and its models.** The platform file, provider registration and a listener for changes, the models, the camera mapping through the device and entity registries. Tests with a demo provider.
4. **[hass-integration] Core: WebSocket API for the frontend.** List events, get playback, get a thumbnail, and the optional timeline, for a user with access to the camera.
5. **[hass-integration] Core: recordings in the Media browser.** One media source over all providers: camera, day, event. Resolving an item returns a playback handle.
6. **[hass-integration] Core: events for automations.** An event entity per mapped camera, fed by the providers' pushed events.
7. **[hass-integration] Core: playback handles and access.** Signed short-lived handles for direct HLS or MP4; the pass-through as the last resort, if the proposal keeps it.
8. **[multi] Core and frontend: WebRTC playback of recordings (after the MVP).** Signalling for "camera at a time" through Home Assistant, as for live view, for providers that offer it. Seeking over WebRTC is part of the design.
9. **[frontend] Frontend: play a playback handle.** In the Media browser and the camera's dialog: pick the first option the client can use, HLS or WebRTC.
10. **[frontend] Frontend: link from the camera to its recording system.** For providers with the link feature.
11. **[hass-integration] Camera Assistant as the first provider.** The Camera Assistant integration implements the platform; its own Media browser tree from the preview is replaced.
12. **[documentation] Developer docs.** A page on developers.home-assistant.io on how to write a provider.
13. **[multi] The media gateway role (after the MVP).** A source in a playback handle goes to go2rtc, or to Camera Assistant when it is installed, and comes back as WebRTC or HLS.

## Order

- Step 1 comes before step 2. Step 2 is approved before any code in Core.
- Step 3 comes first in Core. Steps 4, 5, 6 and 7 need it and can run in parallel.
- Step 8 needs step 4 and a provider that offers WebRTC playback (Camera Assistant). Step 9 needs steps 4 and 7.
- Step 10 needs step 3.
- Step 11 needs steps 3 to 7 and the Camera Assistant integration.
- Step 12 comes with step 3 and is updated as the rest lands.
- Step 13 comes after the MVP.

## Asks to the Home Assistant maintainers

**Core:**

- Review the architecture proposal, and agree on the platform's name and domain. `recorder` is taken by Home Assistant's history database, so it cannot be that.
- Agree that Core does not convert video and does not keep it in temporary files, and decide whether a plain pass-through is allowed as the last resort.
- Decide how the platform relates to `media_source`: a media source over all providers, alongside the existing ones, or a change to `media_source` itself (including delete, architecture#1282).
- Decide whether the platform or each integration creates the event entities.
- Agree how a playback handle is secured, also with remote access, and how WebRTC playback of recordings is signalled, next to the existing WebRTC provider hook for live view.
- Agree the media gateway role, with go2rtc as its default implementation, in line with the go2rtc-first direction of architecture#1417.

**Frontend:**

- Play a playback handle in the Media browser and the camera's dialog (HLS or WebRTC).
- A link from the camera to the recording system's own interface.
- Agree that there is no second timeline in Home Assistant, and that later cards reuse Camera Assistant's player and timeline as web components.

## Open questions

1. **Entity platform or not.** Events fit event entities; listing and searching do not fit entity state. Proposal: a platform without entities, like backup agents, plus event entities for automations.
2. **The event schema.** How much of Frigate's model (labels, sub-labels, zones, score, review grouping) is generic, and how much goes in `extra`?
3. **Several providers for one camera** (a Reolink camera's own card and Frigate): one merged list, or one provider at a time?
4. **Remote playback.** WebRTC from the provider is the proposal. It needs a way to seek (renegotiation or a data channel), and go2rtc's support for recorded sources and seeking is not verified.
5. **Time.** Provider clocks, the time zone for day folders, daylight saving time.
6. **Permissions.** Home Assistant has no per-camera access rights today; discussions about access control are in progress. Recordings are more sensitive than live view.
7. **Configuration owner.** When a provider supports the configure part, does Home Assistant write recording modes and retention, or only show them and link to the provider? Who wins on a conflict?
8. **The past still.** Does the provider's "still at a past time" become the implementation of the camera proposal in architecture#1528, or stay separate?

## Later, and what this must not block

- **The optional parts** (timeline, seek and preview, manage, configure, storage status), where Home Assistant's own views need them.
- **Dashboard cards** that show a camera with its latest events, reusing Camera Assistant's player.
- **A timeline in Home Assistant's camera view,** reusing Camera Assistant's component, if it proves its worth.
- **Providers for Frigate, Reolink, UniFi Protect and others,** added by their maintainers and the community. The required set is kept small so they can.
- **Camera Assistant as media gateway** for other recording systems, and the step-by-step move of `stream`'s functions to go2rtc and Camera Assistant.
