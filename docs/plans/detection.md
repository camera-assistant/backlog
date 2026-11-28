# Local object detection that scales by itself

This is the technical companion of the epic "Local object detection that scales by itself" on the
project board. It is written so that a developer or a coding agent can start on a step without
asking what was meant. Phase: MVP.

## Problem

A camera that only reports "motion" sends a notification for every shadow, cat and passing car.
People want to know what moved: a person at the door, a car on the drive, a parcel on the step.

Some cameras tell this themselves, but many do not, and the ones that do each use their own
labels and protocols. Local object detection fills the gap. It is also the most expensive thing
the server does:

- Recording only rewraps the compressed stream and is cheap. Detection needs decoded pictures and
  a neural network.
- The boxes people run differ a lot. One pass of a small model takes about 650 ms on the CPU of a
  Home Assistant Green (as reported by Cairn) and well under 50 ms on Intel graphics or an NPU.
- Accelerators need their own drivers, runtimes and model formats, and some vendor runtimes are
  closed libraries.

Existing systems solve this with a setting per hardware type, a model to pick and a container
image per accelerator. Camera Assistant has no such knobs: the server measures the host and
decides how much it can analyse.

## What exists

**In Home Assistant.**

- Integrations already expose the camera's own events as entities. The Reolink integration has
  person, vehicle, pet, face, package and visitor sensors
  (https://www.home-assistant.io/integrations/reolink/). The ONVIF integration maps motion and
  "human shape" events (https://www.home-assistant.io/integrations/onvif/).
- Matter cameras report motion per zone (Zone Management cluster). Labels for person, pet,
  vehicle and package exist in Matter only as provisional tags (Ambient Context Sensing in 1.6,
  AV Analysis in the 1.7 draft). They are not on certified cameras today.
- Home Assistant OS enables the Intel NPU driver for x86-64 boards
  (home-assistant/operating-system#4816). On the Home Assistant Green, Home Assistant OS does not
  ship an NPU driver today (see "First modules" below).

**In Cairn** (https://github.com/bbangert/cairn-nvr and https://github.com/bbangert/cairn-rockchip,
Apache 2.0). Its detection engine is donated to this project and is the starting point.

- `plugins/cairn-detect` (Rust): everything between a compressed access unit and a detection.
  Software and hardware decoding through FFmpeg 7.1 (`rsmpeg`), a motion gate, a model catalog
  (YOLOX and others), output-head decoding and NMS, a `Backend`/`Provider` seam, and a command-line
  host that takes RTP in and writes newline-delimited JSON out. It does not depend on the Elixir
  runtime.
- `plugins/cairn-ort` (Rust): an inference engine on ONNX Runtime (the `ort` crate) with a pool of
  slots. Its engine files are plain Rust; a few files carry Rustler (Elixir) types and need a thin
  rewrite.
- `cairn-rockchip` (Rust): bindings for Rockchip's NPU runtime, video decoder and scaler, loaded with
  `dlopen`, and a probe binary. Only two files import Rustler.
- Tooling (Python): model export, quantisation, parity checks and COCO scoring
  (`tools/qdq-export`, `cairn-rockchip/tools/rknn-export`).
- Ideas this project takes over:
  - Hardware profiles with a ladder of models: the most accurate model whose measured budget
    covers the number of cameras is picked.
  - Budgets come from measured runs, not from arithmetic on a latency.
  - No silent fallback to a slow path: a named backend works or reports why it does not.
  - Two capability tiers: presence (no tracker) and tracking.
  - Score-fidelity checks of quantised models on the target accelerator. Two quantisation defects
    passed every CPU check and only showed on the NPU itself.
- Measured on a QCS6490-class NPU: 8.5 to 17 ms per pass for YOLOX tiny to small
  (the evidence files in cairn-nvr's `docs/evidence/`).

**In other projects.**

- Frigate (https://github.com/blakeblackshear/frigate) has 13 detector plugins and 7 container
  images per release. The user picks the image, the detector and the decoder preset. Several
  accelerators are marked "community supported". Its `zmq_ipc` detector runs a detector out of
  process.
- ONNX Runtime (https://onnxruntime.ai/docs/execution-providers/) reaches the CPU, OpenVINO (Intel
  CPU, graphics and NPU), CUDA and TensorRT, and Qualcomm QNN through a separate plugin package. It
  does not reach Hailo or Rockchip's NPU runtime.
- OpenVINO (Intel) chooses between the CPU, graphics and NPU of an Intel host.
- Measured on an Intel N100's graphics with OpenVINO: about 15 ms (SSDLite MobileNet v2) to 30 ms
  (YOLOv9-s at 320) per pass (Frigate's hardware documentation).

## Concept

### Detection modules

A detection module is a separate process, one per kind of hardware. It serves every camera that
the server gives it.

A module is a library first: a `Board` value, which holds everything that differs per hardware
family, plus the shared module host, which runs the chain around it (see "Inside a module"). A
small binary hosts that library as a separate process. The server reaches detection only through
one trait, `Detector` (see "The server's side"), so other ways to host a module stay possible
(open question 12).

- **In:** the compressed substream of a camera, as access units with timestamps, plus the codec
  parameters.
- **Out:** detections (label, score, box, timestamp), motion results and statistics.
- **Inside the module:** decoding, scaling, the motion check and inference. They share memory
  there without copies. The server never decodes video.

Why a separate process:

- A vendor's closed library never loads into the server.
- A driver crash cannot stop a recording. It costs detections, and the server restarts the module.
- A module can be built, tested, released and owned by the people who have the hardware.
- The input is compressed video, so little data crosses the process boundary. No shared memory
  is needed.

A module has a name and a contract version. The server refuses a module with an unknown major
version.

### Inside a module

The module host runs one chain per stream:

1. **Wait for a keyframe.** Access units before the first keyframe are dropped.
2. **Decode every access unit,** also the ones that are not analysed, because a decoder needs its
   reference frames.
3. **Sample gate.** By wall clock, it admits frames at the rate that `set_mode` gives.
4. **Scale** an admitted frame to the model's input size.
5. **Motion check** on the scaled frame.
6. **Model pass,** when the mode and the motion check ask for one.
7. **Detections out,** as `motion` and `detections` messages.

A board supplies decode and scale (one stage, on its own decoder and scaler where it has them) and
the inference backend. The module host and the engine crates supply the rest: the contract, the
chain, the sample gate, the motion check, decoding model outputs into boxes, the canary and
health.

The sketches below show the shape, not final signatures. `Board` and `DecodeStage` live in the
module host crate; `Backend` and `Provider` live in the engine crates. They come from Cairn's
design (https://github.com/bbangert/cairn-nvr): `Board` is the Rust form of the
`Cairn.Stage.Board` behaviour that Cairn's boards implement in Elixir.

```rust
/// Everything that differs between hardware families, in one value.
pub trait Board: Send + Sync + 'static {
    fn name(&self) -> &'static str;

    /// Inference runtimes this board brings.
    fn backends(&self) -> &'static [Provider];

    /// Decode and scale for one stream, delivering frames at the model's input size.
    /// A refusal says why; the module host reports it.
    fn decode_stage(&self, stream: &StreamSpec, input: &InputSpec)
        -> Result<Box<dyn DecodeStage>, Unavailable>;

    /// Hardware profiles: per chip, a ladder of models with a measured budget each.
    fn profiles(&self) -> &[HardwareProfile];

    /// Files this board needs at first start, each pinned by sha256. The server fetches them.
    fn artifacts(&self) -> Vec<Artifact>;

    /// What cannot work on this host, and why, before any stream opens.
    fn preflight(&self) -> Vec<Unavailable>;
}

/// Decode and scale as one stage, because the cheapest path differs per chip.
pub trait DecodeStage: Send {
    /// Decodes every unit. Returns a frame only when `sample` is set.
    fn push(&mut self, unit: &[u8], pts: i64, sample: bool) -> Result<Option<Frame>>;
}

pub enum Frame {
    /// Pixels in host memory.
    Host(RgbFrame),
    /// A frame that stays on the device. Dropping it returns the buffer.
    Device(Box<dyn DeviceFrame>),
}
```

Cairn's decode stream (`DecodeStream` in `plugins/cairn-native`) takes access units the same
way: it decodes every unit, and the caller says per unit whether it wants a frame.

`Backend` is one inference runtime. It exists in Cairn (`plugins/cairn-detect/src/infer/backend.rs`)
and moves over as it is:

```rust
pub trait Backend: Send {
    /// Loads the model. A bad file fails here, not on the first frame.
    fn open(model: &Path, options: &SessionOptions) -> Result<Self> where Self: Sized;
    fn descriptor(&self) -> &'static Descriptor;
    /// The model's inputs and outputs.
    fn io(&self) -> &ModelIo;
    /// Infers one frame.
    fn run(&mut self, input: InputTensor) -> Result<Box<dyn Tensors + '_>>;
    /// A second context on the same model that shares its weights, where the runtime can.
    fn sibling(&mut self) -> Result<Option<Box<dyn Backend>>> { Ok(None) }
}
```

A board registers a backend through a `Provider`: its descriptor, its open function and the number
of contexts to hold, so that passes for several cameras can run at once.

Besides the chain, the module host:

- **Loads a model first in a short-lived child process (a canary),** because a vendor runtime can
  abort the whole process on a bad model file. Only a model that loaded there is opened in the
  module.
- **Lists the files it needs,** each with its sha256: models, and a vendor's runtime where it
  may not be shipped. The server fetches and checks them (see "Models").
- **Reports health** (see "Health and restarts").

### The contract, version 1

Messages over a Unix socket. The exact encoding is an open question. The shapes:

| Message | Direction | Content |
|---|---|---|
| `hello` | module → server | contract version, module name and version |
| `probe` / `probe_result` | both | devices found, decoders available (codec, maximum size), models it can run, why something is not available; per model, an optional measured budget from the module's hardware profile; the files it needs, each with its sha256 |
| `benchmark` / `benchmark_result` | both | for one model: passes per second sustained, latency p50 and p95, measured on a reference clip |
| `open_stream` | server → module | stream id, codec (H.264 or H.265), codec parameters, picture size, the model to use |
| `access_unit` | server → module | stream id, timestamp, keyframe flag, bytes |
| `set_mode` | server → module | stream id, mode (`motion`, `detect_on_motion`, `continuous`), passes per second, optional motion zones |
| `detect_window` | server → module | stream id, from, until: detect in this window whatever the mode (for a camera event) |
| `motion` | module → server | stream id, timestamp, active or not, changed area |
| `detections` | module → server | stream id, timestamp of the frame, list of `{label, score, box}` with the box in coordinates from 0 to 1 |
| `stats` | module → server | per stream: frames decoded, passes run, frames dropped, queue length, latency |
| `health` | module → server | `healthy`, `degraded` or `wedged`, with the time of completed model passes and, per stream, the age of its outstanding call; `degraded` means passes run at the speed of the CPU baseline, `wedged` means a call ran past its deadline |
| `close_stream` | server → module | stream id |

Rules:

- Labels are the tier 1 set: `person`, `pet`, `vehicle`, `package`. A module may add a finer
  label (`dog`, `cat`, `car`, `truck`) next to the coarse one. The names follow Matter's
  `IdentifiedObject` tag list, which is provisional in the spec.
- When a module falls behind, it drops frames up to the next keyframe and says so in `stats`. It
  never blocks the server. The server never waits on a module.
- A module reports a failure as a message, not as a crash, where it can.

### The server's side: the `Detector` trait

The server sees detection only through the `Detector` trait in `ca-detect-contract`. Each message
of the contract is one of its calls or events; `hello` is part of connecting. It has two
implementations: the child-process client, which speaks the contract over the socket, and the fake
module used in tests.

```rust
pub trait Detector: Send {
    async fn probe(&mut self) -> Result<ProbeResult>;
    async fn benchmark(&mut self, model: &ModelId) -> Result<BenchmarkResult>;
    async fn open_stream(&mut self, stream: StreamSpec) -> Result<()>;
    /// Never blocks. A module that is behind drops to the next keyframe itself.
    fn push(&self, stream: StreamId, unit: AccessUnit);
    async fn set_mode(&mut self, stream: StreamId, mode: Mode) -> Result<()>;
    async fn detect_window(&mut self, stream: StreamId, from: Timestamp, until: Timestamp)
        -> Result<()>;
    async fn close_stream(&mut self, stream: StreamId) -> Result<()>;
    /// The next motion, detections, stats or health event, in order.
    async fn next_event(&mut self) -> DetectorEvent;
}
```

### Health and restarts

The module host sends `health` at a fixed interval. It rests on two measurements:

- **The time of completed model passes,** without the time a frame waited in the module's queue,
  compared with a CPU baseline. An accelerator whose passes are not clearly faster than the CPU
  has fallen back to it and is `degraded`. A busy accelerator still completes its passes at
  accelerator speed, so load alone never makes a module `degraded`; the planner sees load in
  `stats`. With too few passes in an interval there is no verdict.
- **The age of each stream's outstanding decode or inference call.** A call that runs past its
  deadline makes the module `wedged`, also while its other streams still complete passes.

The planner plans without a module that is `degraded` or `wedged`, and health tells the person
why.

The module supervisor in the server restarts a module that crashes, and kills and restarts one
that is `wedged` or whose `health` does not arrive in time. When a module keeps failing, the
back-off grows steeply.
Every restart opens new sessions on the accelerator, and a runtime can keep state of old sessions
that only a reboot clears: Cairn's export tools for the QCS6490 class reboot the board after a
fixed number of sessions for that reason. So a restart does not always bring an accelerator back.
A module that comes back `degraded` stays out of the plan, and health reports it.

### Measuring the host and choosing what to do

The server decides per camera how much analysis it runs. Nobody picks a model or a rate.

**What it can tell (tiers).**

| Tier | What it does | Phase |
|---|---|---|
| 1. Presence | A person, pet, vehicle or package is there, or is gone | MVP |
| 2. Tracking | Follows objects from frame to frame and knows when one stops moving | After the MVP |
| 3. Recognition | Faces and licence plates, optional and local | Later |

**How often it looks (steps).** Per camera, the highest step the host can sustain:

1. **The camera's own events.** A Reolink person event or a Matter zone event. This costs nothing
   and works on every box.
2. **A motion check** on the substream, to decide when a closer look is worth it.
3. **Detection only when something moves**, or when the camera reports an event, at a low rate,
   to confirm and label it.
4. **Continuous detection** at a fixed rate.

**The loop.**

1. **Probe.** At start, and when a module appears or goes, the server asks every module what it
   can do.
2. **Measure.** For each module and model it runs a short benchmark on a reference clip. It
   stores the result per host, and measures again after an update of the module or the model.
   Where the module's hardware profile gives a measured budget for a model, the server uses that
   budget and runs no benchmark for it.
3. **Plan.** It chooses per camera a step, a rate and a model, so that the expected load stays
   under a share of the measured capacity. The expected load counts how often a camera has motion,
   learned from what the motion check reports. It prefers, in this order: a step for every
   camera, then a higher step, then a larger model.
4. **Watch.** When a module drops frames or its queue keeps growing, the server steps down. When
   there is room again for some time, it steps back up.
5. **Re-plan** when a camera is added or removed, or a module appears or goes.

Stepping down goes in this order: a lower rate, a smaller model, detection only on motion or
camera events, the motion check only, the camera's own events only. The person sees the result as
a status ("Detection: on motion, person and vehicle"), not as a setting.

With several modules on one host (for example the CPU and Intel graphics), the planner gives each
camera to one module and fills the fastest one first.

**Hardware profiles.** For a chip that a module knows, the module ships a hardware profile: a
ladder of models with a measured budget each, measured by the module's bench scripts on that
hardware. The module reports these budgets in `probe_result`, and the planner uses them as it uses
benchmark results. The benchmark runs only for models without such a budget, for example on the
CPU and on Intel graphics, where hosts differ too much for profiles.

### Tier 1 output

The server turns detections into presence per camera and label: `present` from the first
confirmed detection, `cleared` after the label has not been seen for some time. Confirmation and
the clear time are constants in code, tuned on the reference clips. Presence goes to the event
logic, which builds events with a start, an end and labels. Tier 1 runs no tracker.

### Models

- YOLOX (Apache 2.0, https://github.com/Megvii-BaseDetection/YOLOX) is the default family: nano,
  tiny and small, at input sizes such as 320, 416 and 640. Every accelerator vendor's tools support
  it, and its head is simple.
- An NPU needs the model compiled per chip, with a fixed input size and 8-bit weights. The CPU and
  OpenVINO run ONNX directly.
- Every model file is built in CI, scored against a reference set, and published with a sha256.
  The server fetches it at first start and checks the hash. A module lists the files it needs in
  `probe_result`.

Model families that cannot be used, and why:

| Family | Licence | Why not |
|---|---|---|
| Ultralytics YOLOv5, v8, 11, 26 | AGPL-3.0 | Copyleft over the network. Everyone who ships or hosts the product would be bound by it. |
| YOLOv10, YOLOv12, YOLOv13 | AGPL-3.0 | Same. |
| YOLOv9 (original repository), YOLOv6 | GPL-3.0 | Copyleft. Not compatible with shipping under Apache 2.0. |
| YOLO-NAS pretrained weights | non-commercial licence | The weights may not be used commercially, and anyone may ship this product on hardware they sell. |

Apache 2.0 alternatives to evaluate later, for accuracy on stronger hardware: D-FINE, RT-DETR,
RF-DETR (the Nano, Small, Medium and Large sizes; other sizes have a different licence),
NanoDet-Plus and PP-PicoDet.

### First modules

| Module | Decoding | Inference | Phase |
|---|---|---|---|
| CPU | software (FFmpeg) | ONNX Runtime, CPU | MVP |
| Intel graphics | Quick Sync or VAAPI through FFmpeg | OpenVINO (graphics, or the NPU on Intel Core Ultra) | MVP |
| Rockchip (Home Assistant Green) | software to start with | RKNN | MVP, depends on the driver |
| Qualcomm QCS6490 class | to be settled | QNN, through ONNX Runtime's QNN execution provider | Later |
| Hailo (for example the Raspberry Pi AI HAT+) | software | HailoRT | Later |
| NVIDIA | NVDEC | CUDA or TensorRT | Later |

**CPU.** The reference module. It runs everywhere and is the reference in CI. On a Green or a
Raspberry Pi 4 it carries one or two cameras of continuous detection with a very small model, and
more when it only looks on motion (estimate, to be measured).

**Intel graphics.** About half of the Home Assistant OS installs are x86 (virtual machine 31.8%,
generic x86-64 19.9%, https://analytics.home-assistant.io/). The stack is open and already works
under Home Assistant OS. A virtual machine needs the graphics device passed through; the module
reports when it is not.

**Rockchip (Home Assistant Green).** NPU detection on the Green needs a kernel driver that Home
Assistant OS does not ship today. The route is being worked out with the Home Assistant OS
maintainers (see home-assistant/operating-system#3089). The module is built from Cairn's Rockchip
code so it is ready when the driver is. Until then a Green records, uses the camera's own events
and detects on the CPU. That baseline ships in any case.

Rockchip's NPU runtime (`librknnrt`) is a closed library. The module loads it with `dlopen`.
Whether it may be shipped inside the module or must be downloaded at first start is an open
question; Cairn downloads it at first start, pinned by sha256.

### Conformance kit

Every module ships with a conformance kit and passes it before release:

- **Reference clips.** Short substream-sized clips in H.264 and H.265: day and night, people,
  pets, vehicles, a parcel, and empty scenes with moving trees and rain. All under a licence that
  allows redistribution.
- **Expected detections.** Per clip, the labels and boxes that must be found, with a tolerance on
  score and box overlap, and the frames where nothing may be found.
- **A time budget.** Per model, the maximum time per pass on the module's reference hardware.
- **Behaviour.** The handshake, refusing an unknown contract version, a stream that stops, a
  corrupt access unit, a restart after a crash.

The kit is a command that runs against a module binary and writes a report. CI runs it for the CPU
module on every change. Hardware modules run it on runners that have the hardware.

### Where the code lives

- `camera-assistant/camera-assistant`: the contract crate, the shared engine crates, the server's
  module supervisor and planner, the CPU and Intel modules, the conformance kit.
- A board-specific module can move to its own repository (`camera-assistant-rockchip`,
  `camera-assistant-qualcomm`). It depends on the contract crate and the engine crates by version.

### What moves from Cairn, and where it lands

Paths are in https://github.com/bbangert/cairn-nvr unless the row says otherwise.

| In Cairn | Lands in |
|---|---|
| Model handling: backends, output decoding, labels, quantisation parameters (`plugins/cairn-detect/src/infer/`) | The engine crates |
| Decode and scale (`plugins/cairn-detect/src/decode.rs`, `hwdecode.rs`, `glscale.rs`; the decode stream in `plugins/cairn-native/src/decoder.rs`) | The engine crates. The V4L2 decoding path and the GL scaler go to `camera-assistant-qualcomm`. |
| Motion check (`plugins/cairn-detect/src/motion.rs`, `gate.rs`) | The engine crates |
| Engine and per-camera streams (`plugins/cairn-ort/src/engine.rs`, `stream.rs`) | The engine crates, without the Rustler types |
| The stand-alone detect binary (`plugins/cairn-detect/src/bin/cairn-detect.rs`, `cli.rs`, `multiplex.rs`, `emit.rs`, `control.rs`, `rtp.rs`) | The starting point for the module host crate. Its RTP input is replaced by the contract. |
| The Rockchip code (the cairn-rockchip repository, https://github.com/bbangert/cairn-rockchip: RKNN, MPP, RGA, the DMA buffer pool, the probe) | `camera-assistant-rockchip` |
| The Qualcomm backend (QNN through ONNX Runtime, in `plugins/cairn-detect/src/infer/backend/onnxruntime.rs`) and V4L2 decoding | `camera-assistant-qualcomm` |
| Hardware profiles (`priv/profiles/generic/qcs6490-tier1.yml`; `priv/profiles/rk3566-tier1.yml` in cairn-rockchip) | The repository of each board family |
| Export and verification tools (`tools/qdq-export`, `tools/board-bench`, `tools/board-soak`, `plugins/cairn-detect/verify`; `tools/rknn-export` in cairn-rockchip) | The model builds step and the conformance kit |
| Behaviour that Cairn has in Elixir: the picker (`lib/cairn/pipeline/picker.ex`), the sample gate (`Cairn.Stage.SampleGate`), ladder selection (`lib/cairn/config.ex`), fetching files (`lib/cairn/artifacts.ex`), health and the canary (`lib/cairn/native/health.ex`, `canary.ex`) | Written again in Rust: ladder selection and fetching files in the server, the rest in the module host crate |

Of the model catalog, the server only picks families that the "Models" section allows.

Does not come along:

- the Membrane pipeline and its elements;
- the FFmpeg bridge process;
- the Rustler bindings, and the board loader with its manifest;
- the LiveView pages;
- camera configuration in a YAML file.

## Decisions

1. **Modules are separate processes, one per kind of hardware.** Reason: closed libraries stay out
   of the server, a driver crash costs detections and never recordings, and modules can be owned
   and released apart from the server.
2. **The contract is compressed substream in, detections out.** Reason: decoding, scaling and
   inference share memory inside the module, and little data crosses the process boundary.
3. **No settings for models, rates or tiers.** The server measures and decides. Reason: the
   product is plug and play. An advanced override is added only where real cases show that the
   automatic choice falls short.
4. **Only tier 1 (presence) in the MVP.** Reason: it fits every box that can detect, and it runs
   no tracker.
5. **YOLOX is the default model family.** Reason: Apache 2.0, supported by every accelerator's
   tools, and already used by Cairn.
6. **The first modules are CPU, Intel graphics and Rockchip.** Reason: the CPU runs everywhere,
   about half of the installs are x86, and the Green is the most common single board.
7. **The detection engine comes from Cairn,** moved into this project as Rust library crates
   without the Elixir runtime. Reason: it holds working decode, motion, model and Rockchip code
   with tests, under Apache 2.0.
8. **Every module passes a conformance kit before release.** Reason: quantisation defects can
   pass every CPU check and only show on the accelerator.
9. **No silent fallback.** A module that cannot use its hardware says why, and the server plans
   without it. Reason: a slow path that looks like a fast one breaks the plan.

## Steps

1. **[server] The module contract, version 1.** A crate (`ca-detect-contract`) with the message
   types of the contract, the encoding, the version handshake and a written specification in
   `docs/`. The `Detector` trait and its child-process client. Includes a fake module for tests
   that implements `Detector`.
2. **[server] Move the engine from Cairn.** Bring `cairn-detect` and the engine part of
   `cairn-ort` into the repository as library crates with the `ca-` prefix, as the table "What
   moves from Cairn, and where it lands" says. Remove the Rustler types, keep the tests, keep the
   licence notices and credit Cairn by its repository link in the README.
3. **[CI / tooling] The conformance kit.** On the skeleton of the project setup plan (case format,
   `ca-conformance` runner, report): connect the runner to the real contract, add the behaviour
   checks, the first set of reference clips with expected detections, and run it for the CPU
   module in CI.
4. **[server] The module host.** A shared crate that every module uses: the socket, the
   handshake, one pipeline per stream (decode, scale, motion check, inference), dropping to the
   next keyframe when behind, and `stats`. The `Board` and `DecodeStage` traits, the canary for
   model loads, and `health` from the time of completed passes and the age of outstanding calls.
5. **[server] The CPU module.** Software decoding, ONNX Runtime on the CPU, the YOLOX ladder.
   Passes the conformance kit in CI on x86-64 and aarch64.
6. **[server] The module supervisor.** Finds the installed modules, starts them, runs the
   handshake, restarts a module that crashes or hangs with a back-off that grows steeply when it
   keeps failing, kills a module that is `wedged` or whose `health` does not arrive in time, and
   reports module health
   in the API. A module crash never affects ingest or recording; a test proves it.
7. **[server] Feed the substream to a module.** Takes a camera's substream from the ring and sends
   it as access units. Uses the main stream when a camera has no substream and flags that in the
   camera's health. Never blocks ingest.
8. **[CI / tooling] Model builds.** Export YOLOX nano, tiny and small to ONNX, quantise, compile
   per target, score each result on the target or its simulator, and publish the files with their
   sha256. A fetcher in the server downloads and checks them, and the other files a module lists, at
   first start.
9. **[server] Benchmark the host.** Runs the benchmark per module and model, stores the results,
   and runs again after an update of a module or a model. Skips models that have a budget from the
   module's hardware profile.
10. **[server] The planner.** Chooses the step, rate, model and module per camera, watches the
    `stats`, steps down and up, and re-plans on changes. Exposes the chosen level per camera in
    the API.
11. **[server] Detection on camera events.** A camera event from Home Assistant or a Matter zone
    event opens a detection window on that camera, whatever its step.
12. **[server] Tier 1 presence.** Turns detections into present and cleared per camera and label,
    and hands them to the event logic. The label set follows Matter's `IdentifiedObject` tags.
13. **[hardware] Intel module: OpenVINO inference.** Inference on the graphics, or on the NPU of
    an Intel Core Ultra, with software decoding first.
14. **[hardware] Intel module: Quick Sync decoding.** Hardware decoding of H.264 and H.265 for
    the substream. Falls back to software decoding per stream and says so in `stats`.
15. **[hardware] Rockchip module.** The RKNN module from Cairn's Rockchip code: runtime loaded
    with `dlopen` and fetched at first start, models compiled for the RK3566, its hardware profile
    for the RK3566, software decoding to start with. Passes the conformance kit on a Green once
    the driver is available.
16. **[os] NPU driver on the Green.** Work out the route with the Home Assistant OS maintainers
    (home-assistant/operating-system#3089) and test the Rockchip module on it. Tracking issue.
17. **[frontend] Detection status in the interface.** Per camera: what the box does now
    ("Detection: on motion, person and vehicle") and, when it stepped down, why. No settings.
18. **[server] Map COCO classes to the tier 1 labels.** One table in the engine from COCO class
    to `person`, `pet` or `vehicle`, with the fine class kept next to it, and tests.
19. **[documentation] How to write a module.** The contract, the conformance kit, model builds and
    how a module is released.
20. **[hardware] Measure the cost of the process boundary.** On a Home Assistant Green and on an
    x86 host: memory and CPU of a module as a process, compared with the same module linked into a
    test binary, and what a crash costs in each form. Answers open question 12.

## Order

- Steps 1 and 2 come first and can run side by side. Step 3 needs step 1 and the conformance kit
  skeleton of the project setup plan.
- Step 4 needs steps 1 and 2. Step 5 needs steps 3 and 4.
- Step 6 needs step 1 and can start with the fake module. Step 7 needs step 6 and the ring.
- Step 8 can start early; steps 5, 13 and 15 use its output.
- Steps 9 and 10 need steps 5 and 6. Step 11 needs step 10 and the camera events from the Home
  Assistant integration. Step 12 needs step 7 and feeds the event logic.
- Steps 13 and 14 need step 4. Step 15 needs step 4 and, to finish, step 16.
- Step 17 needs step 10 and the panel. Step 18 stands alone and is used by step 12. Step 19
  follows steps 1, 3 and 8.
- Step 20 needs steps 5 and 6.

## Open questions

1. **Encoding of the contract.** Options: length-prefixed MessagePack or CBOR, Protocol Buffers,
   or JSON lines for control with binary frames for access units.
2. **How a module reaches the box.** Options: all modules inside the app image, released with
   their own version; modules fetched by the server at first start after it probed the host; or a
   separate app or container per module. The first is simplest; the others keep the image small
   and let a module ship apart from the server.
3. **Shipping Rockchip's runtime.** May `librknnrt` be shipped inside the module, or must it be
   downloaded at first start? This needs a licence answer.
4. **Device access in the app.** Does Rockchip's runtime need `/proc/device-tree` inside the app
   container (home-assistant/supervisor#4863)? Which device entries does each module need in the
   app configuration?
5. **FFmpeg build.** Which FFmpeg build do the modules use, so that the whole stays shippable under
   Apache 2.0? Options: an LGPL-only build that the project makes, or a distribution build checked
   for GPL parts.
6. **Packages.** COCO-trained YOLOX has no "package" class. Options: leave `package` to camera
   events in the MVP, or train a model on camera images with an open licence.
7. **COCO weights.** Do weights trained on COCO carry any restriction? The legal position is not
   settled. Options: accept, as most projects do, or train on an own data set later.
8. **Intel inference path.** OpenVINO directly, or ONNX Runtime with the OpenVINO execution
   provider, which reuses the engine from Cairn?
9. **Headroom.** Which share of measured capacity does the planner use, and how long does it wait
   before it steps up again? A starting value comes from runs on real hosts.
10. **Rockchip decoding.** Whether the Green can decode the substream in hardware for the NPU
    module is not settled. Until it is, the module decodes in software, and the number of cameras
    has to be measured on a Green.
11. **Runners with hardware.** Who hosts CI runners with an Intel host and a Green, so that hardware
    modules run the kit on every release?
12. **Linking a module into the server.** The `Detector` trait allows it. It saves the copy of the
    substream and the base memory of a second process. But then a vendor library that aborts or a
    native call that hangs stops recordings, the server links FFmpeg and vendor runtimes, and each
    board family ships its own build of the whole server. Step 20 decides it; until then every
    module is a process.

## Later, and what this must not block

- **Tier 2, tracking.** The contract returns boxes per frame with timestamps, which a tracker
  needs. The tracker runs in the server or in a module; the contract must not assume either.
- **Tier 3, recognition.** Faces and licence plates as optional local modules. They return labels
  with extra attributes; the `detections` message leaves room for that.
- **More modules.** Qualcomm QCS6490 class (QNN), Hailo, NVIDIA, and others through the same
  contract and kit.
- **The Matter AV Analysis Node role.** Detections already use Matter's labels. When Matter 1.7 is
  final, the server can report them as standard Matter events.
- **Detection from other sources.** A camera's own analysis or another analysis node as a source
  next to the local modules.
- **Zones drawn in the interface.** `set_mode` takes motion zones, and presence can be per zone.
- **Sound events.** A module for audio follows the same contract with a different input.
