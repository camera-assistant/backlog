# Repositories that many people and coding agents can work in at once

Phase: Preview. Epic: 01. This is the technical companion of the epic on the project board, written so a person or a coding agent can pick up a step and start. Read [the architecture](../architecture.md) first.

## Problem

Camera Assistant starts as three empty repositories: `camera-assistant/camera-assistant` (the server, Rust), `camera-assistant/frontend` (the web interface, TypeScript) and `camera-assistant/backlog` (plans, issues, project board). The plans after this one will be worked on in parallel, by people and by their coding agents. Before that can go well, the repositories need a few things:

- A layout that tells everyone where code goes, so two contributors do not invent two structures or fight over one shared file.
- One gate that every change passes: format, lint, tests and builds for both architectures. Review by a maintainer then starts from a change that already builds.
- Written rules for contributions, and short instructions that coding agents read before they start.
- A way to release: a container image and a Home Assistant app, for aarch64 and x86-64.
- Licence checks for dependencies and models. The project is Apache 2.0, a hardware vendor may ship it, and some popular detection models and libraries have licences that do not allow that.
- A clear relation between the main repository and later repositories for one kind of hardware, such as `camera-assistant-rockchip`.
- A first form of the conformance kit that every detection module must pass.

This epic must stay small. It sets up the minimum that lets the other epics start. It is not a framework.

## What exists

- **Cairn** ([bbangert/cairn-nvr](https://github.com/bbangert/cairn-nvr), [bbangert/cairn-rockchip](https://github.com/bbangert/cairn-rockchip)), the source of the first detection engine. Useful prior art for this epic:
  - its Rust CI job runs `cargo fmt`, `cargo clippy -D warnings`, tests, a release build and docs;
  - its board repository has its own CI, including a nightly build against the main repository;
  - its CI also boots deliberately broken fixture boards, to prove that the checks catch them;
  - vendor runtimes that may not be shipped are downloaded at first start, pinned by commit and sha256;
  - models are fetched from a release and pinned by sha256 in a profile.
- **Home Assistant's pattern for dedicated servers.** The Matter Server, Z-Wave JS and Music Assistant each ship a server with a versioned WebSocket schema, a Python client library on PyPI, and a Home Assistant app. The integration checks the schema version when it connects.
- **Rust tooling** that this plan relies on, all widely used:
  - `rustfmt` and `clippy`;
  - [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny) for licences, security advisories and allowed sources;
  - Cargo workspaces with `[workspace.lints]` and `[workspace.dependencies]`;
  - `rust-toolchain.toml` to pin the compiler.
- **GitHub features** this plan uses: issue forms, pull request templates, issue types, labels, projects, the GitHub Container Registry and generated release notes.
- **Model licences** (checked for the research behind this plan): YOLOX is Apache 2.0. The Ultralytics YOLO family and several other YOLO versions are AGPL 3.0 or GPL 3.0, and the pretrained YOLO-NAS weights are for non-commercial use only. None of those can be shipped in this project.

## Concept

### The server repository

One Cargo workspace holds the server and the detection modules that are fully open (CPU and Intel graphics). Hardware modules that need a vendor's closed runtime live in their own repositories.

```
camera-assistant/
  Cargo.toml              workspace: members by glob, shared lints, shared dependency versions
  rust-toolchain.toml     pinned stable toolchain, with rustfmt and clippy
  deny.toml               cargo-deny: allowed licences, advisories, allowed sources
  AGENTS.md               instructions for coding agents (CLAUDE.md points here)
  CONTRIBUTING.md
  SECURITY.md
  LICENSE                 Apache 2.0
  crates/
    ca-server/            the binary: configuration, startup, supervision of the parts
    ca-<part>/            one library crate per part of the architecture, added by later plans
  modules/
    ca-detect-<target>/   one binary per open detection module, added by the detection epic
  conformance/
    cases/                reference cases for detection modules
    runner/               the crate that runs a module against the cases (ca-conformance)
  models/
    models.toml           the model manifest: source, licence, sha256 of every model
  docker/
    Dockerfile
  .github/
    workflows/            ci.yml, release.yml, licences.yml
    ISSUE_TEMPLATE/
    pull_request_template.md
```

- **Members by glob.** `members = ["crates/*", "modules/*", "conformance/runner"]`. An epic that adds a crate does not touch the workspace file, so parallel work does not collide there.
- **Crate names follow the architecture,** with the prefix `ca-`: `ca-ingest`, `ca-ring`, `ca-recorder`, `ca-storage`, `ca-media`, `ca-events`, `ca-api`, `ca-matter`. A prefix avoids clashes with crates on crates.io; `ring`, for example, is a widely used cryptography crate. This epic creates only `ca-server`; later plans add the other crates.
- **Shared settings live in the workspace.** Lints and dependency versions sit in `[workspace.lints]` and `[workspace.dependencies]`. A crate opts in with `lints.workspace = true` and `dep = { workspace = true }`, so two crates never pull two versions of one library by accident.
- **`unsafe` code is forbidden** (`unsafe_code = "forbid"`) in every crate except those that wrap a C library. Such a crate is named `*-sys` or is a hardware module, sets its own lints, and says in its README why it needs `unsafe`.

The first `ca-server` does one thing: `ca-server --version` prints the version and the API schema version, and exits. That is enough to build and test the release pipeline end to end. Step 1 of the [server foundation plan](server-foundation.md) builds on this workspace: it adds the crates of the parts and the process skeleton.

### Continuous integration

`ci.yml` runs on every pull request and on every push to `main`:

| Job | Command | Fails on |
|---|---|---|
| format | `cargo fmt --all --check` | any difference |
| lint | `cargo clippy --workspace --all-targets -- -D warnings` | any warning |
| test | `cargo test --workspace` | any failing test |
| build | release build for `x86_64-unknown-linux-gnu` and `aarch64-unknown-linux-gnu` | any build error |
| licences | `cargo deny check` and the model manifest check | a disallowed licence, a known advisory, an unknown source, a model missing from the manifest |
| conformance | the conformance runner against the fake modules | a fake module that should pass and fails, or the reverse |

CI must stay fast, because agents and people iterate on it. Tests that need real cameras or real accelerators do not run in this workflow. They run in the repository of that hardware, or on runners that hardware owners provide.

The frontend repository has the same shape. Its CI calls four scripts from `package.json`, so the workflow does not depend on the framework that the web interface plan picks:

| Script | Does |
|---|---|
| `lint` | the linter and the formatter in check mode |
| `typecheck` | the TypeScript compiler without output |
| `test` | the unit tests |
| `build` | the production build into `dist/` |

### Releases

- **Versions.** The server follows semantic versioning, starting at `0.1.0`. It stays below `1.0` until the MVP. The API schema has its own integer version, which the server reports and the Home Assistant integration checks, as the Matter Server and Z-Wave JS do. A release can change the schema version or not; the two numbers are independent.
- **One image, two ways in.** A tag `vX.Y.Z` on `main` starts `release.yml`. It builds the server for both architectures, puts the binaries and the frontend files in one multi-architecture image, pushes it to `ghcr.io/camera-assistant/camera-assistant:X.Y.Z`, and creates a GitHub release with generated release notes. Each push to `main` also publishes the tag `main` for testers. The Home Assistant app runs the same image.
- **Base image.** A slim Debian image with glibc. Vendor runtimes and the libraries that detection modules use are built against glibc, so a musl image would block them later.
- **Frontend files.** A frontend release produces a versioned archive of the built files. The server image pins one frontend version and copies the files in. The server serves them; it does not build them.
- **The Home Assistant app.** An app definition (a `config.yaml` and its documentation) points to the image for `aarch64` and `amd64`. It maps the storage that the Supervisor grants to apps, so locations set up in Home Assistant reach the server. A smoke test installs it on a test Home Assistant OS system and checks that it starts. Where the app definition lives is an open question below.

### Licence rules

The rules are written down in `CONTRIBUTING.md` and enforced in CI where a tool can do it.

- **Rust dependencies.** `deny.toml` allows Apache-2.0, MIT, BSD-2-Clause, BSD-3-Clause, ISC, Zlib, Unicode-3.0 and the combinations of these. Anything else fails CI. A new licence is added to the list in its own pull request, with the reason in the description. Dependencies come from crates.io only; a git dependency needs an explicit entry in `deny.toml`. Security advisories from the RustSec database fail CI.
- **Frontend dependencies.** The same list, checked by a licence tool in the frontend CI. Which tool is for the step to choose.
- **FFmpeg.** Where a module uses FFmpeg, it uses a build without `--enable-gpl` (LGPL). A GPL build is never shipped.
- **Models.** Every model file that the project ships or downloads has an entry in `models/models.toml`: name, source address, licence, sha256 and a note on the training data. The check fails when a model in a release or in a download list has no entry, or when its licence is not Apache-2.0 or MIT. AGPL, GPL and non-commercial weights are not allowed.
- **Closed vendor runtimes** (such as Rockchip's or Qualcomm's) never enter the main repository or the server image. They live in a hardware module. Where the licence allows shipping them, the module image carries them with the licence text. Where it does not, the module downloads them at first start from a pinned address and checks a pinned sha256.

### Contribution rules

`CONTRIBUTING.md`, the same text in each repository:

- Start from an issue. A change that is not on the board starts as an issue or a discussion. A change to a design starts as a pull request to the plan in `camera-assistant/backlog`.
- One issue per pull request, and small pull requests. The pull request links its issue and the plan step.
- Tests come with the change. A bug fix comes with a test that fails without the fix.
- Design reasoning goes into the plan or the pull request description, never into private notes.
- Contributions written with a coding agent are welcome. The person who opens the pull request has read the change, has run it, and answers for it like for their own code.
- A maintainer reviews every change before it is merged.
- New dependencies follow the licence rules above.
- Security problems are reported privately, as described in `SECURITY.md`, not in a public issue.

### Instructions for coding agents

Each repository gets one `AGENTS.md` and a `CLAUDE.md` that only points to it, so tools that look for either file read the same text. It stays short, under about 150 lines, and links instead of copying:

1. What this repository is, in three lines, and a link to the architecture.
2. Where the plans are: `camera-assistant/backlog`, `docs/plans/`. "Read the plan step of your issue before you write code."
3. How to build, test and check, as exact commands, the same as CI runs.
4. The layout: which crate or folder holds what.
5. Conventions: error handling, logging, naming, tests next to the code, no `unsafe` outside FFI crates, no new dependency without the licence check.
6. What not to do: no changes outside the issue's scope, no edits to generated files, no disabled lints or tests to make CI pass, no secrets or real camera addresses in code or tests.
7. When to stop and ask: the plan does not answer a design question, or a change touches the API schema.

The backlog repository gets an `AGENTS.md` too. It describes the plan format (the sections every plan has), the epic file format, and the rule that plans describe the current design, not its history.

### Issue and pull request templates

- **Server and frontend:** a bug report form (version, how it is installed, hardware, camera brand and model and how it connects, what happened, logs) and a feature request form that points to discussions. Blank issues are off, so a report always has the fields.
- **Backlog:** templates for Epic, Feature and Task, with the same sections as the epic files: Problem, What we build, Challenges, Asks to others, and a checklist.
- **Pull requests:** what changed and why, the linked issue, and a short checklist: tests added, docs updated, no new dependency outside the licence rules, schema version changed if the API changed.
- **Labels and fields** in the backlog and on the board: the labels `feature`, `enhancement`, `chore`, `frontend`, `server`, `hardware`, `hass-integration`, `matter`, `Needs UX Design`, `help wanted`, `good first issue`; the issue types Epic, Feature and Task; the board fields Area, Size and Phase.

### Repositories for one kind of hardware

A hardware repository, such as `camera-assistant-rockchip`, holds one detection module and everything specific to that hardware: bindings, build scripts, model conversion tools, the vendor runtime rules. It relates to the main repository in three ways only:

- **It depends on the detection contract and the shared engine crates, not on server internals.** These crates live in the main repository and have their own versions. The hardware repository pins released versions. The server and the module check the contract version when they connect, and refuse a different major version with a clear message.
- **It passes the conformance kit.** Its CI runs the kit from the main repository at the same pinned version: on the CPU where the module can run there, and on a runner with the real hardware where its owners provide one.
- **It releases on its own.** A module that is late does not hold up a server release. A nightly job builds the module against the contract on `main`, so a breaking change shows up before it is released.

The main repository offers a reusable workflow for this, so a hardware repository needs a few lines of CI, not a copy.

### The conformance kit

The kit proves that a detection module does what the contract says, on the hardware it claims. This epic builds the skeleton: the case format, the runner, the report and the runner's own tests. The conformance kit step of the [detection plan](detection.md) builds on it: it adds the reference clips with their expected detections, the behaviour checks (handshake, unknown contract version, a stream that stops, a corrupt access unit, a restart) and the connection to the real contract.

A case is a folder:

```
conformance/cases/<case-name>/
  case.toml       what the case tests, the clip's codec and size, its licence
  clip.mp4        a short H.264 substream-sized clip
  expected.json   per timestamp: label, box, minimum score; the tolerance for box overlap
```

The time budget is not part of a case: the kit holds a maximum time per pass for each model on each target, and a module declares its target.

The runner, `ca-conformance run --module <path> --target <name>`:

1. starts the module as a process, as the server would;
2. feeds each clip through the contract;
3. compares the detections with the expected ones: same label, box overlap above the tolerance, score above the minimum; nothing extra above the minimum score;
4. measures the time per pass and compares it with the budget of the target;
5. writes a report as JSON and as readable text, and exits with an error when a case fails.

Until the detection contract exists, the runner talks to modules through a small trait with one implementation: a fake module that replays a file of detections. The runner's own tests use two fakes: one that must pass and one that must fail. A runner that cannot tell them apart fails CI.

Reference clips must be free to share: recorded by the project or under a licence such as CC0 or CC BY, and with the consent of anyone who can be recognised in them.

## Decisions

1. **One Cargo workspace for the server and the open modules, with members by glob.** One toolchain and one CI, and parallel epics do not edit a shared list.
2. **Crates are named after the parts of the architecture, with the prefix `ca-`.** A contributor finds the code from the architecture, and the prefix avoids clashes with crates on crates.io.
3. **A pinned stable toolchain.** People, agents and CI build with the same compiler.
4. **`unsafe` only in crates that wrap a C library.** The compiler then guards everything else, which matters most when agents write much of the code.
5. **CI is the gate, and it is fast.** Format, lint with warnings as errors, tests, builds for both architectures, and licence checks on every pull request. Hardware tests run elsewhere.
6. **Release and API schema have separate versions.** The Home Assistant integration checks the schema, not the release.
7. **One image for the container and for the Home Assistant app**, on GitHub's container registry, built for aarch64 and x86-64 on a glibc base. One artifact to test, and the registry sits in the same organization.
8. **Permissive licences only in what ships; closed vendor runtimes only in hardware modules.** The project is Apache 2.0 and must stay shippable by anyone, a hardware vendor included.
9. **Every model has a manifest entry with licence and checksum.** Model licences are easy to get wrong, and a checksum makes a download verifiable.
10. **Hardware repositories depend on the contract only, pass the same conformance kit, and release on their own.** The people with the hardware can own the module, and the server never waits for it.
11. **One `AGENTS.md` per repository, short, linking to the plans.** Agents read it first, and links keep it from going stale.
12. **Design reasoning lives in the open**, in the plans and in pull requests. Code that many people and agents change needs its reasons where everyone can read them.

## Steps

1. **[other] The backlog repository and the board.** Labels, issue types and board fields as listed under "Issue and pull request templates". Issue templates for Epic, Feature and Task. A README that explains how the backlog works: plans in `docs/plans/`, the architecture in `docs/architecture.md`, one epic per plan, sub-issues per step. The backlog `AGENTS.md`.
2. **[server] The Rust workspace.** `Cargo.toml` with members by glob, shared lints and the shared dependency table; `rust-toolchain.toml`; `ca-server` with `--version`; the empty `modules/`, `conformance/` and `models/` folders, each with a README of a few lines; `LICENSE`.
3. **[CI / tooling] Server CI.** `ci.yml` with format, lint, test and build for x86-64 and aarch64, with build caching, on pull requests and on `main`. Branch protection on `main` requires it.
4. **[CI / tooling] Dependency licence and advisory checks.** `deny.toml` with the allowed licences, advisories and sources; a `licences` job in the server CI; the same allowed list checked in the frontend CI.
5. **[CI / tooling] The model manifest and its check.** The format of `models/models.toml`, a check that fails on a model without an entry or with a licence outside the allowed list, and its tests. The manifest starts empty; the detection epic adds the first models.
6. **[frontend] Frontend repository baseline.** A TypeScript project with the four scripts `lint`, `typecheck`, `test` and `build`, CI that runs them, and a release workflow that publishes a versioned archive of `dist/`. The framework and component choices are left to the web interface plan.
7. **[documentation] Contribution rules.** `CONTRIBUTING.md`, `SECURITY.md` and a code of conduct in the server and frontend repositories, following the rules above.
8. **[documentation] Instructions for coding agents.** `AGENTS.md` and the pointing `CLAUDE.md` in the server and frontend repositories, following the outline above, with the exact commands from steps 2, 3 and 6.
9. **[CI / tooling] Issue and pull request templates.** The bug report and feature request forms and the pull request template in the server and frontend repositories.
10. **[CI / tooling] Container image release.** The Dockerfile, the multi-architecture build, the push to the registry on a version tag and as `main` on every push to `main`, a smoke test that runs `ca-server --version` in the image for both architectures, and a GitHub release with generated notes.
11. **[hass-app] The Home Assistant app.** The app definition for aarch64 and amd64 that runs the image, maps the storage the Supervisor grants to apps, and is installed and started once on a test Home Assistant OS system, as a documented check.
12. **[CI / tooling] The conformance kit skeleton.** The case format, the `ca-conformance` runner with the fake-module trait, the pass and fail fakes, its tests, and a `conformance` job in CI.
13. **[CI / tooling] The pattern for hardware repositories.** A reusable workflow in the main repository that builds a module, runs the conformance kit at a pinned version and runs the nightly build against `main`; a short document on how a hardware repository uses it and what it may and may not contain.

## Order

- Step 1 comes first, so the other steps can be filed and tracked.
- Step 2 comes before steps 3, 4, 5, 10 and 12. Steps 3, 4 and 5 can then run in parallel. Step 1 of the server foundation plan needs step 2.
- Step 6 can start at any time.
- Step 7 and step 9 can start at any time. Step 8 needs steps 2, 3 and 6, because it quotes their commands.
- Step 10 needs step 3, and the frontend archive of step 6. Step 11 needs step 10.
- Step 12 needs step 2. The conformance kit step of the detection plan needs step 12.
- Step 13 needs step 12, and the first release of the detection contract crate from the detection plan.

## Open questions

- **How aarch64 builds run in CI.** Native arm64 runners, or cross-compilation from x86-64 (with `cross` or `cargo-zigbuild`). Native runners test what ships; cross-compilation is easier to cache. The step 3 builder picks one and writes down why.
- **Where the Home Assistant app definition lives.** In the server repository (one place, released together), or in a repository of its own, as the Home Assistant app repositories of other projects are. A repository of its own fits the Supervisor's repository format; the server repository keeps the version in one place.
- **Where the Python client library lives.** A repository of its own (published to PyPI, as the Matter Server's client is), or a folder in the server repository. This also decides where its CI and `AGENTS.md` go.
- **How the contract and engine crates reach hardware repositories.** A pinned git tag of the main repository, or releases on crates.io. A git tag needs no publishing step; crates.io gives normal version resolution.
- **How a module from a hardware repository reaches a user's box.** Bundled into an image for that board, shipped as an app of its own, or downloaded by the server when it finds the hardware. This touches the "no knobs" principle and the vendor runtime rules, so the detection plan settles it; this plan only needs the answer before step 13 is final.
- **Where reference clips are stored.** In the repository with Git LFS, as release assets, or in a repository of their own. Clips are binary and can grow; a fresh clone should stay small.
- **The training data of pretrained weights.** YOLOX weights are Apache 2.0, but they are trained on COCO, whose images come under several Creative Commons licences. Whether that limits the weights is not settled. Options: accept it as most projects do, or train on data the project owns later.
- **Sign-off on contributions.** No sign-off, a Developer Certificate of Origin line on each commit, or a contributor licence agreement.
- **Which code of conduct.** An existing text such as the Contributor Covenant, or another.
- **One place for shared files.** An organization-wide `.github` repository can hold the default `CONTRIBUTING.md`, `SECURITY.md`, code of conduct and issue templates for every repository. That avoids copies, but adds one repository.

## Later, and what this must not block

- **An npm package of the web components**, so Home Assistant's dashboard cards can reuse the player and the timeline. The frontend release of step 6 should not assume the archive is the only output.
- **A beta channel** for the Home Assistant app, next to the stable one.
- **Hardware-in-the-loop CI**, on runners that the owners of a kind of hardware provide. The reusable workflow of step 13 takes a runner label as input.
- **Fuzzing** of the parsers for RTSP, RTP and CMAF, which read data from untrusted cameras. The workspace layout leaves room for a `fuzz/` folder per crate.
- **More hardware repositories** follow the pattern of step 13 without a change in the main repository.
