# Extra Lives Research: Project Direction and Future Plan

## Purpose of This Document

This document preserves the project's known background, user-stated goals, current research status, constraints, and possible future directions so a different model or contributor can continue without losing context.

It is a planning and handoff document, not proof that proposed features have been implemented. When something is a hypothesis, recommendation, or untested idea, it is labeled as such.

## 1. Project Identity and Scope

- Research repository: `Fwlai/AnchaHouse-Research`
- Repository URL: https://github.com/Fwlai/AnchaHouse-Research
- Main branch: `main`
- Research target: Extra Lives v1.170.64 by MDickie, from 2017.
- Main game asset investigated: `assets/Game.swf`, extracted from the game's APK.
- Important associated data files: `Characters.txt` and `Frames.txt`.
- The SWF is approximately 40 MB; the original APK was approximately 55 MB.
- The SWF contains one ABC block and 681 exported ActionScript 3 classes, according to the existing static analysis.
- The project currently focuses on reverse engineering, documenting, and evaluating whether parts of the game can be reused, reproduced, run on the web, and eventually synchronized online.

This research project is separate from the Ancha House game itself. Extra Lives was selected in part because its backgrounds and other resources may be useful as references or potential resources for a future web version of Ancha House.

## 2. User-Stated Goals and Constraints

The following points capture the user's stated direction and constraints as understood from prior project conversations.

### Research before modifying Ancha House

The user explicitly required that the original Ancha House code should not be changed immediately as part of the initial Extra Lives investigation. The APK and extracted resources were to be placed in the same research repository, `Fwlai/AnchaHouse-Research`, so they could be inspected first.

Do not treat this as permission to copy assets into Ancha House or modify Ancha House automatically. Research, identify, and document first; any actual reuse or source-project changes should be a separate, deliberate step.

### Investigate the game beyond surface-level observations

The user wants to discover as much useful technical information as possible, not merely produce a short feasibility report. Areas of interest include:

- The SWF and ActionScript code structure.
- Character and frame data.
- Sprite/graphics structure and how characters are rendered.
- Animation and pose reconstruction.
- Gameplay systems, collision, combat, NPC behavior, and saved state.
- Whether the original game can be run in a browser.
- Whether it can be presented as a Telegram Mini App.
- Whether a multiplayer version can be built.

The sprite/animation and browser feasibility areas have been raised as important next research targets. Their full investigation and implementation status must be checked against the current repository documents before work begins.

### Multiplayer is a desired direction, not an implemented feature

The user has asked whether the existing documentation is enough to begin multiplayer work. The current conclusion is that it is enough to begin small, measurable prototypes, but not enough to guarantee a complete, stable online game without runtime experiments and implementation work.

No online multiplayer networking has been implemented in the original SWF by the research described so far. The current repository contains research and proposed prototype designs, not a completed online port.

### Browser and Telegram Mini App are potential delivery targets

The user has asked whether Extra Lives can be moved to the browser and presented as a Telegram Mini App.

This is an active direction to investigate, but no final implementation route has been selected. The main options discussed are:

1. Attempt to run the existing SWF using Ruffle and assess compatibility.
2. Rebuild the necessary game client in HTML5/JavaScript using extracted data and documented behavior.
3. Take a staged approach: validate browser execution first, then introduce web input and networking, and only rewrite systems that prove incompatible or impractical.

Do not assume that Ruffle supports the entire game correctly until tested. A successful browser launch would not automatically add multiplayer or prove Telegram Mini App compatibility.

### Preserve progress across model/provider changes

The user accepts substantial/heavy analysis when it is useful. The key requirement is not to minimize work, but to save results incrementally so token limits, model changes, or provider changes do not erase progress.

Maintain `docs/extralives-swf-analysis/INVESTIGATION-PROGRESS.md` as a durable checkpoint. Update it after every meaningful stage, and commit/push useful findings rather than keeping them only in a conversation.

## 3. Existing Research and Reported Findings

The following findings were reported by the previous analysis phase. Read the corresponding repository documents before relying on them as implementation-level truth.

### SWF and code structure

- The target is Extra Lives v1.170.64.
- The SWF has one ABC block and 681 exported ActionScript 3 classes.
- The game is described as a timeline-hosted, single-threaded 2.5D sprite simulation.
- The existing architecture report identifies 21 scenes and `Game.fla.MainTimeline`.
- The analysis was static. It did not execute the game, so runtime behavior and edge cases remain to be verified.

### Frames.txt

- Reported to contain 1,401 pose records, indexed 0 through 1400.
- Each pose record contains 13 limbs, with 7 fields per limb.
- The fields X, Y, Z, A, S, D, and F were analyzed.
- F was reported as a limb subframe field; D was reported to affect weapon direction and flipping.
- Pose data affects more than appearance: limb pose data can influence runtime collision geometry and strike direction.
- Verify detailed interpretations against `FRAMES-TXT-ANALYSIS.md` and the ActionScript consumers before changing the implementation.

### Characters.txt

- Reported to contain 200 records, each with 327 fields.
- Character identity is ordinal/index-based rather than based on a unique character name.
- Relationships, statistics, customization, RGB values, stance/flight/trigger/prop fields, and local SharedObject saves were investigated.
- Verify field mappings and edge cases in `CHARACTERS-TXT-ANALYSIS.md`.

### Game loop and determinism

- The current game loop was reported to be variable-rate rather than fixed-timestep.
- Randomness uses unseeded `Math.random()`.
- Audio/visual effects also consume values from the same random sequence.
- Deterministic lockstep is therefore not considered reliable for the unmodified game.
- This is a static-analysis conclusion that should be supplemented by runtime measurements before making irreversible architecture decisions.

### Multiplayer

- No online gameplay networking was identified in the analyzed version.
- The original Multiplayer feature was reported to be local/same-device multiplayer.
- The suggested online approach is host-authoritative state snapshots plus reliable events.
- Two proposed prototypes were documented:
  - Prototype A: movement/player-state synchronization.
  - Prototype B: melee, collision, and combat synchronization.
- These are designs, not implemented network prototypes. No online networking was implemented as part of the documented analysis phase.

## 4. Existing Research Documents

The current research documentation includes the following files under `docs/extralives-swf-analysis/`:

- `README.md`
- `SWF-FORMAT-AND-TOOLCHAIN.md`
- `CODEBASE-AND-CLASS-INDEX.md`
- `GAME-ARCHITECTURE.md`
- `NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md`
- `ASSETS-AND-DATA-FORMATS.md`
- `EVIDENCE-AND-OPEN-QUESTIONS.md`
- `FRAMES-TXT-ANALYSIS.md`
- `CHARACTERS-TXT-ANALYSIS.md`
- `DATA-FLOW-AND-DEPENDENCIES.md`
- `DETERMINISM-AND-GAME-LOOP.md`
- `LOGICAL-GAME-STATE-MODEL.md`
- `ONLINE-MULTIPLAYER-ARCHITECTURE.md`
- `ONLINE-MULTIPLAYER-PROTOTYPE.md`
- `INVESTIGATION-PROGRESS.md`

Also read the next-phase instructions in `prompts/EXTRALIVES-NEXT-PHASE.md`. They require recovery of existing work, investigation of real files and their consumers, incremental documentation, validation, and a durable checkpoint.

## 5. Important Potential Research Gaps

These are areas worth auditing against existing documents and source evidence. Do not assume they are wholly unexplored just because they are listed here.

### Sprite and graphics inventory

- Enumerate exported graphics, bitmaps, vector shapes, symbols, MovieClips, and relevant asset IDs.
- Determine how characters are composed from parts, if applicable.
- Investigate layering, draw order, color transformations, flipping, scale, and depth.
- Extract images to usable formats where technically possible.
- Identify duplicated, variant, dynamically generated, or conditionally loaded assets.
- Build an asset index and visual contact sheets when practical.

Possible document: `SPRITE-AND-GRAPHICS-ANALYSIS.md`.

### Animation and pose reconstruction

- Map frame/pose records to actual animation sequences and state transitions.
- Identify idle, walking, running, attacks, defense, falling, recovery, and other animations where evidence permits.
- Trace how pose fields and limb subframes are consumed by ActionScript.
- Investigate playback rate, interpolation, transitions, facing direction, and weapon animation.
- Attempt to reconstruct a small number of real animations from extracted data and compare them with the original game.

Possible document: `ANIMATION-SYSTEM-ANALYSIS.md`.

### Character rendering pipeline

- Trace transformation from pose coordinates into screen/world coordinates.
- Establish limb draw order, depth behavior, weapon attachment, facing flips, color customization, and asset composition.
- Distinguish the visible rendered geometry from gameplay collision geometry.

Possible document: `CHARACTER-RENDERING-PIPELINE.md`.

### Collision, combat, and physics

- Trace collision checks and distinguish body/environment collisions from melee range, weapon geometry, damage, knockback, and hit reactions where the code supports these distinctions.
- Investigate gravity, jumping, falling, terrain, and obstacle interactions.
- Determine which systems must be authoritative on a multiplayer host.

Possible document: `COLLISION-PHYSICS-AND-COMBAT.md`.

### Complete asset dependency inventory

- Connect asset IDs, exported symbols, classes, loaders, and gameplay systems.
- Identify dynamic or conditional asset use and resources not yet explained by the current docs.

Possible document: `ASSET-INVENTORY-AND-DEPENDENCIES.md`.

### AI and state machines

- Trace NPC decision-making, behavior modes, combat decisions, relationships, and state transitions.
- Separate AI decision-making from movement and rendering where possible.

Possible document: `AI-AND-STATE-MACHINE-ANALYSIS.md`.

### Save system and persistence

- Document saved fields, save/load/reset flows, transient match state, and state that persists between sessions.
- Check how local save assumptions affect a web port or multiplayer version.

Possible document: `SAVE-SYSTEM-AND-PERSISTENCE.md`.

Before creating any of these documents, inspect the existing reports and `INVESTIGATION-PROGRESS.md` to avoid duplicating work. Create new documents only where they add verified evidence or useful analysis.

## 6. Browser and Telegram Mini App Feasibility

The user is interested in making the game accessible in a browser and potentially opening it as a Telegram Mini App.

### Route A: Run the existing SWF with Ruffle

First determine whether the exact `Game.swf` works sufficiently in Ruffle.

Test at minimum:

- Startup and loading.
- Main menus and scene transitions.
- Character creation/customization.
- Core gameplay, movement, attacks, collision, and AI.
- Sound, save/load, and local multiplayer if applicable.
- Performance on a mobile browser.
- Input and display scaling.

Record exact failures and unsupported APIs. Do not infer full compatibility from a successful splash screen or menu.

If compatibility is good enough, this may provide the fastest browser demonstration. It does not itself add online multiplayer.

### Route B: Rebuild a web client

If the SWF runtime is incompatible or too restrictive, a web client could be implemented using JavaScript and Canvas/WebGL, with extracted assets and researched game behavior.

This could offer better control over:

- Touch controls and mobile layout.
- Rendering and animation.
- Game-state ownership.
- WebSocket networking.
- Telegram Mini App integration.

However, this may require substantial reimplementation. Static documentation alone does not make the original ActionScript runtime automatically portable.

### Route C: Staged migration

The preferred low-risk investigation sequence is:

1. Preserve and verify the original APK/SWF and current research.
2. Test the exact SWF in Ruffle on desktop-class and mobile browsers where possible.
3. Document compatibility and blockers.
4. Decide whether to retain the SWF runtime or build a web client.
5. Establish a minimal browser-hosted playable test.
6. Add touch controls and responsive layout.
7. Only then integrate Telegram Mini App launch and identity.
8. Introduce networking through isolated prototypes.
9. Expand synchronization to combat and other authoritative state only after tests.

This is a recommendation, not a final user decision.

## 7. Possible Online Multiplayer Architecture

The current proposal is host-authoritative state snapshots plus reliable events, because independent clients are unlikely to remain synchronized when the original game has variable-rate simulation and unseeded randomness.

A potential architecture is:

- Browser/Mini App clients handle input, local presentation, and rendering.
- A server or authoritative host owns shared gameplay state.
- Clients send input/commands rather than claiming final outcomes.
- The authority broadcasts state snapshots for positions and other replicated state.
- Important discrete outcomes such as hits, damage, knockback, deaths, and match transitions are sent as reliable events.
- Clients interpolate remote presentation between received states when useful.
- The system must handle latency, disconnects, late joins, and reconciliation explicitly.

This is a starting hypothesis. The correct replication boundary, tick/update rates, payloads, and server simulation strategy must be chosen from measurements and prototypes.

### Prototype A: movement and player-state sync

Suggested minimum experiment:

- Two clients connect to one test server.
- Each controls a simple character.
- The authority tracks position, velocity or movement state, facing, and relevant animation/state identifiers.
- Snapshots are broadcast to all clients.
- Measure latency, jitter, drift, bandwidth, and visual smoothness.
- Test reconnects and late joins.

### Prototype B: combat and collision sync

Suggested follow-up experiment:

- Use a small, isolated combat test.
- Send attack intents to the authority.
- Determine hits and damage authoritatively.
- Broadcast results and resulting state.
- Check that all clients agree on hit, damage, knockback, and defeat outcomes.

Do not start with a full port of all game systems. Use the prototypes to identify what can be reused and what must be rebuilt.

## 8. Telegram Mini App Considerations

A Telegram Mini App is a web application opened from Telegram. It is a possible presentation/distribution layer, not a multiplayer backend.

A likely arrangement would include:

- A Telegram bot or launch link.
- A hosted HTTPS web page for the game client.
- Telegram Web App integration for launch context and identity, with appropriate validation.
- A WebSocket-capable multiplayer backend hosted separately.
- Server-managed room/session state if multiplayer rooms are needed.

Potential work includes touch controls, viewport and orientation handling, loading time, mobile performance, browser storage limitations, reconnect behavior, and secure validation of Telegram-provided identity data.

Do not assume the bot can host the game simulation or that Telegram provides a multiplayer server. The hosting and network backend are separate responsibilities.

## 9. Suggested Next Actions

Recommended order, subject to the current checkpoint and actual repository state:

1. Read `INVESTIGATION-PROGRESS.md`, `EVIDENCE-AND-OPEN-QUESTIONS.md`, the asset/data reports, and `prompts/EXTRALIVES-NEXT-PHASE.md`.
2. Inspect Git status, current branch, recent commits, and existing uncommitted work. Do not overwrite or repeat existing work.
3. Audit what is already known about SWF graphics, symbols, MovieClips, and animation.
4. Create a concrete asset inventory and test whether graphics can be extracted and previewed.
5. Reconstruct a small number of animations from pose data if feasible.
6. Independently test the exact SWF in Ruffle and document compatibility.
7. Based on that result, decide between direct SWF hosting and a rebuilt web client.
8. Implement movement synchronization as a small, separate prototype.
9. Implement a separate combat/collision synchronization prototype only after the movement test produces measurable results.
10. Update the checkpoint and relevant docs after each meaningful stage; commit and push validated work.

Heavy analysis is acceptable when useful. The priority is reliable, cumulative progress rather than minimizing computation.

## 10. Evidence and Decision Rules

For all future work:

- Separate confirmed facts, static-analysis inferences, runtime observations, hypotheses, and proposals.
- Cite the exact files/classes/functions/data records that support technical claims whenever possible.
- Do not describe a proposed prototype as implemented.
- Do not claim browser or Ruffle compatibility until the exact SWF has been tested.
- Do not claim a full online multiplayer implementation until end-to-end tests demonstrate it.
- Preserve original research assets and do not replace or modify the original APK/SWF as part of analysis.
- Avoid committing generated dumps or large redundant binary files without a clear reason.
- Keep project documentation in English; communicate progress to the user in Persian.
- Maintain `INVESTIGATION-PROGRESS.md` as the canonical handoff checkpoint.
- Commit and push useful stages so progress survives model/provider changes.

## 11. Current Status Summary

Based on the latest reported research phase:

- Static analysis and the data/multiplayer documentation phase was reported complete.
- Latest reported research commit: `05dadcc`, message: `docs: complete Extra Lives data and multiplayer analysis`.
- The report stated that `origin/main` matched that commit and the working tree was clean at the end of that task.
- The current state must be rechecked in Git before continuing; this document may itself be committed after that reported commit.
- Online multiplayer is not implemented.
- Browser/Ruffle compatibility is not yet established by the information currently available.
- Telegram Mini App integration is not yet implemented.
- Sprite/graphics inventory and animation reconstruction remain important areas to audit and potentially investigate further.

This document is intended to help a new model continue from the repository evidence rather than relying on conversational memory alone.
