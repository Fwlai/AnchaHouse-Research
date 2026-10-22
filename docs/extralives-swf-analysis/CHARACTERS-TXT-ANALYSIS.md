# `Characters.txt` Analysis

**Scope:** canonical Extra Lives 1.170.64 character-definition table, loader, runtime identity, and cross-file dependencies.

**Data file:** `assets-research/extralives/apk-extracted/assets/Characters.txt`

**Evidence base:** intact official APK plus decompiled `Game_fla/MainTimeline.as` from the intact SWF.

## Confidence labels

- **Verified:** directly shown by the file contents or an explicit code path.
- **Strong inference:** follows from several verified facts but was not observed in a running game.
- **Open question:** needs runtime observation or more tooling.

## 1. What the file is

`Characters.txt` is the external **default character database**. It defines 200 characters, their names, appearance selections, body proportions, colors, stats, relationships, teams, and animation/weapon behavior IDs.

It is not a save file. Runtime mutations are stored separately in a Flash `SharedObject`; that save data can override fields after the text file is loaded.

Character identity is positional: character 1 is the first record, character 2 the second, and so on. The name is display text, not a stable key.

## 2. Physical format and record grammar

| Property | Value | Confidence |
|---|---|---|
| Encoding | ASCII | Verified |
| Size | 211,581 bytes | Verified |
| Line structure | One single line, no LF or CR | Verified |
| Separator | `\|` | Verified |
| First token | Declared record count `200` | Verified |
| Tokens from `split("\|")` | 65,402 | Verified |
| Trailing empty token | Yes | Verified |
| Character records | 200 | Verified |
| Fields per record | 327 | Verified |
| Names | 200 unique, non-empty, ASCII | Verified |
| Numeric types | Integers except RGB and some scale values | Verified |

The first token is a record count here, not a maximum index. The loader iterates `1..no_saved` and clamps `no_saved` to `no_chars` (200) if the file declares more.

### Complete field layout

Relative field indexes within one 327-field record:

| Offset | Count | Field |
|---:|---:|---|
| 0 | 1 | Character name |
| 1 | 1 | Gender: `0` male, `1` female |
| 2 | 1 | Team ID |
| 3 | 1 | Partner character ID |
| 4 | 1 | Lover character ID |
| 5 | 1 | Friend character ID |
| 6 | 1 | Enemy character ID |
| 7 | 6 | Six numeric stats |
| 13 | 143 | Costume 1 |
| 156 | 143 | Costume 2 |
| 299 | 9 | Nine move IDs |
| 308 | 4 | Ground moves: two IDs for each of two directions |
| 312 | 5 | Five attack IDs |
| 317 | 5 | Five crush/finisher IDs |
| 322 | 2 | Two stance IDs |
| 324 | 1 | Trigger ID |
| 325 | 1 | Flight ID |
| 326 | 1 | Preferred prop/weapon type ID |

Each costume block is:

```text
14 scale values              (limb 0, then limbs 1..13)
32 × [limb ID, R, G, B]      (limbs 1..32)
1 eye/demeanour value
```

That is `14 + 128 + 1 = 143` fields per costume.

The loader reads the scale and limb/RGB fields in an interleaved loop, not as 14 consecutive scales followed by all limb records. A parser must follow the loader's per-limb order.

## 3. Field semantics

### 3.1 Name, gender, team

| Field | Values | Meaning |
|---|---|---|
| Name | 3–16 ASCII characters | Display name; not a key |
| Gender | `0` / `1` | Male / female |
| Team | `0..8` | Faction/role ID |

`textTeam` maps team IDs as:

```text
0 Loner, 1 Master, 2 Native, 3 Survivor, 4 Civilian,
5 Protester, 6 Fighter, 7 Soldier, 8 Rogue
```

The canonical distribution is:

| Team | Characters |
|---:|---:|
| 0 Loner | 25 |
| 1 Master | 15 |
| 2 Native | 15 |
| 3 Survivor | 26 |
| 4 Civilian | 26 |
| 5 Protester | 15 |
| 6 Fighter | 26 |
| 7 Soldier | 26 |
| 8 Rogue | 26 |

Gender distribution: 156 male, 44 female.

Team affects AI/faction behavior and relationship generation. It is not merely a label.

### 3.2 Relationships

The four relationship fields are direct character IDs:

- `partner`
- `lover`
- `friend`
- `enemy`

Zero means no relationship. All canonical nonzero values are in `1..200`, and no character points at itself.

They are not a complete relationship matrix. They are the character's **primary** relation IDs. Runtime code expands them into a two-dimensional relationship array.

`AssumeRelationships` assigns:

| Relation | Relationship value | Runtime effect |
|---|---:|---|
| Partner | `21` | Strong positive partnership |
| Lover | `11` | Romantic/positive relationship |
| Friend | `1` | Friendly |
| Enemy | `-1` | Hostile |

It sets both directions for each listed relation. The order is partner, lover, friend, enemy, so if the same target appears in more than one category, the later category overwrites the earlier relationship value.

The canonical file has 20 records where one target appears in multiple categories (for example, a partner who is also listed as a friend, or a friend who is also listed as an enemy). This is structurally valid but can produce surprising runtime relationships because of the overwrite order.

Non-mutual relations in the file:

| Relation | Nonzero rows | Reciprocal rows |
|---|---:|---:|
| Partner | 26 | 22 |
| Lover | 27 | 24 |
| Friend | 140 | 72 |
| Enemy | 149 | 36 |

Many relationship targets are shared: for example, character 115 is the friend target of six characters and the enemy target of ten. `AssumeRelationships` makes each listed relationship bidirectional, but it does not turn the entire graph into a symmetric matrix when different characters point at different targets.

`GenerateRelationships` is called only under the `optRestore > 0` path. It starts from `AssumeRelationships` and then randomly adds same-team friendships and opposite-gender relationships under additional conditions. This is a source of runtime variation that a deterministic multiplayer design cannot ignore.

### 3.3 Stats

There are six stats. The first five have explicit labels in `textStat`:

| Stat index | Label | Observed range |
|---:|---|---:|
| 1 | Reputation | `0..90` |
| 2 | Strength | `55..125` |
| 3 | Toughness | `55..125` |
| 4 | Speed | `60..90` |
| 5 | Skill | `50..99` |
| 6 | *(no explicit text label found)* | `50..95` |

The sixth stat is loaded, copied into runtime `pStat`, saved, and can participate in stat contests. No `textStat[6]` label was found in the decompiled class. Treat its design name as unresolved.

Runtime behavior:

- `pStat[1..6]` is copied from `charStat` when a player is initialized.
- `ExhaustStats` scales stats 2–6 by a health-dependent factor.
- Toughness (`stat 3`) is heavily used for injury, collapse, and hunger resistance.
- Stats 2–5 are used in combat/dominance contests.
- `FindStrength` samples random indexes `1..10`; only indexes `1..6` exist, while rolls 9–10 compare body scale and can return special strength code `26`. This is code behavior, not file data.
- `CheckStatLimits` clamps stats to `50..99`, but it is not called for every character at load. Initial Strength/Toughness values above 99 therefore exist in the canonical data and remain until a later stat-changing path clamps them.

Stats are integer percentages, not raw damage or health values.

### 3.4 Costume data

Each character has two costumes.

#### Scale

`charScale[char][costume][limb]` values:

| Limb set | Range |
|---|---:|
| Limb 0 (global/body scale) | `74..112` |
| Limb 1 Hips | `72..130` |
| Limb 2 Body | `80..130` |
| Limb 3 Head | `95..125` |
| Limbs 4–7 legs | `72..130` |
| Limbs 8–13 arms/hands | `61..130` |

`charScale[char][2][0]` is overwritten at load with `charScale[char][1][0]`, so costume 2 uses costume 1's global scale.

The global scale affects rendering **and gameplay geometry**: `pScale` is used in hit ranges, push distances, water depth thresholds, body-drop checks, and several grapple positions.

#### Limb IDs

`charLimb[char][costume][limb]` selects a limb sprite variant. For body limbs 1–13, the canonical values are:

| Limb | Name | Observed IDs | Costume array max |
|---:|---|---:|---:|
| 1 | Hips | `1..22` | 22 |
| 2 | Body | `1..14` | 14 |
| 3 | Head | `1..116` | 123 |
| 4 | Right Thigh | `1..17` | 17 |
| 5 | Right Shin | `1..22` | 22 |
| 6 | Left Thigh | `1..17` | 17 |
| 7 | Left Shin | `1..22` | 22 |
| 8 | Right Shoulder | `1..16` | 16 |
| 9 | Right Forearm | `1..8` | 8 |
| 10 | Right Hand | `1..8` | 8 |
| 11 | Left Shoulder | `1..16` | 16 |
| 12 | Left Forearm | `1..8` | 8 |
| 13 | Left Hand | `1..8` | 8 |

These IDs index the `limbCostume[limb]` arrays initialized in `MainTimeline`. All canonical body-limb IDs are within their arrays. The loader would replace an out-of-range body ID with `1`.

Accessory/overlay IDs occupy limbs 14–32. The loader and editor use these categories:

| Limb | Category | Editor maximum |
|---:|---|---:|
| 14 | Hair | 89 |
| 15 | Hair extension | 21 |
| 16/17 | Sleeves | 21 |
| 18/19 | Wristbands | 23 |
| 20/21 | Body accessories | 57 |
| 22/23 | Shoes | 20 |
| 24/25 | Shorts | 23 |
| 26 | Hip pattern | 20 |
| 27 | Shirt | 44 |
| 28/29 | Gloves | 4 |
| 30/32 | Headwear | 78 |
| 31 | Beard | 36 |

All canonical accessory IDs fit these maxima except hair IDs `103` for characters 27 (“Sgt Acer”) and 94 (“Major Merc”), which exceed the editor's `no_hairstyles = 89`. The runtime still addresses hair frame `c` or `200 + c`, so this may be a special hair frame rather than an invalid ID. It is recorded as an anomaly/open question.

Headwear values above 78 would be reset to `0` by the loader; the canonical file has no such values.

#### RGB

`charR/G/B[char][costume][limb]` are per-limb color multipliers. They are applied through Flash `ColorTransform.redMultiplier`/`greenMultiplier`/`blueMultiplier`, not as 8-bit channel values.

Observed range: `0.05..2.52`. Values above `1.0` are valid and brighten the source art. Most limbs sit near `1.0`, but many entries are intentionally non-neutral.

#### Eyes/demeanour

`charEyes[char][costume]` ranges `0..3` and maps to:

```text
0 None, 1 Angry, 2 Neutral, 3 Docile
```

It drives eyebrow frames and facial demeanour, not combat logic.

### 3.5 Animation and behavior IDs

| Field | Count | Observed range | Meaning |
|---|---:|---:|---|
| `charMove` | 9 | `210..382` | Grapple/throw animation IDs |
| `charGroundMove[1]` | 2 | `412..437` | Ground move IDs for one direction set |
| `charGroundMove[2]` | 2 | `412..437` | Ground move IDs for the other direction set |
| `charAttack` | 5 | `1001..1319` | Strike animation IDs by attack category |
| `charCrush` | 5 | `1201..1312` | Crush/finisher animation IDs |
| `charStance` | 2 | `1..6` | Indexes into `libStance` |
| `charTrigger` | 1 | `0..20` | Special projectile/trigger style |
| `charFlight` | 1 | `0..4` | Flight/transformation style |
| `charProp` | 1 | `0..90` | Preferred weapon type ID |

These are not `Frames.txt` indices.

- `charMove` values are passed into animation/grapple logic (`charMove[char][randy]`) and are IDs in the `210..382` action space.
- Attack IDs are passed to `ChangeAnim`, for example `charAttack[char][1]` and `charAttack[char][3]`.
- Ground move IDs are used with `charGroundMove[char][bodyPart][randy]`.
- `charStance[1..2]` indexes `libStance = [0,1,2,133,134,135,141]`; stance IDs therefore map to animation IDs `1, 2, 133, 134, 135, 141`.
- `charTrigger` selects projectile behavior. Code tests values such as `10`, `11`, `18`, and `20`, and maps others through `GetProjectileCode`.
- `charFlight` maps to the names `None`, `Jump`, `Extended Arms`, `Missile`, `Wings`. Values 2–4 change the flight animation start frame (`1122`, `1126`, `1130`) and therefore the selected `Frames.txt` pose.
- `charProp` is a preferred weapon type. `AssignWeapon` uses it, and 0 means no preference.

If a move/attack/crush/ground ID is `0`, the loader replaces it with a random valid entry from the corresponding library. The canonical file has no zero in these fields, but modded or partial data could become nondeterministic at load.

## 4. Loader and lifecycle

`LoadDefaultChars` reads `Characters.txt` through either:

- `URLRequest("Characters.txt")` in the network/browser path, or
- `File.applicationDirectory.resolvePath("Characters.txt")` for local/desktop execution.

`LoadedDefaultChars` then:

1. Splits on `|`.
2. Reads and clamps the record count to `no_chars = 200`.
3. Reads names, gender, team, relations, and six stats.
4. Allocates all per-costume arrays.
5. Reads the interleaved scale/limb/RGB/eyes data.
6. Clamps body limb IDs to the corresponding costume array.
7. Clamps headwear IDs to `no_headwear` or zero.
8. Copies costume 1's global scale into costume 2.
9. Initializes wound visibility for each body limb.
10. Calls `CreateZombie(char, 2)`.
11. Reads move, ground move, attack, crush, stance, trigger, flight, and prop IDs.
12. Calls `ResetCharacter(char, 0)`.
13. Marks `loadedDefaultChars = 1`.
14. Calls `GenerateCharacter(0, 0, 0)`.
15. Optionally calls `GenerateRelationships` for every character.
16. Assigns weapons, resets team bases, and loads saved device data if present.
17. Starts `LoadFrames` if the frame table is not loaded.

This is a one-time load of the static database. Runtime character state is created later by `ResetCharacter` and game setup.

### Device/save overrides

`LoadDeviceChars` reads a Flash `SharedObject` named `Characters` (or `Time_Characters` in one mode). It can override:

- name, gender, team
- stats
- costume
- limb IDs, scales, RGB, eyes
- moves, attacks, crushes, stances
- trigger, flight, prop
- partner/lover/friend/enemy and the runtime relationship matrix

This is critical for multiplayer: two installations are not guaranteed to start from the same character state even when their `Characters.txt` files are identical.

### Save-back

`SaveDefaultChars` writes the current arrays back to `Characters.txt` in the same field order, including the two costume blocks and all behavior IDs. The in-game editor mutates `char[0]` as a working character and can save the database.

## 5. Runtime data flow

```text
Characters.txt
  -> LoadedDefaultChars
  -> charName / charGender / charTeam
  -> charStat[1..6]
  -> charPartner / charLover / charFriend / charEnemy
  -> charRelationship (via AssumeRelationships / GenerateRelationships)
  -> charLimb / charScale / charR / charG / charB / charEyes
  -> charMove / charGroundMove / charAttack / charCrush / charStance
  -> charTrigger / charFlight / charProp
  -> ResetCharacter
  -> pChar / pCostume / pScale / pStat / pRelationship / pMove ...
```

Then:

```text
charLimb + costume
  -> limbCostume[limb] array in the SWF
  -> base limb MovieClip class

Frames.txt frameF
  -> gotoAndStop within that MovieClip
```

The key interaction is:

- `Characters.txt` chooses **which artwork**.
- `Frames.txt` chooses **which pose subframe** and supplies offsets/rotation/scale/depth.
- `charScale` additionally scales the result and feeds combat geometry.
- Character action IDs eventually make `ManageAnimations` choose a `Frames.txt` index.

## 6. Gameplay relevance

### Combat and stats

- Toughness reduces collapse/injury risk and affects hunger damage.
- Strength/Speed/Skill participate in contests and AI evaluation.
- Stats are copied to per-player `pStat` and can be temporarily exhausted by health.
- Character scale affects hit range and body push.
- Relationship values feed targeting, team behavior, and narrative/promotion logic.
- Team IDs drive faction AI and hostility.

### Weapons and special abilities

- `charProp` drives preferred weapon assignment.
- `charTrigger` changes projectile/trigger behavior.
- `charFlight` changes flight/transformation animation states.
- `charMove`/`charAttack`/`charCrush`/`charGroundMove` select combat animations.
- `charStance` selects idle/combat stance animation IDs.

These fields make `Characters.txt` an authoritative gameplay data source, not merely a cosmetic roster.

## 7. Cross-file dependencies and identity

There is no explicit foreign-key header. Dependencies are implicit:

- Relationship IDs refer to records by ordinal ID `1..200`.
- `charLimb` IDs index `limbCostume` arrays embedded in the SWF.
- `charMove`, `charGroundMove`, `charAttack`, `charCrush`, and `charStance` select entries/animation IDs defined in code.
- `charProp` selects a weapon type defined in code.
- `charTrigger` selects projectile logic in code.
- `charFlight` selects flight animation frames in code.
- `charScale` and limb IDs affect `Frames.txt` rendering and derived hit geometry.
- `charEyes` selects facial frames.
- RGB values are color multipliers applied to the selected limb symbols.

Because character identity is the record ordinal, inserting, deleting, or reordering records would silently retarget relationship IDs and possibly save data. A multiplayer/content system should use an explicit stable character ID or a canonical content hash, not just the display name.

## 8. Data anomalies and quality

### Verified anomalies

- 127 scale values contain a decimal fraction. They occur only in costume 2 and affect 26 characters. The loader calls `int(...)`, so the fractional part is discarded under standard AS3 coercion. Example: `85.5` becomes `85`; `112.5` becomes `112`. **Strong inference** from code plus data; not runtime-tested.
- Stats 2 and 3 exceed the runtime clamp of 99 in the source data (up to 125). `CheckStatLimits` is not applied to every character at load, so those values remain until a later mutation path clamps them.
- Two characters use hair ID `103`, above the editor limit `no_hairstyles = 89`. Runtime may address special hair frames; design intent is unresolved.
- 20 characters list the same target in multiple relation categories. `AssumeRelationships` overwrites earlier categories with later ones.
- The sixth stat has no explicit `textStat` label.

### No structural failures

- 200 records and 200 unique names.
- No empty names.
- No relation ID outside `0..200`.
- No self-relations.
- No body-limb ID outside its costume array.
- No invalid headwear IDs.
- No missing fields.
- No non-ASCII or binary content.

## 9. Representative record

Character 1, “Whack Ax”:

```text
Gender: male
Team:   6 (Fighter)
Enemy:  102
Stats:  83, 75, 75, 80, 75, 60
```

Character 3, “Headache”:

```text
Gender: male
Team:   8 (Rogue)
Stats:  0, 80, 90, 70, 60, 90
```

Character 4, “Tsunami”:

```text
Gender: male
Team:   2 (Native)
Partner: 47
Friend:  47
Stats:   65, 125, 125, 60, 90, 70
```

The partner/friend overlap is one of the multi-category cases: `AssumeRelationships` processes partner first and friend later, so the final relationship value is friend (`1`) even though the partner ID remains `47`.

## 10. Implications for reconstruction and multiplayer

### Reconstruction

- Parse with the loader's interleaved costume order.
- Preserve record order; relationship IDs depend on it.
- Treat RGB as multipliers, not normalized colors.
- Preserve scale decimals if editing for fidelity, while accounting for the runtime `int(...)` truncation.
- Do not assume all stats are capped at 99 in the source data.
- Do not treat non-mutual relationships as corruption; expansion/generation is part of the game.

### Multiplayer

The static file is not enough to reproduce a player's character state. A multiplayer design needs to decide how to handle:

- per-installation `SharedObject` overrides
- edited or modded `Characters.txt`
- random relationship generation
- random replacement of zero action IDs
- character ordinal identity versus stable IDs

For an authoritative server or deterministic lockstep model, the safest baseline is:

1. Use the canonical `Characters.txt` hash as part of the content manifest.
2. Reject or explicitly negotiate modified character data.
3. Synchronize runtime character state that differs from the static file (costume, health, stats, relationships, weapons, injuries).
4. Never synchronize derived fields that can be recomputed from static content plus authoritative runtime state.

## 11. Open questions

1. What is the intended name and gameplay role of stat 6? No `textStat[6]` label was found.
2. Why do “Sgt Acer” and “Major Merc” use hair ID 103 when the editor limit is 89? **Open question; needs asset/runtime inspection.**
3. Are the scale fractions intentional precision that was meant to survive, or leftovers from a generator that expected integer output? The runtime truncates them, but intent is unknown.
4. Are multi-category relationship overlaps intentional characterization or generator artifacts?
5. Which combat formulas use stats 2–5 quantitatively beyond the observed contests and Toughness checks?
6. How does `charProp` interact with starting weapons in each game mode? The assignment path is verified, but full mode-specific behavior needs tracing.

## 12. Evidence index

Approximate line numbers in the decompiled `/tmp/extralives-swf-analysis/export_scripts/scripts/Game_fla/MainTimeline.as`:

| Subject | Lines / symbols |
|---|---|
| `LoadDefaultChars` | `19700` |
| `LoadedDefaultChars` | `19726` |
| Record-count clamp | `19731..19735` |
| Name/gender/team/relations | `19737..19755` |
| Six stats | `19756..19764` |
| Costume allocation | `19766..19779` |
| Interleaved costume parsing | `19780..19830` |
| Body limb clamp | `19801..19808` |
| Headwear clamp | `19809..19815` |
| Costume-2 global scale copy | `19815..19817` |
| Wound initialization / zombie creation | `19818..19830` |
| Move/ground/attack/crush parsing | `19831..19880` |
| Stance/trigger/flight/prop parsing | `19881..19889` |
| Random replacement of zero IDs | `19832..19880` |
| Start of `LoadFrames` | `19910..19918` |
| `SaveDefaultChars` | `19638..19698` |
| `AssumeRelationships` | `19462` |
| `GenerateRelationships` | `19490` |
| `charStat` runtime copy | `21222..21225` |
| `ExhaustStats` | `22115..22122` |
| Toughness checks | `21691`, `21767`, `21801`, `21832` |
| `FindStrength` | `20695..20720` |
| `ChangeStat` and stat display | `22580..22610` |
| `textStat` labels | `110089` |
| `textTeam` labels | `110092` |
| `textGender` / pronouns | `110090..110097` |
| `textDemeanour` | `110094` |
| `textFlight` | `110110` |
| `libStance` | `111072` |
| `libMove` | `111354` |
| `libGroundMove` / `libAttack` / `libCrush` | `111532..111575` |
| Limb costume arrays | `110610..110633` |
| Accessory maxima | `110633..110644` |
| `RenderAccessory` | `16462` |
| `ChangeColour` | `3769` |
| `LoadDeviceChars` | `19930..20570` |
| `SaveDeviceChars` device output | `20516..20595` |
