# Assets and Data Formats

## External files shipped in the APK

The APK assets directory contains only a small number of top-level files; most visual and audio content is embedded in the SWF.

| File | Size | Role |
|---|---:|---|
| `assets/Game.swf` | 40,567,300 bytes | code + embedded shapes/bitmaps/sounds |
| `assets/Characters.txt` | 211,581 bytes | 200 character records |
| `assets/Frames.txt` | 320,860 bytes | 1,401 animation frame records |
| `assets/Game-app.xml` | 3,581 bytes | AIR descriptor, version, permissions, extensions |
| `assets/music/theme01.mp3` | 1,206,159 bytes | external music track |

The repository's `Game.swf` is corrupt; use the intact official copy described in [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md). `Characters.txt` and `Frames.txt` in the repository are readable and were used to verify the schemas below.

## SWF asset inventory

Top-level tags (not counting tags nested inside sprites):

| Asset type | Count |
|---|---:|
| `DefineShape` + `DefineShape2` + `DefineShape4` | 1,968 + 312 + 1 |
| `DefineBitsLossless2` (RGBA bitmaps) | 1,475 |
| `DefineSprite` | 670 |
| `DefineSound` | 227 |
| `DefineEditText` | 294 |
| `DefineText` | 18 |
| JPEG tags (`DefineBits`, `DefineBitsJPEG2`, `DefineBitsJPEG3`) | 42 + 66 + 81 |
| `DefineFont3` | 6 |
| `DefineMorphShape` | 4 |
| `DefineBitmap` | 2 |

The complete top-level tag table is in [SWF-FORMAT-AND-TOOLCHAIN.md](SWF-FORMAT-AND-TOOLCHAIN.md#top-level-tag-inventory). The flattened FFDec tag dump contains 46,670 entries because it descends into the 670 sprites.

### Fonts

Six embedded font definitions were exported as five font classes plus associated font-name tags:

- `ComicFont`
- `DigitalFont`
- `GrungeFont`
- `PrintFont`
- `TimesFont`

### Bitmap data classes

Four classes extend `BitmapData`:

- `IntroBG`
- `MenuBG`
- `PaperBG`
- `ShadowImage`

### Sounds

227 `DefineSound` tags correspond to 227 `Sound` subclasses in the exported code. These are mostly short effects; the main music track is the external `music/theme01.mp3` rather than a 227th music class.

## `Characters.txt`

### Container

- Single physical line, pipe-delimited (`|`).
- First token: record count (`200`).
- Then 200 character records.
- Trailing delimiter produces a final empty token.
- Total split tokens: `65,402` = `1` count + `200 × 327` fields + `1` trailing empty.

### Record layout (327 fields)

The field order below is taken directly from `LoadedDefaultChars` (line 19723). “coz 1/2” are the two costume/body variants; coz 0 is initialized from defaults and has no file tokens.

| Order | Count | Field | Notes |
|---:|---:|---|---|
| 1 | 1 | name | string |
| 2 | 1 | gender | int |
| 3 | 1 | team | int |
| 4 | 1 | partner | character ID |
| 5 | 1 | lover | character ID |
| 6 | 1 | friend | character ID |
| 7 | 1 | enemy | character ID |
| 8 | 6 | stats | six integers |
| 9 | 14 | scale, coz 1 | limbs `0..13` |
| 10 | 128 | limb + RGB, coz 1 | for limbs `1..32`: limb ID, R, G, B |
| 11 | 1 | eyes, coz 1 | int |
| 12 | 14 | scale, coz 2 | limbs `0..13` |
| 13 | 128 | limb + RGB, coz 2 | for limbs `1..32`: limb ID, R, G, B |
| 14 | 1 | eyes, coz 2 | int |
| 15 | 9 | moves | nine move IDs |
| 16 | 2 | ground move 1 | two IDs |
| 17 | 2 | ground move 2 | two IDs |
| 18 | 5 | attacks | five attack IDs |
| 19 | 5 | crushes | five crush/grapple IDs |
| 20 | 2 | stances | two stance IDs |
| 21 | 1 | trigger | int |
| 22 | 1 | flight | int |
| 23 | 1 | prop | int |

Total: `7 + 6 + 143 + 143 + 9 + 4 + 5 + 5 + 2 + 1 + 1 + 1 = 327`.

The “limb + RGB” block uses 32 limbs per costume because `no_limbs (13) + no_accessories (19) = 32`; the loop includes limb 0 but skips its limb/RGB values. Each costume also stores a scale for limbs `0..13`, including limb 0. This is why the expected count is 327, not the smaller number one might estimate from “13 limbs” alone.

### First record

The file begins with character ID 1:

```text
200|Whack Ax|0|6|0|0|0|102|83|75|75|80|75|...
```

`Whack Ax` is a valid named character; the leading `200` is the record count, not part of the name.

### Validity checks performed

- Declared count: 200.
- Token count: 65,402.
- Derived record size: 327.
- `65,402 - 2 = 65,400 = 200 × 327`.
- Code field order was matched against the actual token count.

**Verified:** the schema and record count are consistent with the file and the loader code.

## `Frames.txt`

### Container

- Single physical line, pipe-delimited.
- First token: `1400`.
- The loader loops `cyc = 0 .. no_saved` **inclusive**, so the file contains **1,401 frame records** (frame 0 plus frames 1–1400).
- Each frame has 13 limbs × 7 numeric values.
- Total data tokens: `1 + 1401 × 13 × 7 = 1 + 127,491 = 127,492`; plus the trailing empty token = `127,493` split tokens.

### Per-frame layout

For each frame index, and for each limb `1..13`, the loader reads exactly:

| Order | Field | Type | Code identifier |
|---:|---|---|---|
| 1 | X | Number | `frameX[frame][limb]` |
| 2 | Y | Number | `frameY[frame][limb]` |
| 3 | Z | Number | `frameZ[frame][limb]` |
| 4 | A | Number | `frameA[frame][limb]` |
| 5 | S | Number | `frameS[frame][limb]` |
| 6 | D | int | `frameD[frame][limb]` |
| 7 | F | int | `frameF[frame][limb]` |

These are pose/limb parameters consumed by `GetFrame`, `UpdateCostume`, and `ManageAnimations`. The names are the code's storage names; this document does not assign a stronger geometric meaning to `A/S/D/F` than the code supports.

### Why 1,401 frames

`no_frames` is initialized to 1400 (line 110591), but both `LoadFrames` (line 13896) and `SaveFrames` (line 13872) use `<=` over `0..no_saved` / `0..no_frames`. The actual file therefore has one more record than the nominal frame count. If `no_saved < no_frames`, the loader fills missing frames with zero/default poses.

### Animation system tie-in

- `pAnim` identifies a high-level animation state.
- `ManageAnimations` (line 23540) advances that state and selects frame/limb poses.
- `GetFrame` (line 14572) chooses upper/lower body frames for limbs.
- `UpdateCostume` (line 16171) applies costume scale, limb IDs, RGB, and accessories.
- The animation state machine is much larger than the frame file: 1,400 frames are pose data, not 1,400 independent behaviors.

## `SharedObject` save formats

The game uses AIR/Flash local shared objects. These are key-value stores, not server saves.

| SharedObject local name | Purpose | Code |
|---|---|---|
| `System` | game version, boot count, PPV/hack flags, time/date logs | `LoadSystem` (3139) |
| `Options` | options, controls, multiplayer setting | `LoadOptions` / `SaveOptions` (3261 / 3362) |
| `Calibration` | input calibration | calibration screen |
| `Progress` | main save slot | `LoadProgress` / `SaveProgress` (3389 / 3529) |
| `Time_Progress` | Time Machine slot | same functions with `screenAgenda = 101` |
| `Characters` | main character records | `LoadDeviceChars` / `SaveDeviceChars` (19919 / 20516) |
| `Time_Characters` | Time Machine character records | same functions with agenda 101 |

`LoadDeviceData` (line 3108) loads the Time Machine slot first, then the main slot, and also restores animal/item/weapon/tunnel stocks. `SaveDeviceChars` serializes character fields back into a pipe-delimited string and saves it to the local object. This is why the game can be backed up by copying app data, and why the save data is client-owned and untrusted for any future online mode.

## AIR metadata and extensions

From `Game-app.xml`:

| Field | Value |
|---|---|
| AIR namespace | `http://ns.adobe.com/air/application/51.1` |
| App ID | `ExtraLives` |
| Version | `1.170.64` |
| Content | `Game.swf` |
| Render mode | `gpu` |
| Aspect ratio | `landscape` |
| Fullscreen | `true` |
| Auto-orient | `true` |
| Architecture | `64` |

Declared extensions:

```text
com.distriqt.InAppBilling
com.distriqt.Core
com.distriqt.playservices.Base
com.google.android.play
```

The code imports `com.distriqt.extension.inappbilling.*` and implements product loading, purchase handling, and upgrade messaging. This is a billing integration, not a gameplay network layer.

Android manifest additions request network, Wi-Fi/network state, phone state, wake lock, external storage, billing, and cross-user permissions. These support ads, billing, and device integration; they do not imply a multiplayer protocol.

## What was not extracted

- Not every bitmap/sprite was exported to individual image files; the analysis used the tag inventory and AS class mappings.
- No attempt was made to reconstruct the full map art pipeline or to label every sprite symbol.
- No runtime asset cache or decompiled source tree is committed. All analysis output lives under `/tmp/extralives-swf-analysis/`.

## Practical implications

- **Modding/data:** character and animation data are simple text files and can be parsed independently of the SWF. The schemas above are sufficient to build a validator or converter.
- **Save compatibility:** save slots are local objects with code-defined fields; changing character schema or frame counts risks breaking old saves.
- **Networking:** any protocol must serialize the *logical* state represented by these arrays, not the SWF assets or `MovieClip` objects. The text schemas are a useful reference for compact entity state, but the live game state is much broader.
