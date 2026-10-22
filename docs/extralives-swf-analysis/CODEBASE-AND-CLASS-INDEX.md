# Codebase and Class Index

## Shape of the codebase

The SWF contains one ABC block and 681 exported ActionScript 3 classes. The distribution is extremely top-heavy:

| Group | Count | Notes |
|---|---:|---|
| Classes in package `Game_fla` | 1 | `Game_fla.MainTimeline` |
| Classes in the default package | 680 | asset/symbol classes and one utility class |
| Total scripts | 681 | matches the 681 `SymbolClass` mappings |
| Total exported source | ~4.79 MB | not committed |
| `MainTimeline` source | 4,543,866 bytes / 116,160 lines | nearly all behavior |
| `MainTimeline` functions | 789 | including 21 frame scripts |
| `MainTimeline` variables | 1,497 | large parallel-array state model |

### Class base types

| Base | Count | Role |
|---|---:|---|
| `MovieClip` | 444 | visual symbols, maps, effects, UI, animal/character clips |
| `Sound` | 227 | one class per embedded sound symbol |
| `Font` | 5 | `ComicFont`, `DigitalFont`, `GrungeFont`, `PrintFont`, `TimesFont` |
| `BitmapData` | 4 | `IntroBG`, `MenuBG`, `PaperBG`, `ShadowImage` |
| none | 1 | `GameInputControlName` |

`Game_fla.MainTimeline` is declared as:

```as3
public dynamic class MainTimeline extends MovieClip
```

The `dynamic` keyword matters: many properties are created or addressed dynamically, and the class uses large arrays instead of typed model objects.

## `MainTimeline` as the application

`MainTimeline` is both the document class and the game's service locator. It owns:

- the 21-frame scene timeline and all screen handlers;
- options, input, controller/gamepad state, and menus;
- character, animation, world, item, weapon, animal, and recipe tables;
- the active match: player entities, AI, combat, physics-ish movement, camera, lighting, particles, and depth sorting;
- local persistence (`SharedObject`) and external data loading;
- ads, billing, and store/social navigation.

There is no separate `Game`, `World`, `Player`, `Entity`, or `Network` class in the exported set. Entities are represented by parallel arrays on `MainTimeline` plus dynamically attached `MovieClip` sprites.

### Initialization

1. The constructor registers `addFrameScript(0..20)`.
2. Frame 1 (`frame1()`, line 109489) initializes tables, loads options/progress, attaches the loader background and load frame, then waits for the loader screen.
3. `LoaderScreen` moves to the next screen; later frames attach the appropriate screen handler.
4. Frame 21 (`frame21()`, line 116051) routes screen IDs to scene labels.

This is a timeline-driven application, not a conventional `DocumentClass` with a separate scene graph controller.

## Key method index

Line numbers refer to the decompiled `/tmp/.../Game_fla/MainTimeline.as` used during analysis. They are useful for navigating a regenerated export, but can shift if a different FFDec version is used.

### Lifecycle and screens

| Method | Line | Role |
|---|---:|---|
| `frame1()` | 109489 | global initialization, data loads, loader setup |
| `frame2()` … `frame20()` | 114941–116034 | scene setup and `ENTER_FRAME` handler attachment |
| `frame21()` | 116051 | screen router / redirect handling |
| `LoaderScreen` | 103698 | initial load progress |
| `FramerScreen` | 103750 | frame/size setup |
| `TitleScreen` | 104184 | title/menu |
| `OptionsScreen` | 104528 | options |
| `DisplayScreen` | 104605 | display/video options |
| `PurchaseScreen` | 104731 | billing/upgrade UI |
| `CrossoverScreen` | 104830 | cross-promotion |
| `ControlsScreen` | 104971 | controls |
| `CalibrationScreen` | 105050 | input calibration |
| `RecipesScreen` | 105085 | crafting recipes |
| `NewsScreen` | 105156 | in-game news |
| `HiScoreScreen` | 105486 | hall of fame |
| `GameScreen` | 105575 | main gameplay loop |
| `EditScreen` | 106160 | character editor |
| `EditSelectScreen` | 108220 | editor entry |
| `MatchSetupScreen` | 108707 | match configuration |
| `SaveScreen` | 109182 | save/load UI |
| `MapScreen` | 109305 | map |
| `TimeScreen` | 109349 | time machine |
| `CommercialScreen` | 109429 | commercial break |

Each screen frame adds an `ENTER_FRAME` listener; the listeners are visible around lines 114938–115986. The screen system is therefore a set of per-frame handlers switched by timeline frames and `screen` IDs.

### Main loop and world

| Method | Line | Role |
|---|---:|---|
| `GameScreen` | 105575 | per-frame orchestration |
| `ManageTime` | 100814 | game clock, speed-up, frame rate |
| `ManageCamera` | 58951 | follow/zoom/multi-player framing |
| `ManageLighting` | 58762 | day/night/lighting state |
| `ManageCells` | 61285 | jail/cell logic when cells exist |
| `ManageVehicles` | 61030 | vehicles |
| `ManageAtmos` | 5071 | ambient/atmosphere effects |
| `AnimalCycle` | 79463 | animals |
| `AddPlayer` | 20797 | create/attach a player entity |
| `LoadPlayer` | 20996 | configure a player slot |
| `PlayerCycle` | 21348 | per-player update |
| `ItemCycle` | 68702 | items/props |
| `WeaponCycle` | 74118 | weapons |
| `TunnelCycle` | 66884 | tunnels/connections |
| `BulletCycle` | 84257 | bullets |
| `ExplosionCycle` | 81833 | explosions |
| `SpurtCycle` | 82478 | blood/spurt particles |
| `PoolCycle` | 82840 | blood/water pools |
| `ManagePromos` | 85617 | story/promo director |
| `DepthSorting` | 14549 | painter's-order rendering |

The observed order in `GameScreen` is:

```text
PauseProcess
ManageMusic
ResetText
ManageTime
ManageCamera
ManageLighting (if enabled)
CageCycle (if cages)
ManageCells (if cells)
UpdateTimeLogs (if Time Machine area)
ManageVehicles
ManageAtmos
AnimalCycle (if animals)
FindGestures
population spawning via AddPlayer
PlayerCycle
ItemCycle
WeaponCycle
TunnelCycle
BulletCycle
ExplosionCycle
SpurtCycle
PoolCycle
health/clock UI updates
ManagePromos
DepthSorting
```

This confirms a single-threaded, frame-driven simulation. There is no fixed-timestep loop or separate physics thread.

### Characters, animation, AI, and combat

| Method | Line | Role |
|---|---:|---|
| `LoadDefaultChars` | 19700 | load `Characters.txt` |
| `LoadedDefaultChars` | 19723 | parse 200 character records |
| `GenerateCharacter` | 18783 | procedural character creation |
| `GenerateRelationships` | 19490 | social graph initialization |
| `CreateZombie` | 18189 | zombie conversion |
| `ResetCharacter` | 18678 | restore/reset character |
| `AI` | 11622 | per-entity AI/agenda/focus/attack decisions |
| `ManageAnimations` | 23540 | animation state machine, ~34.7k lines |
| `ChangeAnim` | see function index | switch animation state |
| `GetFrame` | 14572 | select frame from animation tables |
| `UpdateCostume` | 16171 | limb/costume rendering |
| `RiskHeadDamage` | 22311 | head injury |
| `RiskInjury` | 22349 | body injury |
| `RiskLimbLoss` | 17114 | limb loss |
| `RiskResurrection` | 23507 | resurrection |
| `ImpactChecks` | 54354 | collision/impact checks |
| `AttackAnims` | 49418 | attack animation mapping |
| `AttackViable` | 32089 | attack availability |
| `AttackSensible` | 13155 | target/attack choice |
| `RiskReversal` / `RiskCounter` | 44835 / 44859 | reversals and counters |

`ManageAnimations` is by far the largest method. It spans roughly lines 23,540–58,286 and contains the state machine for movement, stance, climbing, grappling, attacks, injuries, vehicles, and special animations. The animation IDs index the 1,400 stored frames, not 1,400 distinct high-level states.

### Input and local multiplayer

| Method | Line | Role |
|---|---:|---|
| `TouchBegin` | 8169 | touch input |
| `TouchMove` | 8218 | touch drag |
| `TouchRelease` | 8236 | touch release |
| `FindGestures` | 8253 | gesture recognition |
| `KeyPress` | 8412 | keyboard |
| `KeyRelease` | 8429 | keyboard release |
| `GetInput` | 8607 | per-entity control logic |
| `TouchInput` | 8725 | touch-to-control mapping |
| `TranslateInput` | 8729 | input translation |
| `ControlNewPlayer` | 10119 | assign a controller to a player |
| `ControllerBrand` | 10177 | gamepad brand/type |
| `ControllerActivated` | 10298 | gamepad connected |
| `ControllerRemoved` | 10403 | gamepad removed |
| `UpdateP1Gamepad` … `UpdateP4Gamepad` | 10412–10433 | per-port gamepad updates |

The control state is local. `pControl`, `pController`, and `pGamepad` are arrays on `MainTimeline`, not network identities.

### Persistence and data

| Method | Line | Role |
|---|---:|---|
| `LoadSystem` | 3139 | `SharedObject("System")` |
| `SaveOptions` / `LoadOptions` | 3362 / 3261 | `SharedObject("Options")` |
| `LoadProgress` / `SaveProgress` | 3389 / 3529 | `Progress` and `Time_Progress` |
| `LoadDeviceData` | 3108 | loads Time Machine and main slots |
| `LoadDeviceChars` | 19919 | character slot from `SharedObject` |
| `SaveDeviceChars` | 20516 | character slot to `SharedObject` |
| `LoadFrames` / `SaveFrames` | 13896 / 13872 | external `Frames.txt` |
| `LoadDefaultChars` | 19700 | external `Characters.txt` |
| `LoadWorld` | 58333 | build map/background and coordinate space |

## Symbol-class inventory

The 680 default-package classes are almost all generated wrappers for library symbols. They fall into recognizable families:

| Family / examples | Approx. count | Notes |
|---|---:|---|
| Character/animal/creature clips (`BearClip`, `WolfClip`, `TigerClip`, `HorseClip`, `DeerClip`, `RabbitClip`, etc.) | dozens | animal rendering/behavior support |
| Map/environment clips (`IslandMap`, `CityMap`, `UpperCells`, `BasketballNet`, etc.) | dozens | attached to map areas and props |
| Effect/UI clips | dozens | explosions, blood, menus, text, indicators |
| `Sound` subclasses | 227 | one per embedded sound |
| Font classes | 5 | embedded typefaces |
| Bitmap classes | 4 | loader/menu/paper/shadow |

`GameInputControlName` is the only non-`MovieClip`/`Sound`/`Font`/`BitmapData` class. Its name and 25 KB size suggest an enum/name table for input controls rather than game state.

The main class's own size is the important architectural fact: 4.54 MB of the 4.79 MB exported source is one file. Any refactor or network layer would either extend this class or require extracting state and simulation responsibilities from it.

## Structural observations

- **Verified:** all gameplay state is addressed through public arrays on `MainTimeline` (e.g. `pChar`, `pX`, `pY`, `pGround`, `charName`, `mapName`).
- **Verified:** entity visuals are `MovieClip`s registered with `LogEntity` and sorted by `displayZ`.
- **Verified:** screen transitions and menus are driven by the same timeline class.
- **Strong inference:** the class is effectively a hand-rolled entity-component system flattened into arrays and methods; there are no formal model classes.
- **Unknown:** whether any dead or unused code paths remain from earlier MDickie engine versions. The presence of stubs such as `TriggerStorylines` shows at least some legacy/unfinished areas.
