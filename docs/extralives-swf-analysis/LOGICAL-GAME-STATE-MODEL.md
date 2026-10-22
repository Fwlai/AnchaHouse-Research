# Logical Game State Model

**Scope:** a network-ready decomposition of Extra Lives 1.170.64 state. This is a design model, not a shipped serialization format.

**Evidence:** decompiled `Game_fla/MainTimeline.as`, data-file analysis, and static review of the entity/world arrays. No state was extracted at runtime.

## 1. Why a logical model is required

The game stores almost everything as parallel ActionScript arrays on `MainTimeline`. A player slot is not an object; it is an index into dozens of arrays. Visual entities are MovieClips referenced by other arrays.

A network layer cannot serialize the class or its display objects. It needs a **logical projection**:

- stable identities;
- plain numeric/enumerated fields;
- explicit ownership and authority;
- a distinction between stored truth and recomputed values;
- a distinction between gameplay state and presentation.

The model below is the minimum useful decomposition.

## 2. State categories

| Category | Meaning | Network treatment |
|---|---|---|
| Static content | Shipped code/data/tables | Hash/version, never stream per tick |
| Persistent state | Saves, edited characters, progress | Loaded before session; not authoritative in-match |
| Authoritative simulation | Rules-relevant mutable state | Host/server owns; replicated |
| Derived simulation | Recomputed from authoritative state | Not replicated unless needed for prediction |
| Presentation | MovieClips, frames, colors, sounds, particles | Client-local |
| Client-local | Input, UI, camera preference, options | Never authoritative |
| Session metadata | Players, slots, readiness, match config | Lobby/session service owns |

“Authoritative” is scope-dependent. A campaign host owns the world; an arena server may own only one match.

## 3. Stable identities

### Player slots

Use a session-scoped slot ID, not the array index alone:

```text
sessionId
playerId        stable account/device identity
slotId          1..4 (local array index)
entityId        unique active-match entity ID
characterId     1..200 canonical character ordinal
```

`pChar` is the canonical character ID in the current code. `cyc` is only the slot index and must not be reused as a network entity ID after slot recycling.

### Active entities

Every replicated entity needs a stable ID:

```text
entityId: uint32       monotonically allocated by authority
kind: enum             player, npc, animal, vehicle, item, weapon, projectile, effect
ownerPlayerId?         for player-owned projectiles/items
spawnTick: uint32
```

The current code uses arrays such as `pShadowID` and `no_entities`, but those are display/registration helpers, not a network identity system.

### Off-screen characters

The global character record is keyed by canonical character ID `1..200`. Since identity is positional in `Characters.txt`, use `characterId` plus a content hash. Do not key on `charName`.

## 4. Authoritative match state

The following is the proposed authoritative core for an arena match. Names map to current code arrays.

### 4.1 Match envelope

```text
tick: uint32
matchId: string
mode: enum                 deathmatch | survival | co-op
areaId: int                gamLocation
phase: enum                warmup | active | promo | ended
score / lives / timer
contentManifestHash
randomSeed?                if replay/lockstep ever added
```

### 4.2 Player/NPC entity record

```text
entityId
kind
characterId
costume
team
zombie
active

position:
  x, y, ground, z
  dir
  stagger, accel
  platformId?, platformX/Y/Z

action:
  anim, animTim, animFrame
  upperFrame, lowerFrame, perLimbFrame[]
  transform
  attackTimer, blockTimer
  grappleTarget?, grappling?, grappler?
  holding?, leftHold?, carrying?, bag?

combat:
  health, hp, dt
  injured, dead, blind, dizzy
  stat[1..6]
  attacker?, pinning?, pinner?
  weaponVictim?, itemVictim?

social/AI (NPC and AI-controlled):
  foc, agenda, anger, satisfied
  nearest?
  team/relationship references

control:
  controlType: human | ai | remote
  controllerSlot?
```

Not every field needs high-frequency replication. `characterId`, `costume`, `team`, and max health are spawn-time state. Positions and actions are high-frequency.

### 4.3 Items, weapons, projectiles, effects

```text
item/weapon:
  entityId, type, x, y, z/ground
  holder?, stockId?
  ammo, fireTimer, state
  velocity/rotation if physically active

projectile:
  entityId, type, ownerEntityId
  x, y, z, dir, speed
  lifetime, damage/effect code

explosion/pool/spurt:
  entityId, type, x, y, z
  age, lifetime, owner?
```

Effects that cannot affect gameplay should not be authoritative. `ExplosionCycle` can affect damage; `SpurtCycle`/`PoolCycle` are more presentation-oriented but were not fully classified. The prototype should start with melee only and add projectiles later.

### 4.4 World state

```text
gamSceneTime, gamClock, gamMins, gamHours, gamDay
gamLocation, gamMode, matchPlayer
area dimensions/horizon
active entity count
team bases/stock counts
promo state
win/loss counters
```

### 4.5 Off-screen character record

For campaign-style persistence/authority:

```text
characterId
location, home, x, y, dir
team, teamHistory
health, hp, dt
injured, dead, zombie
costume, holding, leftHold, bag, carrying
relationships[1..200]
knowledge/history/promo counters
```

These can be synchronized at low frequency, on area transition, or only when they become active.

## 5. Derived state

Derived fields should be recomputed by each simulation instance rather than transmitted blindly.

| Derived state | Source | Notes |
|---|---|---|
| `pScale` | `charScale[costume][0]`, screen context | Affects rendering and hit range |
| `pLimbScale` | `charScale[costume][limb]` | Costume data |
| `pLimbX/Y/Z/A/S` | Frame pose + character scale + camera/runtime offsets | Recomputed by `RenderFrame` |
| `pLimbID` | `charLimb` + costume | Sprite selection |
| `pFilterZ` | `pZ`, walls/cells/water | Display depth helper |
| current `pStat` modifiers | base stats, health, injury, temporary effects | Can be authoritative snapshots with derived display values |
| relationship matrix | relation fields + `AssumeRelationships`/generation | Static or host state, not per-tick |
| cost/weapon definitions | code tables | Static content |
| hit geometry | runtime limb positions + scale | Derived from pose and content |

### The pose exception

Limb geometry is derived, but it is **combat-relevant**. Two options:

1. **Host computes hits.** Clients do not need exact limb geometry; they receive hit events and health changes. This is recommended.
2. **Clients reproduce hits.** They need identical pose state and frame data, which reintroduces determinism requirements.

For host-authoritative play, option 1 is much cheaper and safer.

## 6. Presentation state

Purely client-local or recomputable:

- MovieClip instances and child indices;
- limb clip frame labels;
- hat/beard/hair/accessory visibility;
- wound decals and blood overlays;
- eye/mouth frames and blink timers;
- color transforms;
- shadows;
- camera transforms;
- particles, spurts, pools, weather;
- sound channels, pitch, and volume;
- UI meters, text, menus, promos;
- depth-sort order.

Some of these are *observable* consequences of authoritative state. For example, health drives a meter, and an attack drives an animation. Send the cause, not the sprite.

## 7. Client-local state

| State | Why local |
|---|---|
| Input buttons/axes | Device-specific; sent as commands |
| Input calibration | Local device configuration |
| Camera preference/zoom | Can be local if it does not affect simulation |
| Resolution/quality/audio | Local rendering settings |
| Pause menu / UI focus | Local |
| Local save slot choice | Account/device choice |
| Controller assignment in local play | Local; remote assignment is session config |

**Caution:** `optFrameRate`, `optSpeedUp`, and some display options currently influence the simulation. A networked build must move simulation timing to a fixed authoritative tick and treat these options as presentation-only (or host-decided match settings).

## 8. Persistent state

Persistent state should be split into:

### 8.1 Account/global

- player identity/display name;
- unlocks/purchases;
- settings;
- statistics.

### 8.2 World/campaign

- progress;
- world time;
- character roster and relationships;
- stocks (animals/items/weapons/tunnels);
- character edits;
- base/team state.

### 8.3 Match

Usually ephemeral:

- current positions/actions;
- projectiles/effects;
- score/timer;
- active entities.

Do not persist match state unless the product needs reconnect/resume.

## 9. State ownership and synchronization matrix

| State | Owner | Frequency | Reliability |
|---|---|---|---|
| Match phase/config | authority | on change | reliable |
| Player identity/character/costume | lobby + authority | join/change | reliable |
| Input commands | client → authority | every input tick | reliable/ordered |
| Player positions/actions | authority → clients | 10–20 Hz snapshots + events | latest-wins |
| Health/damage/death | authority → clients | on event + snapshot | reliable |
| Projectile spawn/removal | authority | on event | reliable |
| Item/weapon ownership | authority | on event | reliable |
| World clock/location | authority | low frequency | reliable |
| Off-screen characters | authority | low frequency/on transition | reliable |
| Pose/limb transforms | client | per render frame | local |
| UI/camera/sound | client | per render frame | local |
| Saves | client/account service | out-of-match | reliable |

## 10. Suggested wire shape

This is illustrative, not implemented.

```text
EntitySnapshot {
  id: u32
  kind: u8
  character: u16
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
  action: ActionState
}

WorldSnapshot {
  tick: u32
  phase: u8
  area: u8
  clock: u32
  scoreA: i16
  scoreB: i16
  entities: [EntitySnapshot]
}

InputCommand {
  tick: u32
  slot: u8
  sequence: u16
  buttons: u32
  axisX: i8
  axisY: i8
}

GameEvent {
  tick: u32
  type: u8
  sourceId: u32
  targetId: u32
  payload: bytes
}
```

Use a schema with optional fields/deltas rather than serializing all player arrays every tick.

## 11. State initialization

At match start, the authority must define:

1. content manifest (SWF/data hashes);
2. mode, area, rules, score/lives;
3. participating characters/costumes;
4. starting stats/health/equipment;
5. RNG seed if replay is desired;
6. world time and off-screen baseline;
7. entity ID allocation.

Clients should not rely on their local save for these values. They may use local data for prediction, but the authority must override mismatches.

## 12. State transitions requiring explicit messages

- match start / countdown / active;
- player join / leave / reconnect;
- character spawn / despawn;
- area transition;
- weapon pickup/drop/throw/fire;
- hit/damage/death;
- grapple start/break;
- transformation;
- promo/social event start/end;
- round/score change;
- pause/resume;
- match end.

These are better as reliable events than as inferred from snapshot diffs.

## 13. What cannot be serialized

- `MovieClip`/`Sprite`/`DisplayObject` references;
- `SoundChannel` and playback handles;
- event listeners;
- `File`/`SharedObject` handles;
- frame scripts and closures;
- dynamically attached limb/accessory clips;
- the 1,497 class variables as a whole.

A network adapter must traverse the logical arrays and emit model objects. It must never attempt generic object serialization of `MainTimeline`.

## 14. Minimal model for each prototype

### Prototype A — movement

Authoritative fields only:

```text
tick, playerId, slot, x, y, ground, z, dir,
anim, animFrame, health/active, input sequence
```

No NPC combat, items, projectiles, promos, or relationships.

### Prototype B — melee combat

Add:

```text
action state (attack/block/hitstun/grapple)
attack event (attacker, target, move ID, tick)
damage/health/death
hit geometry source (host pose)
```

Still excludes weapons, projectiles, vehicles, animals, world travel, and campaign persistence.

## 15. Open questions

1. Which fields are mutated by `MonitorStatus`/`ManageTransformations` every tick and therefore need high-frequency sync? **Needs field-level instrumentation.**
2. Does camera zoom affect hit geometry or only screen coordinates? **Partially verified:** scale and range are affected by character scale; camera effects need tracing.
3. Which effects (`spurts`, `pools`) can influence gameplay? **Unclassified.**
4. How much off-screen character state must be synchronized for a campaign co-op mode? **Product-dependent.**
5. Can local saves be migrated to a server-authoritative profile without breaking editor semantics? **Open design question.**

## 16. Evidence index

| Subject | Lines / symbols |
|---|---|
| Player slot arrays | `1566..1906` |
| World/character arrays | `1416..1550` |
| `AddPlayer` | `20797` |
| `LoadPlayer` | `20996` |
| `ResetCharacter` | `18678` |
| `PlayerCycle` | `21348` |
| `GetInput` / `TranslateInput` | `8607` / `8729` |
| `ManageAnimations` | `23540` |
| `RenderFrame` | `14003` |
| `GameScreen` | `105575` |
| `LoadDeviceChars` | `19919` |
| `SaveDeviceChars` | `20516` |
| Global character record | `charLocation`, `charHealth`, `charRelationship`, `charTeam`, etc. |

See [ONLINE-MULTIPLAYER-ARCHITECTURE.md](ONLINE-MULTIPLAYER-ARCHITECTURE.md) for architecture choices and [ONLINE-MULTIPLAYER-PROTOTYPE.md](ONLINE-MULTIPLAYER-PROTOTYPE.md) for the staged prototype.
