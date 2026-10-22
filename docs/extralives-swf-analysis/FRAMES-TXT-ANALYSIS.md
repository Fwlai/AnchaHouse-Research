# `Frames.txt` Analysis

**Scope:** canonical Extra Lives 1.170.64 animation/pose table, its loader, runtime data flow, and gameplay coupling.

**Data file:** `assets-research/extralives/apk-extracted/assets/Frames.txt`

**Evidence base:** intact official APK plus decompiled `Game_fla/MainTimeline.as` from the intact SWF. The repository `Game.swf` is corrupt; the data files are not.

## Confidence labels

- **Verified:** directly shown by the file contents or an explicit code path.
- **Strong inference:** follows from several verified facts and the runtime data, but was not observed in a running game.
- **Open question:** needs runtime observation or a different tool.

## 1. What the file actually is

`Frames.txt` is an external, positional **limb-pose keyframe table**. It is not a list of whole-body spritesheet frames and it does not contain named animations.

Each numeric frame record stores seven presentation parameters for each of 13 body limbs:

```text
frame index -> limb 1..13 -> [X, Y, Z, A, S, D, F]
```

Animation IDs select a frame index at runtime. The selected record is then combined with character-specific limb sprites, character scale, runtime offsets, and easing. The result is both the visible pose and, indirectly, part of the geometry used by combat collision checks.

The file supplies pose parameters; it does not supply the limb artwork itself. Artwork comes from the SWF symbols selected through `Characters.txt` and `limbCostume`.

## 2. Physical format and record grammar

| Property | Value | Confidence |
|---|---|---|
| Encoding | ASCII | Verified |
| Size | 320,860 bytes | Verified |
| Line structure | One single line, no LF or CR | Verified |
| Separator | `\|` | Verified |
| First token | Declared maximum frame index `1400` | Verified |
| Token count from `split("\|")` | 127,493 | Verified |
| Trailing empty token | Yes, from the final separator | Verified |
| Frame records | 1,401, indexes `0..1400` inclusive | Verified |
| Fields per limb | 7 | Verified |
| Limbs per frame | 13 | Verified |
| Values per frame | 91 | Verified |
| Numeric data tokens | 127,491 | Verified |
| Number types | All integer text; no decimal values | Verified |

The count token is a **maximum index**, not a record count. `FramesLoaded` loops inclusively:

```text
for (cyc = 0; cyc <= no_saved; ++cyc)
```

Therefore `1400` declares indexes `0` through `1400`, or 1,401 records. The runtime trace string says “Loaded 1400 frames of animation”, which can be misleading if read as a count.

### Field order

For every frame and limb:

```text
X | Y | Z | A | S | D | F
```

The file has no headers, frame names, comments, hashes, or identifiers. Meaning is entirely positional and comes from the loader and consumers.

## 3. Field reference

| Field | Observed range | Loader type | Verified role |
|---|---:|---|---|
| `X` | `-41..19` | `Number` | Limb horizontal offset in game units before camera/character scaling. |
| `Y` | `-48..153` | `Number` | Limb vertical offset; hips also receive `pOffsetY`. |
| `Z` | `1..13` | `Number` | Depth rank added to `pZ * 1000 + pLimbZ`; controls limb display ordering. |
| `A` | `-500..470` | `Number` | Limb rotation in degrees. Every value is a multiple of 5. |
| `S` | `83..110` | `Number` | Limb scale in percent. |
| `D` | `-1` or `1` | `int` | Limb direction/flip. No zero occurs in the canonical file. |
| `F` | `1..12` | `int` | Limb subframe selector, not a global frame index. |

### `X` and `Y`

`SharpTransition` and the smooth-render path copy these values into runtime `pLimbX` and `pLimbY`. For limb 1, `Y` gets the active `pOffsetY` added. The root limb position also incorporates character position, camera, bounce, breathing, and character scale.

For child limbs, the final position is calculated from the parent's runtime position plus a bind angle, bind distance, character scale, parent limb scale, and camera zoom. The `X`/`Y` fields are therefore local pose offsets, not world coordinates.

### `Z`

`RenderFrame` computes a display depth:

```text
logZ = pFilterZ[cyc] * 1000 + pLimbZ[cyc][limb] + frameZ[f][limb]
```

`DepthSorting` then sorts sprites numerically by `displayZ`. The field is primarily a depth-layer number, but it is also read in a few conditional checks involving baggy clothing and limb ordering.

In 1,388 of 1,401 frames, the 13 `Z` values are a permutation of `1..13`. Thirteen frames have two limbs at the same rank. See section 8.

### `A`

`A` is direct limb rotation in degrees. It is added to parent rotation and dynamic offsets (`pLimbOffsetA`) and applied to the limb MovieClip. The editor changes it in five-degree steps and labels it “Angle”. Every value in the file is a multiple of 5.

Limbs also use bind-angle tables for limbs 1 and 2, indexed by the parent limb's `F` value. This is a separate rotation contribution, not another meaning for `A`.

### `S`

`S` is a percentage scale. It multiplies limb dimensions and contributes to child offset calculations. Values below 100 are common on hands, shins, and similar limbs.

### `D`

`D` is a direction sign and flipping flag:

- `frameD[f][limb] < 0` negates the limb's effective angle.
- The sign chooses positive or negative `scaleX`.
- When `D` is negative on a parent limb, the child bind angle is also negated.
- Weapon direction uses `frameD` directly in `weapDir`.

The canonical file never stores `0`; the runtime default for limb 11 is `-1`, and all other default limbs use `1`.

### `F`

`F` has two verified roles:

1. **Sprite subframe selector.** `RenderFrame` does `limbSprite.gotoAndStop(frameF[f][limb])`. For hands and gloves it is also combined with weapon-hold and glove-frame mappings.
2. **Bind-table selector for limbs 1 and 2.** When a child limb's parent is limb 1 or 2:

   ```text
   angle = limbBindAngle[limb][frameF[f][parent]]
   distance = limbBindDistance[limb][frameF[f][parent]]
   ```

   Thus the body and hips subframe control attachment of child limbs, not only the parent's artwork.

The valid range is limb-specific:

| Limb | Name | Valid observed `F` |
|---:|---|---|
| 1 | Hips | 1–3 |
| 2 | Body | 1–3 |
| 3 | Head | 1–2 |
| 4 | Right Thigh | 1–2 |
| 5 | Right Shin | 1–2 |
| 6 | Left Thigh | 1–2 |
| 7 | Left Shin | 1–2 |
| 8 | Right Shoulder | 1–2 |
| 9 | Right Forearm | 1 |
| 10 | Right Hand | 1–12 |
| 11 | Left Shoulder | 1–2 |
| 12 | Left Forearm | 1 |
| 13 | Left Hand | 1–12 |

The editor enforces the same maxima. `F` is not an index into `Frames.txt`.

## 4. Loader and initialization lifecycle

`LoadFrames()` is defined in `MainTimeline.as`:

- In browser/network mode it uses `URLRequest("Frames.txt")`.
- Otherwise it resolves `Frames.txt` through `File.applicationDirectory`.
- Completion handler: `FramesLoaded`.
- It is called after default character initialization.
- `loadedFrames` guards repeated initialization.
- The loading-screen fallback calls it only when `loadedFrames == 0`.
- A successful load sets `loadedFrames = 1`.

`FramesLoaded` allocates seven arrays and reads exactly 91 values per frame, for 13 limbs.

If the declared maximum is below the built-in `no_frames` constant, the loader fills the missing indexes with defaults:

- `X/Y = 0`
- `Z = limbZ[limb]`
- `A = 0`
- `S = 100`, except limbs 6, 7, 11, 12, 13 use `95`
- `D = 1`, except limb 11 uses `-1`
- `F = 1`, except limb 3 uses `2`, limb 10 uses `8`, limb 11 uses `2`, and limb 13 uses `10`

The canonical file supplies all 1,401 records, so this fallback is not used at startup.

`SaveFrames()` serializes the arrays back into the same pipe-delimited format. It is called by the in-game frame editor at `MainTimeline.as:104104`. `CopyFrame` copies selected limbs or a whole frame, including all seven fields.

### Initialization versus gameplay

- **Initialization:** read once, allocate seven arrays, populate indexes `0..1400`, set `loadedFrames`.
- **Per-gameplay update:** animation code sets `pAnimFrame`; `RenderFrame` reads the selected record.
- **Per-frame presentation:** `RenderFrame` converts seven static fields into runtime `pLimbX/Y/Z/A/S`, clip rotation, scale, and subframe.
- **Editing/debug:** the frame editor mutates the same arrays and can save the external text file.

## 5. Runtime consumer path

### Primary path

```text
Frames.txt
  -> FramesLoaded
  -> frameX / frameY / frameZ / frameA / frameS / frameD / frameF
  -> GetFrame / RenderFrame
  -> pLimbX / pLimbY / pLimbZ / pLimbA / pLimbS / pLimbOldFrame
  -> limb MovieClip transform and gotoAndStop
  -> DepthSorting and screen rendering
```

### `GetFrame`

`GetFrame(cyc, limb)` begins with `pAnimFrame[cyc]`, then applies:

- `pLowerFrame[cyc]` for limbs 1, 4, 5, 6, 7
- `pUpperFrame[cyc]` for other limbs
- `pLimbFrame[cyc][limb]` if non-zero

This gives animation code a way to split a character into independent upper/lower or per-limb overrides. The returned value is an index into `Frames.txt`.

### `RenderFrame`

`RenderFrame` is the main consumer and is called once per active player per simulation/render cycle from `PlayerCycle`, as well as by editor, preview, and UI paths.

For each of 13 limbs it:

1. Chooses the frame with `GetFrame` logic.
2. Reads `frameA/X/Y/S` through `SharpTransition` or `SmoothTransition`.
3. Reads parent bind angle/distance, selecting a bind-table entry with `frameF` when the parent is limb 1 or 2.
4. Applies `frameD` sign changes to angles and `scaleX`.
5. Positions the limb relative to parent or root.
6. Reads `frameF` as the limb clip subframe and calls `gotoAndStop`.
7. Updates accessories, wounds, gloves, hair, hats, and other overlays using the same subframe.
8. Adds `frameZ` to the display depth and logs the sprite for `DepthSorting`.

`RenderFrame` therefore does more than draw: it establishes the runtime limb geometry consumed by later gameplay queries.

### Accessory and wound paths

`frameF` is also passed into:

- `UpdateAccessoryFrame`
- `UpdateHeadAccessoryFrame`
- `UpdateWoundFrame`

These functions map the limb subframe to a frame inside an accessory or wound MovieClip. A frame value therefore selects not only the base limb pose but also compatible overlays.

### Animation path

`ManageAnimations` sets `pAnimFrame` for movement, attacks, stances, reactions, grapples, and special states. A direct scan of the decompiled class found 1,844 literal `pAnimFrame[...] = <number>` assignments:

- minimum `0`
- maximum `1346`
- 1,027 distinct values

No literal assignment above `1346` was found. Frames `1347..1400` are therefore default filler in normal data paths. They are still valid array indexes because the loader reads them.

`pAnimFrame` is not simply an animation counter that increments by one for every animation. The large animation state machine maps timers and states to specific frame indexes, including non-contiguous poses, upper/lower splits, and per-limb overrides.

## 6. Relationship to gameplay

`Frames.txt` is often described as visual data, but that is incomplete. Three verified coupling paths matter.

### 6.1 Hit geometry

Combat calls such as `FindImpacts` use `pLimb[cyc][3].y` as the head/impact height and other limb positions as attack origins. Those runtime limb coordinates come from `frameX`, `frameY`, parent bind transforms, character scale, and camera transform.

Changing pose data can therefore change where a strike lands, even if the attack ID, range, and damage values stay the same. This is a presentation-derived gameplay input.

### 6.2 Weapon direction

`frameD` is read directly when establishing weapon direction, for example:

```text
weapDir[cyc] = frameD[pLimbLastAnimFrame[v][limb]][limb] * pDir[v]
```

It also affects hand subframe remapping inside `RenderFrame` when a held weapon changes the effective hand frame. Direction is thus both a visual flip and a gameplay-facing orientation.

### 6.3 Animation/AI gates

`frameF` values are inspected by gameplay code, not only by rendering:

- Limb 3 subframe `1` versus `2` is used in head/eyes logic.
- Limb 2 subframe `3` changes depth treatment for certain body accessories.
- Body subframes 1/2/3 select different behavior in special animation blocks.
- `frameF[f][3] > 1` is used in a head/face check.
- A generated/default animation condition tests `frameF[anim][2] > 1`.

These are small but direct logical dependencies. The frame table is part of the executable data model, not an optional cosmetic layer.

### 6.4 What is still presentation-derived

The following are primarily visual or derived:

- limb sprite subframe selection (`F`)
- depth ordering (`Z`)
- absolute limb position/rotation (`X/Y/A/S`)
- breathing, easing, camera, bounce, and character costume overlays

They become gameplay-relevant only where collision, weapon orientation, or animation gates consume the resulting runtime state.

## 7. Relationship to `Characters.txt` and SWF assets

There is no key or foreign-key join between `Frames.txt` and `Characters.txt`. They are independent positional tables.

The runtime relationship is:

```text
Characters.txt
  -> charLimb[char][costume][limb]
  -> limbCostume[limb] array
  -> a limb MovieClip class (Hips, Body, Head, ...)

Frames.txt
  -> frameF[f][limb]
  -> gotoAndStop(frameF) inside that selected MovieClip
```

In other words:

- `Characters.txt` chooses **which limb artwork** a character uses.
- `Frames.txt` chooses **which pose subframe** within that artwork is shown.
- Frames also provide the limb offsets, angle, scale, and depth that position the artwork.
- `charScale` from `Characters.txt` multiplies frame positions/scales and therefore also influences effective hit geometry.

A character's `charMove`, `charAttack`, `charCrush`, and related values are animation/action IDs. They are not `Frames.txt` indexes. They eventually drive `ManageAnimations`, which selects frame indexes.

## 8. Data quality, anomalies, and repetition

### 8.1 Duplicate frames

There are 1,328 unique 91-value records among 1,401 frames. Twenty duplicate groups account for 93 records.

| Duplicate count | Frame indexes |
|---:|---|
| 55 | `0`, `1347..1400` |
| 2 | `1`, `687` |
| 2 | `8`, `102` |
| 2 | `49`, `408` |
| 2 | `184`, `241` |
| 2 | `218`, `220` |
| 2 | `366`, `369` |
| 2 | `412`, `415` |
| 2 | `429`, `432` |
| 2 | `504`, `507` |
| 2 | `520`, `523` |
| 2 | `532`, `536` |
| 2 | `594`, `598` |
| 2 | `693`, `696` |
| 2 | `698`, `699` |
| 2 | `701`, `708` |
| 2 | `871`, `875` |
| 2 | `880`, `883` |
| 2 | `993`, `996` |
| 2 | `1113`, `1237` |

The 55-record group is different from the small pairs: every record there is identical to frame 0, and no normal literal animation assignment selects an index above 1346. The small duplicate pairs may be intentional reused poses, copied editor frames, or authoring accidents; the data alone cannot distinguish those cases.

### 8.2 Duplicate `Z` ranks

Thirteen frames are not permutations of `1..13` because limbs 3 (Head) and 7 (Left Shin) both use `Z = 5`:

```text
57, 69, 85, 172, 836, 1032, 1033, 1034, 1052, 1302, 1303, 1310, 1311
```

Most of these share the exact sequence:

```text
[9, 4, 5, 8, 7, 6, 5, 12, 11, 13, 2, 1, 3]
```

Frame 836 differs only in the two later ranks (`11, 12` rather than `12, 11`).

A depth tie is not automatically an error. `Array.sortOn` can impose a stable or arbitrary order for equal depths, and runtime limb offsets can break ties. Without a visual capture of these exact poses, treat this as an anomaly to verify, not a confirmed bug.

### 8.3 No invalid field values

- No `Z` outside `1..13`.
- No `D` outside `-1`/`1`.
- No `F` outside its limb-specific bound.
- No negative or fractional `S`.
- No missing values and no partial records.
- No non-ASCII or binary content.

The file is structurally well-formed for the loader.

## 9. Representative records

Frame 0 is the default pose. Its 13 limb tuples are:

```text
Limb 1  Hips:           [ 0, 0, 10,   0, 100,  1,  1]
Limb 2  Body:           [ 0, 0,  4,   0, 100,  1,  1]
Limb 3  Head:           [ 0, 0,  5,   0, 100,  1,  2]
Limb 4  Right Thigh:    [ 0, 0,  9,   0, 100,  1,  1]
Limb 5  Right Shin:     [ 0, 0,  8,   0, 100,  1,  1]
Limb 6  Left Thigh:     [ 0, 0,  7,   0,  95,  1,  1]
Limb 7  Left Shin:      [ 0, 0,  6,   0,  95,  1,  1]
Limb 8  Right Shoulder: [ 0, 0, 12,   0, 100,  1,  1]
Limb 9  Right Forearm:  [ 0, 0, 11,   0, 100,  1,  1]
Limb 10 Right Hand:     [ 0, 0, 13,   0, 100,  1,  8]
Limb 11 Left Shoulder:  [ 0, 0,  2,   0,  95, -1,  2]
Limb 12 Left Forearm:   [ 0, 0,  1,   0,  95,  1,  1]
Limb 13 Left Hand:      [ 0, 0,  3,   0,  95,  1, 10]
```

Frame 1 differs mainly by small rotations on the arms and a large left-hand rotation (`-360`), consistent with a breathing/idle or transition pose.

Frame 57 is one of the `Z`-tie frames: limb 1 is raised (`Y = 63`, `A = 85`) and limb 3 head angle is `20`, while the Head/Left Shin share depth rank 5.

Frame 121 contains a more asymmetric pose, including a right-thigh `X = -3`, a right-shin angle of `125`, and a left-hand rotation of `-360`.

## 10. Practical implications

### For reconstruction or modding

- Preserve the exact positional order; there is no schema or checksum.
- Treat the first token as a maximum index, not a count.
- `F` must respect limb-specific maxima or the selected limb clip may not contain the target frame.
- `D` should stay `±1`; changing it flips both art and some logical direction decisions.
- `Z` should normally remain a permutation unless a deliberate depth tie is understood.
- Editing poses can alter combat geometry and weapon orientation, not only appearance.
- Frames above 1346 are safe defaults today, but custom animation code may choose to use them.

### For deterministic simulation

The frame table is static, deterministic input. Nondeterminism comes from runtime selection and transitions (timers, random animation choices, RNG, and input timing), not from reading the file. A network or replay system can treat the table as a fixed shared asset and reference frames by index.

### For a multiplayer state model

Do not synchronize all seven static fields per limb. They are shared immutable content. Synchronize the runtime selection and overrides instead:

- character ID
- costume
- animation ID
- current frame index
- upper/lower/per-limb frame overrides
- transition/easing state if remote interpolation must match exactly

The authoritative combat simulation still needs frame-derived limb geometry, but that can be regenerated from the same content and runtime pose state.

## 11. Open questions

1. Are the 13 duplicate-`Z` frames intentional depth ties or authoring errors? **Open question; needs visual/runtime observation.**
2. Are the 19 non-default duplicate frame pairs intentional reuse or accidental copies? **Open question.**
3. Are frames `1347..1400` reserved for future content, editor scratch space, or simply filler? No normal literal assignment uses them, but the reason is not documented.
4. Does any rare dynamic animation path construct a frame index above 1346 without a literal assignment? The direct scan says no normal path does; a full instrumented run would settle it.
5. How much do exact frame-derived hit heights change combat outcomes in practice? Code proves the dependency; quantitative impact requires simulation or runtime tracing.
6. Are `frameF` body/head special cases gameplay rules or merely visual layering conditions? The code contains direct branch tests, but their intended design meaning is not documented.

## 12. Evidence index

Approximate line numbers in the decompiled `/tmp/extralives-swf-analysis/export_scripts/scripts/Game_fla/MainTimeline.as`:

| Subject | Lines / symbols |
|---|---|
| Frame array declarations | `1320..1332` |
| `CopyFrame` | `13823` |
| `SaveFrames` | `13872` |
| `LoadFrames` | `13895` |
| `FramesLoaded` | `13913` |
| Loader call after default characters | `19913..19915` |
| Loading-screen fallback | `103708` |
| `RenderFrame` | `14003` |
| Field-to-runtime transition | `14161..14200` |
| Limb transform | `14202..14266` |
| `frameF` subframe and weapon mapping | `14268..14320` |
| Accessory and overlay updates | `14320..14430` |
| Depth calculation | `14435..14465` |
| `GetFrame` | `14572` |
| `SharpTransition` | `14584` |
| `SmoothTransition` | `14600` |
| `UpdateAccessoryFrame` | `14469` |
| `UpdateHeadAccessoryFrame` | `14488` |
| `UpdateWoundFrame` | `14509` |
| `PlayerCycle` render call | `21559` |
| Frame editor | `103760..104125` |
| Editor field labels | `104108..104111` |
| Weapon direction from `frameD` | `75001`, `75029` |
| Impact geometry from limb positions | `53459` and callers around `50137..50457` |
| `FindImpacts` head/impact use | `53459`, `53642` |

Data measurements in this document were produced by parsing the canonical file directly, not by sampling or by trusting the declared count.
