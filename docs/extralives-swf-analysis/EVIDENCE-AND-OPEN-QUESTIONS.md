# Evidence and Open Questions

## Evidence standard

This analysis used static evidence from the intact official APK/SWF plus the repository's data files. It did **not** execute the game under instrumentation, intercept network traffic, or build a patched client. Claims are therefore labeled **Verified**, **Strong inference**, **Hypothesis**, or **Unknown**.

The repository's SWF is corrupt, so it cannot be used as a complete source. The intact official APK was kept outside the repository and is not committed.

## Evidence table

| # | Claim | Evidence | Confidence |
|---:|---|---|---|
| 1 | Repository `Game.swf` is corrupt | Bad APK CRC (`a18c9544` expected vs `cd56cfec` actual); zlib failure at offset `4,718,600`; truncated/zeroed ranges | Verified |
| 2 | Official APK is intact and same app/version | APKPure APK SHA-256 `f710…633b`; `Game-app.xml` says `air.ExtraLives` / `1.170.64`; extracted SWF decompresses fully | Verified |
| 3 | SWF is `CWS`, version 51, 21 frames, 15 fps | Header parse; declared length `42,480,597`; reconstructed `FWS` matches | Verified |
| 4 | Code is AS3/AVM2 | Single `DoABC2` tag (1,703,915 bytes); FFDec exports AS3 classes; `MainTimeline` AS3 source | Verified |
| 5 | 681 classes/symbols | `SymbolClass` + 681 exported `.as` files; 444 `MovieClip`, 227 `Sound`, 5 `Font`, 4 `BitmapData`, 1 plain class | Verified |
| 6 | Nearly all logic is in one class | `Game_fla.MainTimeline` is 4,543,866 bytes / 116,160 lines / 789 functions / 1,497 variables | Verified |
| 7 | 21 scenes match frame scripts | `DefineSceneAndFrameLabelData` lists 21; constructor registers `addFrameScript(0..20)` | Verified |
| 8 | Main loop is `ENTER_FRAME` | Screen functions add `Event.ENTER_FRAME` listeners; `GameScreen` per-frame order observed | Verified |
| 9 | World has 64 areas | `no_areas = 64` at line 112150; map tables | Verified |
| 10 | 200 character records, 327 fields each | `no_chars = 200`; `Characters.txt` = 1 + 200×327 + trailing; loader field order | Verified |
| 11 | 1,401 frame records × 13 limbs × 7 values | `Frames.txt` token count; loader loops `0..no_saved` inclusive | Verified |
| 12 | 33 item types, 92 weapon types, 8 animal types, 32 recipes | `no_itemtypes`, `no_weaptypes`, `no_animaltypes`, `no_recipes` assignments | Verified |
| 13 | Saves are local `SharedObject`s | `SharedObject.getLocal` / save/load functions; no server endpoints | Verified |
| 14 | No gameplay network API | Systematic search found no sockets/NetConnection/NetStream/RPC/NetGroup/RTMFP | Verified absence in exported code |
| 15 | Only URL loader is ad/instruction text | `URLLoader(new URLRequest("http://www.mdickie.com/ads/extralives_instruction.txt"))` at line 109835 | Verified |
| 16 | Multiplayer is local | `textMultiplayer = Off/Versus/Co-op/Mystery`; controller slots; gated by `optGamepad`; no network session | Verified |
| 17 | “Online multiplayer” is not a hidden toggle | No session, transport, lobby, remote input, or state replication; menu only local slots | Strong inference |
| 18 | Simulation is active-area full, off-screen abstracted | PlayerCycle systems operate on active entities; world relocation/population code handles off-screen NPC locations | Strong inference |
| 19 | Frame-rate/RNG affects outcomes | `Rnd`, `RndDec`, `RndOr`, `stage.frameRate` changes, single-threaded loop | Strong inference |
| 20 | Cross-device determinism is unproven | No determinism test or runtime replay was performed | Unknown |
| 21 | Online can be added as a substantial fork | Requires transport, state model, authority, session, serialization, UI | Engineering assessment |

## Exact code locations worth citing

All paths refer to the analysis copy outside the repository:

```text
/tmp/extralives-swf-analysis/export_scripts/scripts/Game_fla/MainTimeline.as
```

| Area | Method / line |
|---|---|
| Constructor / frame scripts | `addFrameScript(0..20)`; `frame1` line 109489; `frame21` line 116051 |
| Loader/scene startup | `LoaderScreen` 103698; `FramerScreen` 103750; `TitleScreen` 104184 |
| Game loop | `GameScreen` 105575 |
| Clock/speed | `ManageTime` 100814 |
| Camera | `ManageCamera` 58951 |
| Lighting | `ManageLighting` 58762 |
| World build | `LoadWorld` 58333 |
| Population/spawning | `GameScreen` around 105614; `mapPopulation` line 112169 |
| Add/load player | `AddPlayer` 20797; `LoadPlayer` 20996 |
| Player update | `PlayerCycle` 21348 |
| AI | `AI` 11622 |
| Animation state machine | `ManageAnimations` 23540 |
| Input | `GetInput` 8607; `TranslateInput` 8729; `FindGestures` 8253 |
| Local multiplayer | `IncludeHuman` 20757; `textMultiplayer` 110117; controller setup ~115419 |
| Persistence | `LoadSystem` 3139; `LoadProgress` 3389; `SaveProgress` 3529; `LoadDeviceChars` 19919; `SaveDeviceChars` 20516 |
| Characters | `LoadedDefaultChars` 19723 |
| Frames | `FramesLoaded` 13912; `SaveFrames` 13872 |
| Network fetch | line 109835 |
| Ads/billing | billing around 3920–3950; ad setup around 4154–4276 |

## What the absence of networking does and does not prove

**Proves (within exported code):**

- No gameplay `Socket`/`NetConnection`/`NetStream`/RPC/NetGroup/RTMFP call exists.
- No serialization or remote-player synchronization path exists.
- The “Multiplayer” option uses local controller slots.
- No matchmaking/lobby/session code is present.

**Does not prove:**

- That the AIR runtime or a native extension cannot open a socket outside the AS3 source. The imported extensions are billing/core/Play services, and no gameplay path through them was found, but a full native audit was not performed.
- That the game has never had online features in another version. Only `1.170.64` was analyzed.
- That the server endpoints used by ads/billing are benign or necessary; they were not audited for privacy/security.

## Open questions

### Runtime and behavior

1. **Does the game run deterministically across devices at the same frame rate?** Need replay tests with identical inputs and logged RNG/state hashes.
2. **How does Android backgrounding affect the simulation?** Does `ENTER_FRAME` pause/resume cleanly, and are large time gaps clamped?
3. **What is the exact `gamMode` mapping?** Static conditionals strongly indicate survival vs deathmatch, but a runtime trace would confirm all mode transitions.
4. **Which “Multiplayer” values are reachable on which devices?** The code gates on gamepads and `optGamepad`; UI behavior on touch-only devices should be tested.
5. **Is `Mystery` a role assignment or a random hidden mode?** `IncludeHuman` selects a local entity by role; the full menu semantics were not exhaustively traced.
6. **What exactly do `frameA/S/D/F` mean geometrically?** The loader names them; their full animation semantics would require correlating `GetFrame`/`UpdateCostume` and rendering tests.

### Content and assets

7. **Are all 681 symbol classes used?** Some may be legacy/unreachable. A runtime symbol-usage audit is needed.
8. **Which tag/sprite corresponds to which map tile or prop?** Tag inventory is complete, but semantic mapping of all art is not.
9. **Are there unused maps, animations, or promos?** The code has stubs (e.g. empty `TriggerStorylines`) and may contain dead content.
10. **What is the exact save-field set for each slot?** Shared object names and roles are verified; a complete field-by-field save schema would require dumping runtime objects or tracing every save assignment.

### Networking and porting

11. **Would a host-authoritative model be viable at the game's entity counts?** Need profiling of active entities, projectiles, items, and per-frame state size.
12. **Can `MainTimeline` state be extracted without changing gameplay?** The class is dynamic and uses `MovieClip` references; a logical-state boundary must be designed.
13. **Does AIR provide a viable Android socket/server path in this packaging?** Not tested; a native extension or external relay may be required.
14. **Would lockstep survive floating-point and frame-rate variation?** Unknown and likely the highest-risk approach.
15. **What is the product goal?** Local same-device play is already supported; true online versus is a different product scope and cost.

## Limitations of this analysis

- **Static only.** No packet capture, debugger, memory dump, or runtime hooks.
- **Decompiler reconstruction.** FFDec source is readable but not guaranteed byte-identical to the ABC; `-onerror ignore` means isolated errors could have been skipped.
- **No recompilation.** The exported AS was not rebuilt into a SWF or compared instruction-by-instruction.
- **No exhaustive native audit.** AIR, billing, and Play services native code were not reverse-engineered.
- **No exhaustive asset semantics.** Tag counts and class families are verified; every sprite's purpose is not.
- **Line numbers are export-specific.** They refer to the FFDec 22.0.2 output and may shift with another version.

## What would change the networking conclusion

Any of the following would require revising the conclusion:

1. A runtime trace showing an undocumented socket/NetConnection opened by the game or an extension.
2. A hidden online menu path not present in the decompiled source (e.g. server-controlled config enabling a mode).
3. Native code in an extension implementing gameplay transport not visible in AS3.
4. Evidence that “Multiplayer” connects devices despite local-only code paths.

Until such evidence exists, the defensible conclusion is: **Extra Lives 1.170.64 is an offline/single-device game with local multiplayer, and online multiplayer would require a new networking architecture.**

## Reproduction notes

- FFDec `22.0.2`, OpenJDK `21.0.12`, Python `3.13.5`.
- Intact SWF: `17f2e23b…8766b`; decompressed `FWS`: `c70b6a6a…6621`.
- Script export: 681 files, ~4.79 MB.
- Tag dump: 46,670 flattened entries; 5,398 top-level tags.
- All analysis artifacts are under `/tmp/extralives-swf-analysis/`; none are committed.

See [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md) for exact hashes/commands and [NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md](NETWORKING-AND-MULTIPLAYER-FEASIBILITY.md) for the feasibility assessment.
