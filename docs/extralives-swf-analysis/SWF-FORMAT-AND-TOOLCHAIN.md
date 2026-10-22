# SWF Format and Toolchain

This document records how the binary was validated, how the SWF was decompiled, and what the container actually contains. It separates container facts from decompiler output.

## Source integrity and the corrupt repository copy

The repository contains:

```text
assets-research/extralives/apk-extracted/assets/Game.swf
```

That copy is **not usable** as a complete SWF:

| Check | Result |
|---|---|
| SHA-256 | `92b351d136d8833af8d2f723fb1995c0ac463ffaa7c1dc1bfa912c4d6c47beba` |
| APK entry CRC for `assets/Game.swf` | expected `a18c9544`, actual `cd56cfec` |
| zlib decompression | fails at compressed offset `4,718,600` with `invalid stored block lengths` |
| Practical consequence | header and some early data are readable, but the code/asset payload is truncated or zeroed |

All other APK entries passed CRC. The corruption is localized to `assets/Game.swf`.

### Intact source used for analysis

The intact official APK was downloaded from APKPure for analysis and kept outside the repository:

```text
/tmp/extralives-swf-analysis/extralives_official.apk
/tmp/extralives-swf-analysis/official/assets/Game.swf
/tmp/extralives-swf-analysis/official/Game_full.swf
```

The official APK identifies itself as the same app/version (`air.ExtraLives`, `1.170.64`). It differs in hash from the repository APK because the repository copy is damaged, not because a different app was analyzed.

## Verified hashes

| Artifact | SHA-256 | Size |
|---|---|---:|
| Repository SWF (corrupt) | `92b351d136d8833af8d2f723fb1995c0ac463ffaa7c1dc1bfa912c4d6c47beba` | `40,567,300` bytes |
| Official APK (intact, `/tmp`) | `f71021000d35e275112f6a4e9b8c811578ba2fc2cac134d427c99d6a353d633b` | — |
| Extracted official SWF (`CWS`) | `17f2e23b31447125e2487009f464b5ba1dca863c31121ca58a380502ade8766b` | `40,567,300` bytes |
| Reconstructed decompressed SWF (`FWS`) | `c70b6a6a1e31fc2dbad3bfc7e4c8e8ae81662e538150366de0b4aa14325d6621` | `42,480,597` bytes |

The `CWS` file declares an uncompressed length of `42,480,597`; the zlib payload is `42,480,589` bytes. A full decompression succeeds and the reconstructed `FWS` is exactly the declared length.

## Toolchain actually used

| Tool | Version | Role |
|---|---|---|
| FFDec / JPEXS Decompiler | `22.0.2` | SWF inspection, script export |
| OpenJDK | `21.0.12` | FFDec runtime |
| Python | `3.13.5` | binary parsing, data schema checks |
| GNU coreutils / zip / sha256sum | Debian/Termux environment | file validation |
| Node.js | 20 (environment) | not needed for this analysis |

No `swftools`, `rabcdasm`, or custom AVM2 disassembler was used. FFDec was sufficient for container tags and AS3 source recovery.

FFDec entry point used:

```bash
cd /tmp/extralives-swf-analysis
FFDEC_MEMORY=3g ./ffdec.sh -onerror ignore -timeout 120 \
  -export script /tmp/extralives-swf-analysis/export_scripts \
  /tmp/extralives-swf-analysis/official/assets/Game.swf
```

FFDec printed a Jansi native-library warning on this ARM64/Termux-like environment, but the decompiler still ran and exported scripts. Treat that warning as a terminal-integration issue, not a decompilation failure.

## Container layout

### Header

| Field | Value | Notes |
|---|---|---|
| Signature | `CWS` | zlib-compressed SWF |
| Version | `51` (`0x33`) | SWF file format version, not “SWF 3” |
| Declared file length | `42,480,597` | uncompressed length |
| Stage rectangle | `[0, 14000, 0, 8000]` twips | logical `700 × 400` px (20 twips = 1 px) |
| Frame rate | `15.0` fps | runtime can change `stage.frameRate` |
| Frame count | `21` | one timeline frame per top-level scene |
| First tag offset | `0x15` | after 8-byte header + RECT + frame fields |

The stage is a fixed logical canvas. The app scales it to the device (`resX`, `resY`, `screenWidth`, `screenHeight`) rather than reauthoring the layout.

### Code

There is exactly **one top-level code tag**:

| Tag | Count | Size | Meaning |
|---|---:|---:|---|
| `DoABC2` (82) | 1 | `1,703,915` bytes | one AVM2 ABC bytecode block containing all classes |

`DoABC2` plus FFDec's successful AS3 export establishes **ActionScript 3 / AVM2**, not AS1/AS2. The SWF version `51` is the container version and should not be confused with the ActionScript version.

The `SymbolClass` tag maps 681 symbols to classes. `MainTimeline` is the document class via that mapping. FFDec exported **681 `.as` files** (~`4.79 MB`).

### Scenes

The `DefineSceneAndFrameLabelData` tag declares 21 scenes, in this order (timeline frame numbers are zero-based in the tag; the gameplay frame labels are the names):

| Frame index | Scene name |
|---:|---|
| 0 | `Loader` |
| 1 | `Framer` |
| 2 | `Titles` |
| 3 | `Options` |
| 4 | `Display` |
| 5 | `Purchase` |
| 6 | `Crossover` |
| 7 | `Controls` |
| 8 | `Calibration` |
| 9 | `Recipes` |
| 10 | `News` |
| 11 | `Hall Of Fame` |
| 12 | `Game` |
| 13 | `Editor` |
| 14 | `Edit Select` |
| 15 | `Match Setup` |
| 16 | `Save` |
| 17 | `Map` |
| 18 | `Time Machine` |
| 19 | `Commercial Break` |
| 20 | `Redirect` |

There are no additional named frame labels in the scene tag. The main class registers `addFrameScript(0..20)` in its constructor, so these frames are also code entry points.

### Top-level tag inventory

A direct parse of the reconstructed `FWS` yields **5,398 top-level tags**. FFDec's flattened dump (`official/tags.txt`) reports **46,670** tags because it also walks tags inside the 670 sprite definitions.

| Tag code | Name | Top-level count | Interpretation |
|---:|---|---:|---|
| 2 | `DefineShape` | 1,968 | vector shape definitions |
| 36 | `DefineBitsLossless2` | 1,475 | RGBA bitmap assets |
| 39 | `DefineSprite` | 670 | movie-clip/sprite symbols |
| 22 | `DefineShape2` | 312 | extended vector shapes |
| 37 | `DefineEditText` | 294 | editable/dynamic text fields |
| 14 | `DefineSound` | 227 | embedded sound effects |
| 74 | `CSMTextSettings` | 184 | text rendering settings |
| 35 | `DefineBitsJPEG3` | 81 | JPEG + alpha |
| 21 | `DefineBitsJPEG2` | 66 | JPEG |
| 6 | `DefineBits` | 42 | JPEG with shared tables |
| 1 | `ShowFrame` | 21 | one per scene frame |
| 11 | `DefineText` | 18 | static text |
| 75 | `DefineFont3` | 6 | embedded fonts |
| 73 | `DefineFontAlignZones` | 6 | font metrics |
| 88 | `DefineFontName` | 6 | font names |
| 46 | `DefineMorphShape` | 4 | morph shapes |
| 26 | `PlaceObject2` | 3 | top-level placements |
| 28 | `RemoveObject2` | 3 | removals |
| 20 | `DefineBitmap` | 2 | bitmap definitions |
| 82 | `DoABC2` | 1 | all AS3 code |
| 69 | `FileAttributes` | 1 | file flags |
| 9 | `SetBackgroundColor` | 1 | background |
| 64 | `EnableDebugger2` | 1 | debugger metadata |
| 86 | `DefineSceneAndFrameLabelData` | 1 | scene table |
| 45 | `SoundStreamHead2` | 1 | streaming sound header |
| 83 | `DefineShape4` | 1 | newer shape |
| 76 | `SymbolClass` | 1 | symbol → class map |
| 8 | `JPEGTables` | 1 | shared JPEG tables |
| 0 | `End` | 1 | end of file |

The asset counts explain the app's visual character: it is a large library of small vector shapes, alpha bitmaps, sprites, and short sounds, driven by one large code block. The game does not ship a general-purpose 3D engine or external scene files for maps; map geometry is represented through sprite/asset frames plus code-side coordinate tables.

## What decompilation recovered

- **Complete class set:** 681 scripts exported, matching the 681 `SymbolClass` mappings.
- **Main logic:** `Game_fla/MainTimeline.as`, 116,160 lines, 789 functions, 1,497 declared variables, 21 frame scripts, 474 `trace` calls.
- **Asset classes:** 444 `MovieClip` subclasses, 227 `Sound` subclasses, 5 `Font` subclasses, 4 `BitmapData` subclasses, and one plain enum-like class (`GameInputControlName`).
- **Data:** `Characters.txt` and `Frames.txt` remain external files in the APK assets; they are loaded at runtime rather than embedded in the SWF.

### Completeness caveats

- FFDec can recover readable AS3 for the classes it exported, but decompiler output is a reconstruction. It was **not** byte-for-byte recompiled or compared against the original ABC instruction stream.
- The export command used `-onerror ignore`, so isolated decompiler warnings could have been skipped. No complete per-file error audit was performed.
- Some code may be obscured by minified identifiers, dynamic property access, frame-script layout, or Flash timeline conventions. The main class is dynamic and uses many parallel arrays, which limits what static typing can prove.
- Asset semantics (which sprite is which map tile, animation, or prop) require correlating symbol names, frame numbers, and code tables; this analysis covers the major tables but not every art asset.

## Reproducibility checklist

```bash
# 1. Confirm the repository copy is damaged
sha256sum assets-research/extralives/apk-extracted/assets/Game.swf
unzip -t assets-research/extralives/extralives.apk | grep Game.swf

# 2. Use an intact official APK outside the repository
sha256sum /tmp/extralives-swf-analysis/extralives_official.apk

# 3. Extract and verify the SWF
unzip -p /tmp/extralives-swf-analysis/extralives_official.apk assets/Game.swf \
  > /tmp/extralives-swf-analysis/official/assets/Game.swf
sha256sum /tmp/extralives-swf-analysis/official/assets/Game.swf

# 4. Decompile scripts
cd /tmp/extralives-swf-analysis
FFDEC_MEMORY=3g ./ffdec.sh -onerror ignore -timeout 120 \
  -export script /tmp/extralives-swf-analysis/export_scripts \
  /tmp/extralives-swf-analysis/official/assets/Game.swf
```

**Do not commit** `/tmp` output, the official APK, the SWF, or the decompiled source tree. The repository stores only the analysis documents.
