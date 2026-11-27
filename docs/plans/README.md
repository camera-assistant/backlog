# Plans

Technical companions of the epics on the Camera Assistant Project Board. Each epic links its
plan here so the reasoning is public and the document can be fed to tooling.

Start with the [architecture overview](../architecture.md).

| Plan | Phase | What it covers |
| --- | --- | --- |
| [Project setup](project-setup.md) | Preview | Repositories, CI, contribution rules, instructions for coding agents, releases |
| [Server foundation](server-foundation.md) | Preview | Process model, camera registry, the API, sign-in, health |
| [RTSP ingest and the ring](rtsp-ingest-and-ring.md) | Preview | One connection per camera stream, kept in memory for pre-roll and viewers |
| [Recorder and playback](recorder-and-playback.md) | Preview | Segments, the index, events, retention, HLS and MP4 |
| [Storage locations](storage-locations.md) | Preview | Local and network locations, checks, background copies, the connector interface |
| [Matter cameras](matter-cameras.md) | Preview | Recording through Push AV, with the Matter Server |
| [Home Assistant integration](home-assistant-integration.md) | Preview | The Python client library and the integration |
| [Web interface](web-interface.md) | Preview | Cameras, events, settings and health |
| [Detection](detection.md) | MVP | The module contract, the first modules, how detection scales by itself |
| [Live view](live-view.md) | MVP | WebRTC from the server, as a provider in Home Assistant |
| [Recording provider platform](recording-provider-platform.md) | MVP | A generic recordings layer proposed for Home Assistant Core |
