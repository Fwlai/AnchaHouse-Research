# Investigation Progress

**Status:** Complete — committed and pushed
**Last updated:** 2026-10-22
**Branch:** `main`
**Starting HEAD:** `e55db84` (`docs: add next-phase Extra Lives investigation prompt`)

This file is the durable checkpoint for the second-phase Extra Lives investigation. If work resumes after an interruption or model/provider change, read this file first and continue from the first incomplete stage.

## Initial workspace audit

- Working tree was clean at the start of this phase; local `main` matched `origin/main` at `e55db84`.
- Existing first-phase docs are present under `docs/extralives-swf-analysis/`.
- Existing analysis artifacts are still available outside the repository under `/tmp/extralives-swf-analysis/`:
  - intact official APK: `/tmp/extralives-swf-analysis/extralives_official.apk`
  - intact extracted SWF: `/tmp/extralives-swf-analysis/official/assets/Game.swf`
  - decompressed SWF: `/tmp/extralives-swf-analysis/official/Game_full.swf`
  - FFDec tag dump: `/tmp/extralives-swf-analysis/official/tags.txt`
  - 681 exported AS3 scripts: `/tmp/extralives-swf-analysis/export_scripts/scripts/`
  - main decompiled class: `/tmp/extralives-swf-analysis/export_scripts/scripts/Game_fla/MainTimeline.as`
  - function index: `/tmp/functions.txt`
- FFDec `22.0.2`, OpenJDK `21.0.12`, Python `3.13.5` remain available.
- Repository data files are readable; the repository `Game.swf` remains corrupt and unchanged.
- No files outside the intended documentation directory have been modified in this phase.

## Canonical data files

Both the repository extraction and the intact official APK contain byte-identical data files:

| File | SHA-256 | Size |
|---|---|---:|
| `Characters.txt` | `7acb482fc4f540a33bd2df382cc31e8f0ab392f4ff4e832e7376290b7cec6ec1` | 211,581 bytes |
| `Frames.txt` | `850e753965907c73e69dfae5e5e2f7726c5405f296ab8078f5ae4a758d956538` | 320,860 bytes |

Repository paths:

```text
assets-research/extralives/apk-extracted/assets/Characters.txt
assets-research/extralives/apk-extracted/assets/Frames.txt
```

The official APK copies have the same hashes, so the extracted data is not affected by the `Game.swf` corruption. `Frames.txt` is the canonical file for this phase.

## Stage plan

1. **Stage 0 — workspace audit and checkpoint:** complete.
2. **Stage 1 — `Frames.txt` complete analysis:** complete.
3. **Stage 2 — `Characters.txt` and cross-file identifiers:** complete.
4. **Stage 3 — execution paths, animation/state consumers, determinism:** complete.
5. **Stage 4 — logical state model and architecture comparison:** complete.
6. **Stage 5 — minimal prototype plan:** complete.
7. **Stage 6 — cross-check, summary updates, commit/push:** in progress.

## Stage 1 result

Deliverable: `docs/extralives-swf-analysis/FRAMES-TXT-ANALYSIS.md`.

Verified findings recorded there:

- `Frames.txt` is a single-line ASCII pipe-delimited table.
- Declared value `1400` is a maximum frame index; the loader reads indexes `0..1400`, i.e. 1,401 records.
- Each record is 13 limbs × 7 fields (`X,Y,Z,A,S,D,F`) = 91 integer values.
- The loader is `FramesLoaded`; `LoadFrames` runs once after default character data and has a loading-screen fallback.
- `RenderFrame` is the primary consumer, converting static fields into runtime limb position, rotation, scale, depth, flipping, and MovieClip subframes.
- `F` is a limb subframe selector, not a global frame index. It also selects bind-angle/distance entries for children of limbs 1 and 2.
- `D` flips limb angles/scales and directly feeds weapon direction.
- Frame-derived limb positions are used by combat impact checks, so the table is not purely cosmetic.
- `pAnimFrame` literal assignments reach a maximum of `1346`; frames `1347..1400` duplicate frame 0 and function as unused/default filler in normal paths.
- 13 frames contain a duplicate `Z=5` rank on limbs 3 and 7; this is recorded as an anomaly, not asserted to be a bug.
- Twenty duplicate-record groups exist, including the 55-record frame-0 group and 19 two-frame pairs.
- All values are structurally valid for their observed ranges.

## Stage 2 result

Deliverable: `docs/extralives-swf-analysis/CHARACTERS-TXT-ANALYSIS.md`.

Verified findings recorded there:

- `Characters.txt` has 200 records × 327 fields and 65,402 split tokens including count and trailing empty.
- Character identity is positional; names are display text, not stable keys.
- Record layout: name, gender, team, four relationship IDs, six stats, two 143-field costume blocks, nine moves, four ground moves, five attacks, five crushes, two stances, trigger, flight, prop.
- Costume blocks are interleaved: 14 scales plus 32 × (limb ID, RGB), then eyes.
- All relationship IDs are valid character ordinals or zero; no self-relations. Many file relationships are not reciprocal, but `AssumeRelationships` writes both directions and `GenerateRelationships` may add random ones.
- Team IDs `0..8` map to Loner, Master, Native, Survivor, Civilian, Protester, Fighter, Soldier, Rogue.
- Six stats exist; stats 1–5 are labelled Reputation, Strength, Toughness, Speed, Skill. The sixth has no explicit label in `textStat`.
- Strength/Toughness in the file reach 125 even though later stat mutation clamps to 99.
- Body-limb IDs all fit their SWF costume arrays. Hair ID 103 appears for characters 27 and 94, above the editor's hair maximum 89; recorded as an anomaly.
- RGB values are Flash color multipliers, not 0–255 channels, and legitimately exceed 1.0.
- 127 costume-2 scale values are fractional; the loader casts them with `int(...)`, so fractional parts are discarded under standard AS3 coercion.
- Move/attack/crush/ground IDs are animation/action IDs, not `Frames.txt` indexes.
- `charStance` indexes `libStance`; `charFlight` maps to None/Jump/Extended Arms/Missile/Wings; `charProp` is a preferred weapon type.
- Local `SharedObject` character saves can override nearly every loaded field, so two installations are not guaranteed to share character state.

## Stage 3 result

Deliverables:

- `docs/extralives-swf-analysis/DETERMINISM-AND-GAME-LOOP.md`
- `docs/extralives-swf-analysis/DATA-FLOW-AND-DEPENDENCIES.md`

Verified findings recorded there:

- `GameScreen(event)` is the gameplay `ENTER_FRAME` handler and runs one variable-rate tick per display frame.
- `ManageTime` sets `stage.frameRate` to the configured rate, `60` during speed-up, and `2` in one debug path. There is no fixed timestep or delta-time accumulator.
- `PlayerCycle` processes slots in numeric order, uses `1 + FindSuperSpeed(cyc)` inner loops, and cancels remaining loops when high-numbered animation frames change.
- `GetInput`/`TranslateInput` are local, unsynchronized polls. AI drives slots without a human controller.
- `RenderFrame` runs inside the player update and writes limb geometry later used by combat, so rendering is not a read-only pass.
- Randomness uses unseeded `Math.random` through wrappers and hundreds of direct calls. Static text scan: 405 `Math.random` occurrences and 1,738 `Rnd(` occurrences. Presentation/sound calls also consume the same random sequence.
- No deterministic PRNG, replay log, input tick, state hash, or fixed network tick exists.
- Active-area entities are fully simulated; off-screen characters are abstract global state with relocation/population logic.
- Startup order is `frame1` → `LoadDefaultChars` → `LoadDeviceData` (save overrides) → `LoadFrames` → match setup.
- `Characters.txt` and `Frames.txt` are independent positional files. Character ordinal is identity; costume limb IDs select SWF clips; action IDs eventually select frame indexes.
- Pose data affects hit geometry and weapon direction, not only appearance.
- Local `SharedObject` character saves can override nearly all static character fields.
- A future online mode should be host-authoritative rather than deterministic lockstep unless a separate determinism refactor succeeds.

## Stage 4 result

Deliverables:

- `docs/extralives-swf-analysis/LOGICAL-GAME-STATE-MODEL.md`
- `docs/extralives-swf-analysis/ONLINE-MULTIPLAYER-ARCHITECTURE.md`

Key design decisions recorded there:

- Separate static content, persistent state, authoritative simulation, derived simulation, presentation, and client-local state.
- Player slots need stable entity/player IDs; the existing `cyc` array index is not a network identity.
- High-frequency authority state is transform/action/health; pose geometry, display depth, sprites, particles, and UI are derived/presentation.
- Host-authoritative snapshots + reliable events is the recommended baseline.
- Deterministic lockstep is rejected unless a separate fixed-tick/seeded-RNG refactor is proven.
- Fixed 30 Hz authority tick, 10–20 Hz snapshots, input sequencing, local movement prediction, remote interpolation, and content-hash handshake are the proposed core.
- Remote-play/streaming is possible but infrastructure-heavy; dedicated servers are the competitive target.

## Stage 5 result

Deliverable: `docs/extralives-swf-analysis/ONLINE-MULTIPLAYER-PROTOTYPE.md`.

Plan recorded there:

- **Prototype A:** 1v1 movement/state sync, fixed-tick headless authority, input commands, snapshots, prediction/reconciliation, interpolation, reconnect and content-mismatch tests.
- **Prototype B:** melee combat authority using pose-derived hit geometry, reliable attack/hit/damage/death events, block/hitstun, duplicate-input/loss tests, health/round convergence.
- Explicit exclusions: matchmaking, accounts, campaign/world transitions, NPCs, weapons/projectiles, vehicles, grapples, spectator/chat, production UI.
- Decision gates G0–G6 and a first build/headless feasibility spike.
- No networking was implemented, as required.

## Stage 6 result

Verification completed:

- All 15 Markdown documents checked; no broken relative links or unbalanced code fences.
- Markdown tables checked for consistent pipe counts.
- `git diff --check` passed.
- Only documentation files are modified/added; no SWF, APK, data, decompiled source, cache, or temporary analysis file is staged.
- Canonical data hashes re-verified:
  - `Characters.txt` = `7acb482fc4f540a33bd2df382cc31e8f0ab392f4ff4e832e7376290b7cec6ec1` (211,581 bytes)
  - `Frames.txt` = `850e753965907c73e69dfae5e5e2f7726c5405f296ab8078f5ae4a758d956538` (320,860 bytes)
- Existing summary/index/evidence docs updated only where the new evidence warranted changes.
- Commit contains documentation only; pushed to `origin main` and remote state verified.

## Blockers

None.
