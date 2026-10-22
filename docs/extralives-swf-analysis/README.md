# Extra Lives 1.170.64 — SWF Analysis

Static reverse-engineering notes for **Extra Lives** by MDickie, packaged as an Adobe AIR Android application. This directory documents the SWF runtime, decompiled ActionScript, game systems, data formats, and the feasibility of adding networked multiplayer.

- **App version:** `1.170.64`
- **AIR SDK / namespace:** AIR `51.1`
- **SWF version:** `51` (`0x33`), signature `CWS` (zlib-compressed)
- **ActionScript:** ActionScript 3 / AVM2, emitted as a single `DoABC2` tag
- **Document class:** `Game_fla.MainTimeline`
- **Analysis date:** 2026-10-22
- **Analysis type:** static only. No runtime instrumentation, packet capture, or patched build was used.

> Important: the SWF inside this repository (`assets-research/extralives/apk-extracted/assets/Game.swf`) is corrupt. The intact official APK was obtained separately and used **only** in `/tmp`. No APK, SWF, extracted asset, decompiled source tree, or binary is committed by this analysis. See [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md#source-integrity-and-the-corrupt-repository-copy).

## Executive summary

1. **This is a local, single-process game.** Almost all gameplay is implemented in one 116k-line ActionScript class, `Game_fla.MainTimeline`, which owns the world, characters, animation state, AI, combat, camera, UI, and save data.
2. **There is no gameplay networking code.** No `Socket`, `NetConnection`, `NetStream`, `XMLSocket`, `WebSocket`, `NetGroup`, RTMFP, or `RemoteObject` use was found. The only URL loader is an HTTP text fetch for ad/instruction configuration. Billing and store links are also network-facing, but they do not synchronize gameplay.
3. **“Multiplayer” is local same-device multiplayer.** The option is `Off / Versus / Co-op / Mystery`; it requires a gamepad (`optGamepad > 0`) and assigns local controller slots to up to four players. It does not create a network session, lobby, or remote simulation.
4. **The simulation is variable-rate and not deterministic as shipped.** Characters are 2D limb sprites moved in a 3D-ish coordinate/ground model, depth-sorted each frame. The loop is frame-counted, unseeded `Math.random` is pervasive, and rendering/sound consume the same random sequence. The active area is simulated; off-screen characters are abstract world state.
5. **Data is external and pipe-delimited.** `Characters.txt` contains 200 character records × 327 fields; `Frames.txt` contains 1,401 pose records (`0..1400`) × 13 limbs × 7 values. Pose data affects combat geometry, not only appearance. Saves live in local `SharedObject`s.
6. **Online multiplayer is not a configuration switch.** The defensible baseline is host-authoritative snapshots/events, not deterministic lockstep. It would require a new transport, fixed authoritative tick, logical-state extraction, input sequencing, snapshots, reconciliation, and session work. See the new architecture and prototype documents.

## Documents

| Document | Contents |
|---|---|
| [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md) | File hashes, repair path, SWF/ABC format, tag inventory, FFDec workflow, reproducibility and limitations |
| [CODEBASE-AND-CLASS-INDEX.md](CODEBASE-AND-CLASS-INDEX.md) | Class inventory, `MainTimeline` structure, key methods and line references, symbol classes |
| [GAME-ARCHITECTURE.md](GAME-ARCHITECTURE.md) | Startup/scene flow, frame loop, world model, entities, AI, combat, animation, camera, persistence |
| [FRAMES-TXT-ANALYSIS.md](FRAMES-TXT-ANALYSIS.md) | Full `Frames.txt` grammar, field meanings, render/combat consumers, anomalies and unused frames |
| [CHARACTERS-TXT-ANALYSIS.md](CHARACTERS-TXT-ANALYSIS.md) | Full `Characters.txt` schema, relationships, stats, costumes, behavior IDs, save overrides, anomalies |
| [DATA-FLOW-AND-DEPENDENCIES.md](DATA-FLOW-AND-DEPENDENCIES.md) | Startup/load order, text/SWF/save/runtime dependency graph, content-integrity implications |
| [DETERMINISM-AND-GAME-LOOP.md](DETERMINISM-AND-GAME-LOOP.md) | Verified tick order, variable frame rate, RNG coupling, active/off-screen boundary, determinism tiers |
| [LOGICAL-GAME-STATE-MODEL.md](LOGICAL-GAME-STATE-MODEL.md) | Authoritative/derived/presentation/client/persistent state and a network-ready entity model |
| [NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md](NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md) | Network/API audit, local multiplayer behavior, online feasibility and implementation options |
| [ONLINE-MULTIPLAYER-ARCHITECTURE.md](ONLINE-MULTIPLAYER-ARCHITECTURE.md) | Architecture comparison, recommended host-authoritative design, transport/tick/security decisions |
| [ONLINE-MULTIPLAYER-PROTOTYPE.md](ONLINE-MULTIPLAYER-PROTOTYPE.md) | Prototype A movement sync, Prototype B melee/combat sync, protocol schemas, tests and metrics |
| [ASSETS-AND-DATA-FORMATS.md](ASSETS-AND-DATA-FORMATS.md) | Bitmaps/sounds/fonts, `Characters.txt`, `Frames.txt`, `SharedObject` schemas, AIR extensions |
| [EVIDENCE-AND-OPEN-QUESTIONS.md](EVIDENCE-AND-OPEN-QUESTIONS.md) | Evidence table, confidence levels, unknowns, limitations, next experiments |

## Confidence labels used

- **Verified** — directly observed in the intact SWF, exported AS, data file, or manifest.
- **Strong inference** — follows from multiple verified facts, but was not executed dynamically.
- **Hypothesis** — plausible interpretation that needs runtime or additional data to confirm.
- **Unknown** — not established by this analysis.

## Trust boundary

The repository copy is useful as a pointer to the packaged app but is not a complete SWF. All structural claims in these documents are based on the intact official APK and its intact SWF, whose hashes are recorded in [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md#verified-hashes). The official APK was never copied into the repository.

## Quick verification commands

```bash
# Repository copy (expected to fail integrity checks)
sha256sum assets-research/extralives/apk-extracted/assets/Game.swf

# Intact copy used for analysis (outside the repository)
sha256sum /tmp/extralives-swf-analysis/extralives_official.apk
sha256sum /tmp/extralives-swf-analysis/official/assets/Game.swf
sha256sum /tmp/extralives-swf-analysis/official/Game_full.swf
```

Expected intact hashes are listed in [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md#verified-hashes).
