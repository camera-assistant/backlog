# Instructions for coding agents

This is the backlog of Camera Assistant (a working title), a local recorder for the cameras in a home, built for Home Assistant first.
It holds the technical plans and the issues of the [project board](https://github.com/orgs/camera-assistant/projects/2). No code is built here.
Read the [architecture](docs/architecture.md) first, then the [plans](docs/plans/README.md).

- **One plan per epic.** Each epic on the board links its plan in `docs/plans/`.
- **One sub-issue per plan step.** The sub-issues of an epic are the steps of its plan, in order.

## Picking up work

1. Find the sub-issue, and read its epic and the plan step it comes from before you write code. The sub-issue names the plan sections it relies on.
2. The code goes to the repository of its area: `camera-assistant/camera-assistant` for the server (Rust), `camera-assistant/frontend` for the web interface (TypeScript). Follow the `AGENTS.md` of that repository.
3. The pull request there says `Closes https://github.com/camera-assistant/backlog/issues/<n>`.
4. When the plan does not answer a design question, stop and ask the person you work for. Do not decide it silently.

## The plan format

Every plan in `docs/plans/` has these sections, in this order:

- **Title.** The goal, in one line, as an outcome.
- **Opening paragraph.** The phase (Preview, MVP or after the MVP) and the epic it belongs to. Optionally "Terms used here" or "Related plans".
- **Problem.** What is missing today, and for whom.
- **What exists.** Prior art, standards, libraries and tools the plan relies on, with links.
- **Concept.** The design, split into `###` subsections: parts, data shapes, formats, behaviour.
- **Decisions.** A numbered list: the decision in bold, then the reason in a sentence or two.
- **Steps.** A numbered list of work items. Each starts with its area in square brackets and a short bold title, for example `1. **[server] The ring.**`, followed by what it delivers, tests included.
- **Order.** Which steps depend on which, and which can run in parallel. Dependencies on other plans are named here.
- **Open questions.** Design questions the plan leaves open, each with its options and what decides it.
- **Later, and what this must not block.** Work after this epic, and how the current design leaves room for it.

A plan whose work happens partly in another project adds a section with its asks to that project, after "Order" (see the [recording provider platform](docs/plans/recording-provider-platform.md)).

The areas of steps are the values of the board's Area field: `server`, `frontend`, `hass-integration`, `hass-app`, `hardware`, `matter`, `documentation`, `CI / tooling`, `os`, `ux-design`, `multi`, `other`.

## The issue format

Issues are written from the plans. Titles are short and functional ("Matter camera support", "RTSP client library choice"), not sentences.
The [issue templates](.github/ISSUE_TEMPLATE) hold the formats below.

**Epics** (issue type Epic) are the only issues on the board. The body uses bold lead-ins:

- `**Problem.**` what is missing, for a household and for the project.
- `**What we build.**` the outcome, as a short list.
- `**Challenges.**` what is hard or not yet verified.
- `**Asks to others.**` what other projects must add or agree to, or "None".
- Optionally `**Later, and not blocked by this.**`
- It closes with "The steps are the sub-issues below, in order." and a link to its plan.

**Sub-issues** (issue type Feature or Task) are linked to their epic as GitHub sub-issues. Feature builds a capability; Task is setup, documentation, a spike or a choice. The body has:

- `**Problem.**` in one or two sentences.
- `**What we build.**` pointing to the plan section it builds.
- A task list, each line starting with its area: `- [ ] server: ...`. Common prefixes: `server:`, `frontend:`, `hass-integration:`, `hass-app:`, `matter:`, `hardware:`, `CI:`, `documentation:`, `docs:`.

**Labels:** `feature`, `enhancement`, `chore`, `bug` for the kind of work; `server`, `frontend`, `hass-integration`, `matter`, `hardware` for the area; `Needs UX Design`, `help wanted` and `good first issue` where they apply.

**Milestones** set the phase of an issue: Preview, MVP or AVP (after the MVP).

**The board** has the columns Triage, Later, Next, In progress and Done, and the fields Priority, Size and Area.

## Writing rules

- Plans describe the current design, not its history. No "previously", no change logs. A changed decision replaces the old text.
- Plain English and short sentences.
- A design question that a plan leaves open goes under "Open questions". Do not answer it silently in a plan or in code.
- A change to a design is a pull request to its plan in this repository, and the epic and sub-issues follow it.
- Nothing private in public text: no names of people, no addresses, passwords, tokens or camera URLs. Use made-up examples.
- No dates or deadlines. The phase is the milestone.

## What not to do

- Do not create, close, retitle or relabel issues, or move items on the board, unless the person you work for asked for exactly that.
- Do not rewrite other people's plans beyond the change at hand.
