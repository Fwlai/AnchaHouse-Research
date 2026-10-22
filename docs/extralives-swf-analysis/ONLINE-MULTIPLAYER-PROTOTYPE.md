# Online Multiplayer Prototype Plan

**Scope:** a minimal, testable path from the shipped offline build to host-authoritative online play. This document specifies two prototypes; it does not implement them.

**Architecture:** [ONLINE-MULTIPLAYER-ARCHITECTURE.md](ONLINE-MULTIPLAYER-ARCHITECTURE.md)

## 1. Goals

### Prototype A — movement and state sync

Prove that:

- two clients can join one headless authoritative match;
- inputs are sequenced and applied on a fixed tick;
- each client sees the other player move smoothly;
- local prediction and reconciliation stay within tolerance;
- the authority is the single source of truth for position and animation state.

**No combat, items, NPCs, promos, or campaign state.**

### Prototype B — collision and combat sync

Prove that:

- the authority resolves melee hits using the game's pose-derived geometry;
- damage, block, hitstun, death, and round state are replicated as reliable events;
- duplicate/late inputs and packet loss do not duplicate damage;
- clients can present combat convincingly despite prediction corrections.

**No weapons, projectiles, grapples, vehicles, animals, or area transitions.**

## 2. Preconditions

The prototype cannot be built by dropping a socket call into the current APK. It needs:

1. **A reproducible build path** from the decompiled ActionScript to a testable AIR/Android build, or an equivalent harness that can run the game logic. This is the first engineering spike.
2. **A logical state bridge** over the parallel arrays (`pX`, `pGround`, `pAnim`, etc.).
3. **A fixed-tick adapter** independent of `ENTER_FRAME`/`stage.frameRate`.
4. **A headless or render-disabled authority mode** that still runs movement and combat.
5. **A transport** (LAN TCP/WebSocket is acceptable for A; UDP preferred for B/production).
6. **Deterministic test fixtures**: fixed characters, fixed spawn points, fixed arena, no random spawns.

If the build path fails, the project should stop before investing in protocol work. Static analysis cannot validate a networking design on its own.

## 3. Common test configuration

Use one arena with flat ground and simple boundaries:

```text
mode:       1v1 arena
area:       fixed (for example the Arena map)
characters: fixed character IDs/costumes
spawns:     fixed left/right positions
NPCs:       disabled
items:      disabled
weapons:    disabled
promos:     disabled
weather:    fixed/disabled
round:      first to 3 or 120 seconds
tick rate:  30 Hz authority
snapshot:   15 Hz
render:     device rate, interpolation enabled
```

The authority must load the same:

- SWF/code version;
- `Characters.txt` hash;
- `Frames.txt` hash;
- rules/config.

## 4. Protocol baseline

Use a small versioned protocol. JSON is acceptable for a LAN prototype; binary encoding is expected for production.

### 4.1 Handshake

```text
ClientHello {
  protocolVersion: u16
  appVersion: string
  swfHash: bytes32
  charactersHash: bytes32
  framesHash: bytes32
  displayName: string
  capabilities: u32
}

ServerWelcome {
  sessionId: string
  playerId: string
  slot: u8
  matchId: string
  tickRate: u8
  snapshotRate: u8
  rules: MatchRules
  accepted: bool
  reason?: string
}
```

### 4.2 Input

```text
InputFrame {
  matchId: string
  playerId: string
  slot: u8
  tick: u32
  sequence: u16
  buttons: u32
  moveX: i8       -127..127
  moveY: i8       -127..127
  clientTimeMs: u32
}
```

Logical buttons:

```text
ATTACK, BLOCK, RUN, JUMP/UP, DOWN, PICKUP, USE, TAUNT
```

Use logical actions, not raw key codes.

### 4.3 Snapshot

```text
Snapshot {
  tick: u32
  snapshotId: u32
  ackInputSequence: u16
  phase: u8
  entities: [EntityState]
  removedEntityIds: [u32]
}

EntityState {
  entityId: u32
  kind: u8
  characterId: u16
  costume: u8
  team: i8
  x: f32
  y: f32
  ground: f32
  z: i16
  dir: i8
  anim: u16
  animFrame: u16
  flags: u32
  health: i16
}
```

A full snapshot is sent on join/reconnect. Normal traffic uses deltas:

```text
SnapshotDelta {
  tick, snapshotId, ackInputSequence,
  changedEntities: [EntityDelta],
  removedEntityIds: [u32]
}
```

### 4.4 Reliable events

```text
EventBatch {
  tick: u32
  events: [GameEvent]
}

GameEvent {
  eventId: u32
  tick: u32
  type: u8
  sourceEntityId?: u32
  targetEntityId?: u32
  payload: bytes
}
```

Prototype B events:

```text
ATTACK_START
HIT_CONFIRMED
DAMAGE
BLOCKED
HITSTUN
DEATH
ROUND_END
RESPAWN
```

Events are idempotent by `eventId`. Clients must not apply the same damage twice.

### 4.5 Acknowledgement and time sync

```text
SnapshotAck { snapshotId, clientTimeMs }
Ping { clientTimeMs }
Pong { clientTimeMs, serverTimeMs }
```

Use ping/pong for an RTT estimate and a simple clock offset. The client should not use wall-clock time to advance the authority simulation.

## 5. Authority loop

At each fixed tick:

1. drain and validate queued input commands;
2. reject duplicate/stale sequences and out-of-range values;
3. apply accepted inputs to player slots;
4. run the extracted movement/animation update;
5. resolve collisions and combat (B);
6. update health/death/round state;
7. emit reliable events;
8. generate a snapshot every second tick (at 30 Hz / 15 Hz);
9. record a state hash and debug trace.

The authority must never read a client's local save for authoritative state.

## 6. Client loop

At render frame rate:

1. sample local input;
2. send `InputFrame` for the next simulation tick;
3. apply local predicted movement;
4. receive snapshots/events;
5. acknowledge the latest snapshot;
6. reconcile local prediction against the authoritative state;
7. interpolate remote entities;
8. render using the existing presentation path.

### Prediction scope

Prototype A:

- predict local movement and facing;
- do not predict the remote player;
- do not predict damage.

Prototype B:

- predict local attack animation and movement;
- do **not** predict hit confirmation or health loss;
- correct immediately on `HIT_CONFIRMED`/`DAMAGE`.

## 7. Prototype A — movement/state sync

### 7.1 Minimal state

```text
tick
player slot/entity IDs
x, y, ground, z
dir
anim, animFrame, animTim
active/alive
last accepted input sequence
```

### 7.2 Minimal actions

- move left/right;
- jump/up/down where the arena supports it;
- run;
- turn/facing;
- idle/walk/run animation selection.

No attacks, blocking, pickups, or interactions.

### 7.3 Acceptance tests

| Test | Procedure | Pass criterion |
|---|---|---|
| Join | Two clients join one session | Both receive the same slot/entity mapping |
| Basic sync | Client A moves for 5 s | Client B sees the move; no teleporting at rest |
| Prediction | 100 ms artificial RTT | Local input feels immediate; correction below threshold |
| Convergence | Stop input, wait 2 s | Client and authority positions within 1–2 game units |
| Wall boundary | Run into arena edge | No client/authority boundary divergence |
| Packet loss | 5% loss, 50 ms jitter | Session remains usable; entities interpolate |
| Duplicate input | Replay an old sequence | Authority ignores it |
| Late input | Delay a command several ticks | Authority applies or drops according to policy; no desync |
| Reconnect | Disconnect/rejoin | Full snapshot restores state |
| Content mismatch | Different `Frames.txt` hash | Server rejects with a clear reason |

### 7.4 Metrics to record

- RTT, jitter, loss;
- input-to-authority latency;
- input-to-render latency;
- snapshot size and rate;
- prediction corrections per second;
- maximum/mean position error at rest;
- authority tick duration;
- client frame duration;
- reconnect duration.

### 7.5 Exit criteria

Prototype A is complete when a 10-minute 1v1 movement session remains stable under 100 ms RTT and 5% loss, with no persistent boundary or position divergence and no authority crash.

## 8. Prototype B — collision/combat sync

### 8.1 Added state

```text
action: idle | walk | attack | block | hitstun | dead
actionTick
attackMoveId
health
injured
dead
lastHitEventId
roundScore
```

### 8.2 Combat authority

The host must resolve hits from its own pose state. It should use a pure function equivalent to the game's impact logic:

```text
resolveHit(attackerState, defenderState, frameData, characterData)
  -> hit | miss | blocked
```

It must use:

- attacker limb/weapon origin;
- defender head/body geometry;
- facing (`frameD` and runtime direction);
- range/character scale;
- attack/active-frame timing;
- block state and facing.

Clients may play the animation locally but must accept the authority's hit result.

### 8.3 Event flow

```text
Client: ATTACK_START (predicted animation)
Host:   validates input and cooldown
Host:   simulates active frames
Host:   HIT_CONFIRMED + DAMAGE (reliable)
Client: plays hit reaction, updates health
Host:   DEATH when health <= 0
Host:   ROUND_END when score/time condition met
```

### 8.4 Acceptance tests

| Test | Procedure | Pass criterion |
|---|---|---|
| In-range hit | Attack just inside valid range | Exactly one hit and one damage event |
| Out-of-range | Attack just outside range | No damage |
| Facing flip | Attack after turning | Hit geometry/direction matches authority |
| Pose dependence | Compare different attack frames | Hit timing/height matches host pose |
| Block | Defender blocks facing attacker | `BLOCKED`, no or reduced damage per rules |
| Duplicate attack | Re-send same input/event | No double damage |
| Simultaneous attacks | Both attack same tick | Defined deterministic order; consistent result |
| Packet loss | Drop event batch then retransmit | Health/round state converges |
| Client disagreement | Client predicts a hit that host rejects | Correction removes phantom damage |
| Death | Damage to zero | One `DEATH`; no further actions |
| Round end | Score/time limit | All clients agree on result |
| Reconnect mid-round | Rejoin during fight | Full snapshot restores health/action/score |

### 8.5 Metrics

- false-positive/negative hits versus local baseline;
- damage-event duplicates and drops;
- correction magnitude after rejected prediction;
- time from attack input to confirmed hit;
- event retransmission rate;
- authority tick cost with combat;
- desync incidents per 100 rounds.

### 8.6 Exit criteria

Prototype B is complete when 100 scripted melee exchanges produce identical authority results across repeated runs, no duplicate damage under loss/retry, and all clients converge to the same health/round state after corrections.

## 9. Test harness

### 9.1 Scripted inputs

Create deterministic input scripts:

```text
move-right 60 ticks
attack at tick 120
block at tick 121
turn, attack, jump
```

Run the same script against:

- local simulation only;
- authority + one client;
- authority + two clients at 0/50/100/150 ms RTT;
- 1%, 5%, 10% loss;
- reconnect scenario.

### 9.2 State hash

At each authority tick, hash:

```text
tick
entity IDs and kinds
x, y, ground, z, dir
anim, animFrame
health, dead, action
round score/phase
```

Do not include sprites, sounds, camera, particles, or UI. If two runs of the same script produce the same authoritative hash sequence, the authority core is reproducible even if clients render differently.

### 9.3 Network simulator

Use a local proxy or test transport that supports:

- fixed latency;
- jitter;
- packet loss;
- duplication;
- reordering;
- disconnect/reconnect.

Do not rely on real Wi-Fi for the first correctness tests.

## 10. Scope exclusions

Explicitly out of scope for A and B:

- matchmaking and accounts;
- NAT traversal/relays;
- anti-cheat beyond authority validation;
- host migration;
- campaign progression and saves;
- area transitions;
- NPC AI, population, promos, relationships;
- vehicles, animals, cages, cells;
- weapons, bullets, explosions, tunnels;
- grappling and multi-player combos;
- spectator mode;
- voice/text chat;
- mobile background/resume hardening;
- production UI.

Adding any of these changes the state/sync design and requires a new test plan.

## 11. Milestones and decision gates

| Gate | Deliverable | Decision |
|---|---|---|
| G0 | Reproducible test build or headless harness | Continue only if game logic can run deterministically enough for authority |
| G1 | Logical state extraction + fixed tick | Measure entity state size and tick cost |
| G2 | Prototype A movement sync | Continue only if prediction/convergence is stable |
| G3 | Prototype B melee combat | Continue only if authority hit resolution is correct |
| G4 | 2v2 or survival scaling | Measure bandwidth/CPU/entity limits |
| G5 | Online session + reconnect | Decide host vs dedicated server |
| G6 | Production hardening | Security, moderation, regional infrastructure |

Relative effort is **high** for G0–G1, **high** for A, and **higher** for B because combat authority touches the core hit/animation paths.

## 12. Risks

1. **Recompilation risk.** FFDec output may not rebuild cleanly. This blocks in-client prototypes.
2. **Simulation extraction risk.** Combat is entangled with rendering and MovieClips.
3. **Hit-geometry risk.** Authority must reproduce the exact limb geometry used by the original.
4. **Latency risk.** Melee timing may require tighter prediction/interpolation than a casual brawler can tolerate.
5. **Platform risk.** Android backgrounding and network changes can break sessions.
6. **Bandwidth risk.** Naive full-state snapshots are too large at high entity counts.
7. **Cheat risk.** A player-hosted authority is not trustworthy for ranked play.
8. **Product risk.** Local multiplayer already works; the online mode must justify its infrastructure and support cost.

## 13. Recommended first experiment

Before building either prototype, do a two-week-equivalent feasibility spike:

1. rebuild a minimal AIR test app from the decompiled code;
2. run a headless 1v1 movement loop at a fixed tick;
3. log positions/actions for one scripted minute;
4. prove the same script produces the same authority hash twice;
5. measure tick time with 2, 4, and 8 active entities.

If step 4 fails even with rendering disabled, invest in determinism instrumentation before networking. If step 4 passes, proceed to Prototype A.

## 14. Final recommendation

**Prototype A is the correct first network milestone; Prototype B is the first gameplay milestone.**

Do not start with matchmaking, accounts, or campaign co-op. Do not choose lockstep. Build authority, sequencing, snapshots, and reconciliation first, then add melee combat against the authority's pose geometry.
