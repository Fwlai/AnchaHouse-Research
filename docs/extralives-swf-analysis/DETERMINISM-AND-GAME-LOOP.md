# Determinism and the Game Loop

**Scope:** how Extra Lives 1.170.64 advances simulation, samples input, consumes randomness, and why the shipped build is not a deterministic-lockstep candidate without substantial refactoring.

**Evidence:** static inspection of the intact official SWF's decompiled `Game_fla/MainTimeline.as`. No runtime replay, frame-rate experiment, or cross-device comparison was performed.

## Confidence labels

- **Verified:** explicit code path or data fact.
- **Strong inference:** follows from several verified facts; not runtime-observed.
- **Hypothesis:** plausible but unproven.
- **Unknown:** requires runtime instrumentation.

## 1. Bottom line

The game is a **variable-rate, frame-counted, presentation-coupled simulation**.

It has:

- no fixed simulation timestep;
- no accumulated delta time;
- no deterministic PRNG seed;
- no input tick/sequence numbers;
- no replay capture or state hashing;
- rendering and sound code interleaved with simulation;
- hundreds of direct `Math.random` calls;
- global off-screen state alongside fully simulated on-screen entities.

The same match on the same device can therefore differ between runs even with similar input. Cross-device deterministic lockstep is **not supported by the current architecture** and would require replacing timing, randomness, and presentation coupling before networking could rely on it.

This does not prevent online play. It means the defensible architecture is **host-authoritative simulation with snapshots/events**, not deterministic peer lockstep.

## 2. Tick model

### 2.1 `ENTER_FRAME` is the clock

`GameScreen(event:Event)` is registered as the `ENTER_FRAME` listener for the gameplay scene. Each display-list frame calls it once.

There is no separate physics thread, fixed-step accumulator, or network tick. Flash/AIR invokes `ENTER_FRAME` at the stage frame rate, subject to device scheduling and rendering load.

**Verified.**

### 2.2 `stage.frameRate` is changed at runtime

`ManageTime` sets `stage.frameRate` every gameplay frame:

- normal play: `optFrameRate`;
- speed-up (`gamSpeedUp == 2`): `60`;
- debug key path (`optDevice == 0`, `keyF > 0`): `2`;
- otherwise restores `optFrameRate` when the debug condition ends.

The SWF header declares 15 fps, and `optFrameRate` is the user-configurable normal rate. Speed-up is a frame-rate increase, not a carefully scaled fixed-step simulation.

This means a “tick” is not a unit of wall-clock time. Many timers decrement by one per `ENTER_FRAME`, while movement is also applied per frame. Changing frame rate changes the real-time speed of the simulation and can change gameplay outcomes when input arrives at different wall-clock moments.

**Verified.**

### 2.3 No delta-time integration

Player positions, bounce, gravity, falling, animation timers, attack timers, blocking timers, and AI decisions are mutated directly in the per-frame path. The code does not compute `elapsedSeconds` and multiply movement by it.

Representative example:

```text
faller = (pGround[cyc] - pY[cyc]) / 6
pY[cyc] += faller
```

The `6` is a per-frame convergence factor, not a time-scaled velocity.

**Verified** for the player update path. **Strong inference:** the same frame-counted style is pervasive across combat, animation, and world systems.

## 3. Verified main-loop order

`GameScreen` runs approximately in this order while unpaused and while the match is not paused:

1. key timer decrement
2. `PauseProcess`
3. `ManageMusic`
4. `ResetText`
5. `ManageTime`
6. `ManageCamera`
7. optional lighting
8. optional cages
9. optional cells
10. optional time logs
11. vehicles
12. atmosphere
13. optional animals
14. gestures
15. population spawn roll
16. `PlayerCycle(0)`
17. items
18. weapons
19. tunnels
20. bullets
21. explosions
22. spurts
23. pools
24. health meter / clock UI
25. `ManagePromos`
26. `DepthSorting`
27. blackout / death / win checks

The exact order matters because systems mutate shared arrays and consume the same RNG sequence. Population spawning happens before player updates; projectiles update after players; promotions happen after combat; depth sorting happens after all movement.

**Verified** from `GameScreen`.

## 4. `PlayerCycle` order

`PlayerCycle(initiate)` iterates active slots `cyc = 1..no_plays` in numeric order. Slot order is therefore itself a determinism input.

For each slot:

1. compute `no_time_loops = 1 + FindSuperSpeed(cyc)`;
2. repeat the inner simulation loop;
3. if not an AI-only or special path, call `GetInput(cyc)`;
4. call `TranslateInput(cyc)`;
5. run `ManageAnimations(cyc)`;
6. `EnforceBlocks(cyc)`;
7. update bounce state;
8. apply falling/ground logic;
9. update depth (`pZ`, `pFilterZ`);
10. call `MonitorStatus(cyc)`;
11. call `ManageTransformations(cyc)`;
12. break remaining time loops if the animation frame changed and the animation ID is in the `1000..1399` range;
13. after the simulation loop, render every active slot with `RenderFrame(cyc)`, update facial expressions, shadows, and depth records.

`FindSuperSpeed` grants extra inner loops based on speed stat thresholds (`110`, `130`, `150`, `170`, `190`). This is a second, non-uniform update-rate layer inside the already variable frame loop.

**Verified.**

### Why the time-loop break matters

High-numbered animation states (`pAnim >= 1000`) represent attacks, grapples, transformations, and other fixed sequences. When a new animation frame is selected, the code cancels the remaining inner loops. The number of simulation substeps a character receives can therefore depend on which animation was active and whether the frame changed.

This makes “same input = same outcome” conditional on exact per-frame state, not merely on player intent.

## 5. Rendering and simulation are coupled

`RenderFrame` is called from `PlayerCycle` after the movement/animation portion of the update. It is not a separate read-only render pass.

`RenderFrame`:

- writes `pLimbX/Y/Z/A/S`;
- modifies limb MovieClips (`rotation`, `scaleX`, `scaleY`, transforms, frame labels);
- applies wounds, accessories, holds, and color transforms;
- computes display depth;
- registers/updates sprites for `DepthSorting`.

Combat code later reads runtime limb positions such as head height from the rendered pose. Therefore the pose update is both visual and gameplay-relevant.

**Verified.** This coupling is a major obstacle to a clean “simulation only” replay because the simulation depends on presentation-derived geometry.

## 6. Randomness

### 6.1 No deterministic RNG

Randomness comes from ActionScript `Math.random()`. There is no seeded PRNG, state snapshot, replay seed, or RNG tick counter.

The class wraps it in:

- `Rnd(min, max)` — rounded integer range;
- `RndDec(min, max)` — floating range;
- `RndOr(a, b)` — binary choice;
- `RndArray(array, min)`;
- `RandomBetween` / `RandomDecimal`.

A static text scan found:

- **405** occurrences of `Math.random`;
- **1,738** occurrences of `Rnd(`;
- **253** of `RndDec(`;
- **118** of `RndOr(`.

These are textual occurrence counts, not unique logical call sites, but they demonstrate that randomness is pervasive.

**Verified count from the exported source; exact runtime distribution was not measured.**

### 6.2 Randomness is used for gameplay and presentation

Gameplay examples:

- AI target/attack selection;
- spawn and population rolls;
- attack/crush odds;
- relationships and social events;
- weapon behavior;
- damage/reaction variation;
- move selection when data is zero.

Presentation examples:

- sound pitch/selection;
- blinking and facial variation;
- particles, spurts, and atmosphere;
- reactions and promo text.

**Critical:** sound/particle randomization calls still consume the same global `Math.random` sequence. Even if sound has no gameplay effect, calling `Math.random` for sound changes the number sequence that later gameplay calls receive. A lockstep implementation would need every random draw, including cosmetic draws, to occur in identical order on every peer.

This is not a theoretical concern: `PainSound(cyc, Math.random())` and other rendering/sound paths call `Math.random` directly.

### 6.3 Randomness from data loading

The loader can randomize missing/zero IDs:

- `charMove == 0` → random valid move;
- `charAttack == 0` → random attack;
- `charCrush == 0` → random crush;
- `charGroundMove == 0` → random ground move.

The canonical data has no such zeros, but modified or partial data would produce a nondeterministic initial state before the match starts.

`GenerateRelationships` can also add random social links during startup when `optRestore > 0`.

**Verified.**

## 7. Input model

Input is polled locally through `GetInput`/`TranslateInput` and local controller state. It has:

- no input sequence number;
- no client timestamp;
- no tick alignment;
- no buffering/replay;
- no network transport.

Slots are processed in numeric order. AI drives any slot not assigned to a human controller. Local Versus/Co-op adds more human-controlled slots to the same simulation.

**Verified.**

For networking, input must be converted into discrete, numbered commands that the authority applies on a known tick. The current code has no such concept.

## 8. Active-area versus global state

The game tracks 200 character records globally, but only the current area receives full entity simulation.

- **Active area:** movement, collisions, AI, combat, animation, items, weapons, projectiles, and effects.
- **Off-screen:** characters retain location, health, relationships, team, inventory/equipment, and other world state; relocation/population logic can move them or stage encounters.

This is a hybrid simulation:

- a high-frequency local physics/combat scope;
- a low-frequency global social/world scope.

A network design cannot assume that “all 200 characters are simulated every tick.” It must define which entities are active, which are abstract, and where authority for transitions lives.

**Strong inference**, based on active-entity arrays and off-screen relocation logic.

## 9. Determinism tiers

| Tier | Question | Current status |
|---|---|---|
| D0 — static data | Are `Characters.txt`/`Frames.txt` deterministic inputs? | **Verified yes** |
| D1 — same process replay | Can a recorded input sequence reproduce a match? | **No support** |
| D2 — same device, same binary, exact frame inputs | Would repeated runs agree if RNG were controlled? | **Plausible after instrumentation**, unproven |
| D3 — cross-device, same frame rate | Do floating-point and timing paths agree? | **Unknown; likely fragile** |
| D4 — cross-device lockstep | Can peers exchange inputs only and stay synchronized? | **Not supported** |
| D5 — host-authoritative | Can one simulation define truth and replicate results? | **Feasible as a fork/prototype** |

Reasons D4 fails without refactoring:

1. variable frame rate and no fixed timestep;
2. extra time loops based on stats and animation state;
3. global unseeded RNG with presentation draws;
4. render-derived combat geometry;
5. frame-order mutation of shared arrays and entity lists;
6. local saves and modded character data;
7. floating-point and display-dependent calculations;
8. no input sequencing or rollback.

## 10. What deterministic replay would require

If the project ever wants replay or lockstep, the minimum instrumentation is:

1. **Fixed tick.** Run simulation at a fixed rate (for example 30 or 60 Hz), decoupled from rendering.
2. **Replace RNG.** Introduce a seeded PRNG with an explicit state and a draw counter.
3. **Remove presentation draws from simulation order.** Either use a separate cosmetic PRNG or make all random draws deterministic and recorded, including sound/particles.
4. **Input commands.** `{tick, playerSlot, buttons, axes, sequence}` with duplicate suppression.
5. **Deterministic entity ordering.** Stable IDs and stable iteration order for spawns, collisions, and arrays.
6. **State hashing.** Log a digest of authoritative fields every N ticks.
7. **Replay test harness.** Two runs with identical seeds/input must produce identical hashes.
8. **Render decoupling.** Ensure no gameplay branch depends on visual frame timing or display objects.

This is a refactor of the game loop, not a patch to a menu.

## 11. Measurement plan (recommended before any network work)

### Instrumentation

Add non-shipping debug instrumentation that records:

```text
tick index
gamSceneTime
stage.frameRate
per-slot input digest
RNG draw count / seed state
per-slot position, ground, animation, health
active entity counts
state hash at tick boundaries
```

### Experiments

1. **Same input, same device:** run a scripted 60-second movement/attack sequence twice. Compare state hashes and final positions.
2. **Frame-rate variation:** run the same sequence at 15/30/60 fps. Measure divergence in timers, movement, and combat.
3. **RNG isolation:** replace `Math.random` with a seeded generator in a test build. Repeat experiment 1.
4. **Presentation isolation:** mute/disable sound and particle paths. Check whether the gameplay hash changes when cosmetic random calls are removed.
5. **Cross-device:** repeat on at least two Android devices with different CPU/GPU profiles.
6. **Input jitter:** feed the same logical input at one-frame offsets. Measure outcome sensitivity.

### Acceptance threshold for lockstep

Do not choose lockstep unless repeated runs at different frame rates and devices produce identical authoritative hashes for a meaningful scenario. A single mismatch is enough to require rollback or authority.

## 12. Implications for multiplayer

### Not recommended

- Deterministic peer lockstep.
- “Send inputs and trust every client to simulate the same fight.”
- Reusing the current `ENTER_FRAME` rate as the network tick without quantization.

### Recommended

- Host-authoritative or dedicated-server simulation.
- Fixed authoritative tick independent of render frame rate.
- Clients send inputs; host resolves movement/combat and sends snapshots/events.
- Clients predict only local movement and reconcile against authoritative state.
- Presentation remains client-local and may diverge; gameplay state may not.
- Off-screen world state is host-authoritative and synchronized at low frequency or on transition.

## 13. Open questions

1. How much divergence occurs between two runs on the same device with identical input? **Unknown; needs replay instrumentation.**
2. Does AIR's `ENTER_FRAME` rate remain stable under load, or does it skip/coalesce frames? **Unknown.**
3. Are there hidden fixed-point assumptions or platform-specific float behavior in combat? **Unknown.**
4. How many active entities and projectiles exist in the worst-case arena? **Needs profiling.**
5. Does disabling cosmetic random calls materially change gameplay outcomes? **Strong inference yes; needs experiment.**
6. Can the game be recompiled from FFDec output for instrumentation? **Unknown; a separate build-pipeline investigation is required.**

## 14. Evidence index

Approximate line numbers in `/tmp/extralives-swf-analysis/export_scripts/scripts/Game_fla/MainTimeline.as`:

| Subject | Lines / symbols |
|---|---|
| `GameScreen` handler | `105575` |
| Main-loop order | `105579..105700` |
| `ManageTime` and `stage.frameRate` | `100814..100880` |
| `PlayerCycle` | `21348` |
| Time-loop creation and break | `21355..21545` |
| `FindSuperSpeed` | `23007` |
| Input | `GetInput` `8607`, `TranslateInput` `8729` |
| AI | `AI` `11622` |
| Animation state machine | `ManageAnimations` `23540` |
| Render entry | `RenderFrame` `14003`; called at `21559` |
| RNG wrappers | `3577..3608`, `3764` |
| `Math.random` direct use examples | `4803`, `5033`, `17788` |
| Spawn roll | `105613..105645` |
| Active-area simulation boundary | `PlayerCycle`, world relocation/population code |
| Save data loading | `LoadDeviceData` `3108`; `LoadDeviceChars` `19919` |

See [DATA-FLOW-AND-DEPENDENCIES.md](DATA-FLOW-AND-DEPENDENCIES.md) for the data/loop graph, [LOGICAL-GAME-STATE-MODEL.md](LOGICAL-GAME-STATE-MODEL.md) for state ownership, and [ONLINE-MULTIPLAYER-ARCHITECTURE.md](ONLINE-MULTIPLAYER-ARCHITECTURE.md) for the resulting architecture.
