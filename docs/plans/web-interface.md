# A web interface to add cameras, watch events and set up recording

Status: draft plan for the Preview phase. Epic: 08. This is the technical companion of the epic on the project board. It is written so a developer or a coding agent can start on a step without asking what was meant.

Terms used here:

- **The interface**: the web interface in `camera-assistant/frontend`.
- **The server**: the Camera Assistant server, which serves the interface's files and its API (WebSocket and HTTP, with a versioned schema).
- **Components**: the reusable web components of the interface, first of all the player.

## Problem

In the preview, Camera Assistant is set up in its own interface, not in Home Assistant. Without that interface a household cannot:

- add an RTSP camera by its address, or pick a Matter camera;
- choose what to record, how long to keep it and where;
- see what was recorded and play it back;
- see whether every camera is recording and how full the disks are.

The interface also has a longer life. From the MVP it is shown as a panel in Home Assistant, and later Home Assistant's dashboard cards reuse its player and timeline. The choices made now decide whether that works without a rewrite.

## What exists

**In Home Assistant.** Home Assistant's frontend ([home-assistant/frontend](https://github.com/home-assistant/frontend)) is written in TypeScript with Lit web components. Themes are CSS custom properties (`--primary-color`, `--card-background-color` and others), and a custom panel or card is a web component that receives the `hass` object. Apps show their interface through Supervisor ingress, under a path prefix on Home Assistant's own address; ingress streams every response through Core, video included.

**In Music Assistant.** Music Assistant's interface ([music-assistant/frontend](https://github.com/music-assistant/frontend)) is its own web app, served by the Music Assistant server and shown in Home Assistant as a panel. It is the pattern this project follows: one interface, built once, usable on its own and inside Home Assistant.

**Prior art for camera views.** The Advanced Camera Card ([dermotduffy/advanced-camera-card](https://github.com/dermotduffy/advanced-camera-card)) is a Home Assistant card with live view, an event gallery and a timeline over Frigate and other systems. Frigate's interface ([blakeblackshear/frigate](https://github.com/blakeblackshear/frigate)) has review items, a timeline and exports. Both are references for what a household expects later, not for the preview's scope.

**In Cairn.** [Cairn](https://github.com/bbangert/cairn-nvr) has a server-rendered interface with live view and event clips. It is not reused here, because this interface must be built from web components that Home Assistant can reuse.

## Concept

### Layout of the repository

```
camera-assistant/frontend
  packages/
    api/          typed client for the server's API (TypeScript)
    components/   reusable web components: the player now, the timeline later
  app/            the interface: pages, navigation, settings
```

- `api` and `components` have no dependency on `app`. `components` depends on nothing but `api` types and its media libraries. That is what makes them reusable by Home Assistant later.
- The server serves the built `app` as static files. All URLs in the build are relative, so the interface also works under an ingress path prefix.

### Technology

- TypeScript and Lit, the same as Home Assistant's frontend, so its developers can work on both and its cards can host the components.
- A build tool, a test runner and a linter, picked in step 1. Vite is worth evaluating.
- HLS playback with the browser's own player where it has one (Safari), and with `hls.js` elsewhere. `hls.js` is worth evaluating against the CMAF segments the server serves.
- All text in one translation file, English first. Dates and times in the browser's locale and time zone.

### The API client (`packages/api`)

- Connects to the server's WebSocket, sends commands, receives events, reconnects.
- Types follow the server's versioned schema. How they stay in sync (generated from a schema file, or checked by a test) is decided with the Python client library, so both work the same way.
- Gives the app and the components the same objects: camera, event, recording span, playback URL, storage location, health.

### Pages

**Cameras.**

- A list of cameras with name, kind (RTSP or Matter), recording mode and state (recording, idle, error).
- *Add an RTSP stream by address.* Fields: name, main stream URL, optional substream URL, user name and password. A "test connection" button asks the server to open the stream and shows codec, resolution and audio, or the error. The password is never shown again after saving.
- *Pick a Matter camera.* The interface lists the camera endpoints the server sees through the Matter Server, with whether each can upload recordings (Push AV). Picking one asks the server to set it up, and shows the progress and any error. When no Matter Server is connected, the page says how to connect one (its address is a setting).
- Edit the name and stream addresses; remove a camera, with a question whether to keep its recordings.

**Events.**

- A list of events, newest first, for all cameras or one. Filters: camera, day, type (motion, doorbell, clip on request, continuous).
- A row shows camera, type, zone, start, length, and a still when the server has one.
- Opening an event plays it in the player, with a download button for the MP4.
- Continuous footage is listed per camera and per hour. Scrubbing across hours comes with the timeline, later.

**Settings.**

- Per camera: recording mode (off, on events, continuous) and pre-roll.
- Retention: keep for N days and at most N GB. Emergency cleanup when a disk fills up is automatic and not a setting.
- Storage locations: one local and one network location in the preview, chosen from the locations the server reports (the data folder, and the mounts it is given). The interface shows free space and whether the server can write there. Mounts themselves are created outside the app: in Docker, or in Home Assistant's storage settings.
- The address of the Matter Server, when the server cannot find it by itself.

**Health.**

- Per camera: connected or not, recording or not, the last segment written, the last error.
- Per storage location: used and free space, write errors.
- The Matter Server connection, the server's version and uptime.

### Navigation and links

- One route per page and per object: `/cameras`, `/cameras/<id>`, `/events`, `/events/<id>`, `/settings`, `/health`.
- A link to an event, or to a camera at a moment (`/cameras/<id>?t=<time>`), must keep working. Home Assistant will link to them from the camera view and from notifications later.

### The player (`packages/components`)

- A web component, for example `<ca-player>`; the prefix is open.
- Input: a playback answer from the server (HLS or MP4, signed URL) or a camera and a time. Output: events for play, pause, time, end and error.
- Controls: play and pause, seek within the item, full screen, download.
- When the browser cannot play the codec (H.265 in some browsers), the player says so instead of showing a black picture.
- No dependency on the app's pages or on Home Assistant. A demo page and tests run it on its own.
- Live view is added to the same component in the MVP, so going from live into recorded video stays in one player.

### Look and feel

- In the preview the interface is opened on its own address or through Home Assistant's "Open web UI" for the app.
- It follows Home Assistant's design language: the same colour roles, light and dark mode, the same density and spacing. Colours are CSS custom properties with the names Home Assistant uses, so the panel in the MVP can take over the user's theme.
- Pages work on a phone and on a desktop.

## Decisions

1. **TypeScript web components with Lit.** Home Assistant's frontend uses the same, so the panel feels native and the cards can reuse the player and the timeline.
2. **Reusable parts live in their own packages** (`api`, `components`) with no dependency on the app. Reuse by Home Assistant is then a matter of publishing, not of untangling.
3. **The server serves the interface.** The interface works without Home Assistant, and inside it through ingress, from the same build.
4. **Settings in the preview are the minimum:** recording mode, pre-roll, retention, storage locations, the Matter Server address. Technical tuning per camera is not offered.
5. **Design-heavy work starts from a design.** Pages labelled `Needs UX Design` get a mock-up that is agreed before they are built.
6. **Stable links from the start,** so later links from Home Assistant need no change.

## Steps

1. **[frontend] Project setup.** On the frontend baseline of the project setup plan (its four scripts, CI and release archive): repository layout with `packages/api`, `packages/components` and `app`; Lit, and the build tool, test runner and linter behind the four scripts; relative URLs; the translation file; the server serves the build.
2. **[frontend] API client.** WebSocket connection, commands, events, reconnect, types that follow the server's schema.
3. **[frontend] App shell, navigation and theme (Needs UX Design).** Navigation between pages, the routes and deep links, light and dark mode with Home Assistant's colour roles, layout for phone and desktop, a sign-in page when the server asks for one, and the page that creates the first user on first start.
4. **[frontend] Cameras: list and RTSP streams (Needs UX Design).** The list, adding an RTSP stream with a connection test, editing and removing.
5. **[frontend] Cameras: pick a Matter camera (Needs UX Design).** The list from the Matter Server, setup progress and errors, the state when no Matter Server is connected.
6. **[frontend] The player component.** HLS and MP4 playback, the controls, codec check, a demo page, tests.
7. **[frontend] Events and playback (Needs UX Design).** The list with filters, playing an event, continuous footage per hour, download.
8. **[frontend] Settings (Needs UX Design).** Recording mode and pre-roll per camera, retention, storage locations, the Matter Server address.
9. **[frontend] Health.** Cameras, storage and the Matter Server connection.
10. **[documentation] Docs.** A short guide to the interface in the project docs, with screenshots of made-up cameras.

## Order

- Steps 1 and 2 come first. Step 1 needs the frontend baseline of the project setup plan. Step 2 needs the first version of the server's API schema.
- Step 3 needs step 1. Its design can start at once.
- Steps 4, 5, 7, 8 and 9 need steps 2 and 3. Step 5 needs the server's Matter Server client. Step 7 needs step 6.
- Step 6 needs step 1 and the server's playback URLs. It can be built against a static HLS test stream before the server serves recordings.
- Step 10 comes last.

## Open questions

1. **Sign-in when used without Home Assistant.** The server foundation plan gives the server its own users with a password and API tokens, and creates the first user on first start in this interface; inside Home Assistant, ingress signs the user in. Open: how the interface keeps a session (a token in browser storage or a cookie), and what the first-start page asks.
2. **Video through ingress.** Ingress streams every response through Home Assistant Core. Should the player load video straight from the app's own address when the browser can reach it, and fall back to ingress otherwise? This is the same question as remote playback in the Home Assistant integration plan.
3. **Home Assistant's components.** Use only Home Assistant's colour roles and design rules, or also reuse its components? Its components are not published as a separate package today, as far as we know; to check with the Frontend maintainers.
4. **Theme inside Home Assistant.** How does the panel get the user's theme: through ingress from the page around it, or from the integration that registers the panel? Decided with the panel in the MVP, but the CSS custom properties are named for it now.
5. **The component prefix and package names**, for example `ca-` and `@camera-assistant/components`. They follow the project's final name.
6. **Translations.** Which platform the community uses to translate, and when to start.

## Later, and what this must not block

**In the MVP:**

- **The interface as a panel in Home Assistant**, with Home Assistant's sign-in and theme, registered by the integration.
- **Live view in the player** for RTSP cameras served by the app. Matter cameras keep their own WebRTC session in the same player.
- **Detection status** per camera (the detection plan offers no detection settings) and event labels (person, pet, vehicle, package) in the events list.

**After the MVP:**

- **The timeline with scrubbing**, as a reusable component next to the player.
- **Several cameras side by side.**
- **Export** of a time range as a file.
- **Zones drawn in the interface.**
- **Full storage rules**: several locations, what goes where and for how long.
- **Dashboard cards in Home Assistant** that reuse the player and the timeline.

The package layout, the stable routes and the player's input (a playback answer or a camera and a time) are chosen so none of these needs a rewrite.
