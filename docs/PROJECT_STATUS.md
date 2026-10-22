# Ancha House — Project Status

Last inspected: 2026-10-22
Repository: `Fwlai/AnchaHouse-Research`
Inspected commit: `0fede983127f1474db65fa2d7dd5f028228c7b98`
Branch: `main`

This document describes the repository as it exists at the inspected commit. It separates verified facts from reasonable inferences and unknowns. It is not a roadmap replacement; it records the current state.

---

## 1. Project overview

### Purpose and intended platform

Ancha House is described as an independent, small multiplayer 2D fantasy castle world for a small group of friends. The first presentable release target is Nowruz 1406 (around March 2027).

The only currently implemented platform is a static web page deployed through GitHub Pages. The current page is landscape-oriented and explicitly hides its menu and 3D stage in portrait orientation in favor of a rotate-device prompt.

Verified sources:
- `README.md`
- `docs/PROJECT-PLAN.md`
- `index.html`
- `.github/workflows/pages.yml`

### Intended core gameplay and user-facing features

The project intent documented in `docs/PROJECT-PLAN.md` includes:
- A 2D top-down fantasy world with four-direction character movement.
- A large central castle with towers, walls, walkways, an arched entrance, rooms, multiple floors, a courtyard, a garden, and hidden areas.
- Social interactions and small activities for friends.
- A small multiplayer experience.
- A main menu with a decorative rotating 3D castle, menu buttons, background music, and a pixel witch on a broom.

None of the actual game-world, movement, interaction, or multiplayer features are implemented in this repository. The repository currently contains a single-page main-menu presentation shell and supporting research/documentation.

### Intended architecture and technologies

Confirmed by repository contents:
- Static HTML/CSS/JavaScript site with no build step.
- Three.js `0.169.0`, loaded through an import map from jsDelivr, used for the menu scene.
- `GLTFLoader` from the Three.js addons package, used to load a local GLB castle model.
- GitHub Pages deployment through GitHub Actions.
- Landscape-first mobile/web presentation.
- Local GLB and PNG assets loaded over HTTP.

Proposed or not yet decided (not verifiable in the repository):
- The architecture of the eventual playable 2D world.
- The rendering technology for gameplay (2D canvas, WebGL, Three.js, or another approach).
- Character animation and sprite-loading architecture.
- Multiplayer networking, server authority, accounts, chat, and persistence.
- Audio implementation and music selection.
- Backend/hosting for anything beyond the static site.
- A future "interactive bot" and "Mini App" flow mentioned in `credits/ASSET-CREDITS.md`. The platform and implementation are unknown and the feature is not implemented.

---

## 2. Current implementation

### Implemented features and source locations

#### Static main-menu presentation

Source: `index.html`.

Implemented in the page:
- Full-screen dark fantasy title-screen shell with a fixed 3D stage and a right-aligned floating menu area.
- Five button elements. The first four are blank placeholders with only accessibility labels (`Menu slot 1` to `Menu slot 4`). The fifth is a fullscreen toggle with an accessibility label and title.
- Responsive layout rules for the menu and a portrait-orientation overlay reading "Rotate your device".
- An error banner used for model, texture, and fullscreen failures.

#### Three.js scene

Source: `index.html`.

Implemented:
- Scene, fog, perspective camera, WebGL renderer, hemisphere light, and two directional lights.
- A sky dome using a custom shader (`topColor`, `horizonColor`, `lowColor`).
- A star field made of small box meshes.
- A procedural low-poly island with an extruded sandy base, grass top, grass patches, voxel-style trees, shrubs, and scattered rocks.
- A large ocean plane with animated water glints.
- Voxel-style block clouds that drift slowly.
- A day/night cycle driven by a 180-second period (three minutes per full cycle), including sky-color interpolation, light-intensity changes, star fading, a sun sprite, and a moon sprite.
- A moon-phase drawing routine that updates the moon canvas texture.
- A fullscreen error message if the castle model fails to load.

#### Castle model integration

Source: `index.html`, asset `assets/castle_3d_model.glb`.

Implemented:
- The GLB is loaded from `./assets/castle_3d_model.glb`.
- The model is measured with `THREE.Box3`, re-centered horizontally, placed on the island surface, and scaled so its longest dimension is approximately `10.2` scene units.
- Shadows are disabled on the castle meshes.
- The castle is attached to a group that orbits the scene.

Verified asset facts:
- File size: `27,359,444` bytes (about 26 MB).
- Header magic: `glTF`; glTF version: `2`.
- License metadata and attribution are recorded in `credits/ASSET-CREDITS.md` as CC BY 4.0, creator JoshGarber.

#### Camera interaction

Source: `index.html`.

Implemented:
- Slow automatic orbit around the castle.
- Pointer-drag orbit control for mouse and touch through Pointer Events.
- Pitch clamping and damped easing toward target yaw/pitch.
- Orbit direction updates based on the horizontal drag direction.
- Releasing the pointer resumes automatic orbit.

#### Witches on broomsticks

Sources: `index.html`, `assets/sprites/witch-on-broomstick/PNG/48x64_scale2x/`.

Implemented:
- Two witch sprites use the supplied `witch-001-NESW.png` and `witch-002-NESW.png` sheets.
- Both sheets are `144x256` PNGs, i.e. a 3-column by 4-row frame layout.
- UV offset selection is implemented for a 3-frame walk cycle and four direction rows.
- Animation runs at approximately 0.15 seconds per frame.
- Witches orbit the castle, are attached to the castle group, and are visible only at night with opacity tied to the day/night blend.

Note: `README.md` and `credits/ASSET-CREDITS.md` still describe the witches as procedural placeholders or not wired into the menu. That is stale documentation; the current `index.html` loads the supplied sprite sheets.

#### Orientation handling

Source: `index.html`.

Implemented:
- A CSS media query for `orientation: portrait` shows a rotate-device overlay and hides the 3D stage and menu.
- A `#brand` selector is also hidden in portrait mode, but no `#brand` element exists in the document.

#### Fullscreen toggle

Source: `index.html`.

Implemented:
- The fifth menu button requests or exits fullscreen using the Fullscreen API where available.
- It updates its own accessibility label after fullscreen changes.
- It displays an error message when fullscreen is unavailable or fails.

### Partially implemented or non-functional features

- The main menu itself is not functional. The first four buttons have no click handlers, no visible labels, and no navigation. They are visual placeholders only.
- There are no "Enter the House", "Characters", "Settings", or "Credits & Sources" menu entries in the current page, despite older notes in `docs/MVP-PROTOTYPE.md` describing them.
- No modal, dialog, or transition system exists in the current page, despite older notes describing modal placeholders.
- Background music and sound are not implemented. No audio playback code exists in `index.html`.
- The daylight system is implemented, but the documentation conflicts with the code: `index.html` uses a 180-second cycle (three minutes), while `README.md` says the loop is about 12 minutes.
- The paving helper `tileFloor()` exists but is never called; a comment states "Castle paving disabled: keep the ground beneath the castle completely clear." The `paving` group therefore remains empty.
- The renderer is initially sized to roughly 62% of the viewport and is not followed by an initial `resize()` call. On first load, the canvas backing buffer may therefore be lower resolution than the CSS-displayed canvas until a resize or fullscreen event occurs. This is an inference from the code; browser rendering was not executed during this inspection.
- The `starField` loop contains a `star !== moonSprite` check even though the moon sprite is not a child of `starField`. It is harmless but redundant.
- The variable `starsVisible` is assigned but never used.
- `README.md` contains a literal `\n\n` sequence in the "Current prototype notes" heading area instead of real line breaks.
- The current menu has no visible title or brand text. The only visible UI text in landscape is the menu buttons themselves and error/prompt text.

### Planned but not implemented

All of the following are documented intent or research, not working features:
- The playable 2D world and castle interiors.
- Four-direction movement, camera following, collision, map loading, and depth sorting.
- Character selection and layered character appearance.
- Multiplayer, accounts, chat, social systems, and persistence.
- A game-entry transition from the menu.
- Settings and audio controls.
- A functional credits screen inside the site.
- Any Stendhal code or asset integration.
- Any final art selection or complete licensing review.
- Any music/soundtrack integration.
- The future bot/Mini App credits flow mentioned in `credits/ASSET-CREDITS.md`.

### Assets, resources, scripts, configuration, and documentation

#### Runtime assets

- `assets/castle_3d_model.glb` — local glTF 2.0 binary castle model used by the menu. About 26 MB.
- `assets/sprites/witch-on-broomstick/` — full Witch on Broomstick v1.2 asset pack:
  - `PNG/24x32/` — four sheets in NESW and SWEN variants.
  - `PNG/48x64_scale/` — four sheets in NESW and SWEN variants.
  - `PNG/48x64_scale2x/` — four sheets in NESW and SWEN variants; the two NESW sheets are used by `index.html`.
  - `README.txt`, `CHANGES.txt`, `sources.md`, and three license files (`LICENSE-CC-BY-3.0.txt`, `LICENSE-CC-BY-4.0.txt`, `LICENSE-OGA-BY-3.0.txt`).
  - GIMP source files under `source/` and preview images.

#### Research assets

- `assets-research/extralives/extralives.apk` — a third-party APK, about 55 MB.
- `assets-research/extralives/apk-extracted/` — extracted APK contents, about 75 MB, including:
  - `assets/Game.swf` — about 39 MB. Per task constraints, this file was not investigated or reverse-engineered.
  - `assets/music/theme01.mp3` — about 1.2 MB.
  - `assets/Characters.txt`, `assets/Frames.txt`, `assets/Game-app.xml`.
  - Android resources, native libraries, `classes.dex`, and AIR extension files.
  - `assets/META-INF/AIR/license.txt`.
- `assets-research/extralives/file-list.txt` — a generated file listing with roughly 900 entries.

The extracted `Game-app.xml` identifies the application as "Extra Lives", version `1.170.64`, copyright MDickie 2017, a "Desktop version of the zombie survival sim". No project documentation explains why this APK was added or what may be reused from it. Its licensing status is not recorded in `credits/ASSET-CREDITS.md`. Treat it as unverified research material, not approved project content.

#### Documentation and configuration

- `README.md` — high-level project description, current menu summary, deployment note, boundaries, and documentation links.
- `docs/PROJECT-PLAN.md` — intended game concept, visual direction, research priorities, and asset rule.
- `docs/MVP-PROTOTYPE.md` — menu-shell notes. Several statements are now out of date.
- `docs/STENDHAL-TECHNICAL-NOTES.md` — research leads for Stendhal sprite, camera, map, and tileset work. Nothing from Stendhal has been copied.
- `credits/ASSET-CREDITS.md` — asset attribution and license tracker.
- `.github/workflows/pages.yml` — validates required files and deploys the repository root to GitHub Pages on pushes to `main` and manual dispatch.
- `.github/workflows/validate-research.yml` — checks that required documentation exists and that project-boundary/asset-tracking notes are present.

No `package.json`, lockfile, build configuration, linter configuration, test framework configuration, `.gitignore`, or root `LICENSE` file exists.

---

## 3. Repository structure

```text
.
├── .github/workflows/
│   ├── pages.yml
│   └── validate-research.yml
├── assets/
│   ├── castle_3d_model.glb
│   └── sprites/witch-on-broomstick/
├── assets-research/
│   └── extralives/
│       ├── extralives.apk
│       ├── file-list.txt
│       └── apk-extracted/
├── credits/
│   └── ASSET-CREDITS.md
├── docs/
│   ├── MVP-PROTOTYPE.md
│   ├── PROJECT-PLAN.md
│   ├── PROJECT_STATUS.md
│   └── STENDHAL-TECHNICAL-NOTES.md
├── index.html
└── README.md
```

Important files:
- `index.html` — the entire current application: markup, CSS, Three.js scene, assets loading, animation loop, and interactions.
- `assets/castle_3d_model.glb` — the only 3D model used by the menu.
- `assets/sprites/witch-on-broomstick/` — licensed 2D sprite pack; two sheets currently used.
- `assets-research/extralives/` — third-party APK research material; not referenced by runtime code.
- `credits/ASSET-CREDITS.md` — authoritative in-repo asset/license tracker, though partly stale regarding witch integration.
- `docs/PROJECT-PLAN.md` — intended direction and boundaries.
- `.github/workflows/pages.yml` — the only functioning validation/deployment pipeline.

### Git state

- Current branch: `main`.
- HEAD: `0fede983127f1474db65fa2d7dd5f028228c7b98` (`Add Extra Lives APK and extracted resources`).
- Working tree before this report: clean (`git status --short` produced no output).
- Local `main` is in sync with `origin/main` at inspection time (`0` commits ahead, `0` commits behind).
- No uncommitted changes existed before this report was created.
- The only change made by this task is the addition of `docs/PROJECT_STATUS.md`.

### Uncommitted changes

Before writing this document: none.
After writing this document: `docs/PROJECT_STATUS.md` is a new untracked file. See the final `git status --short` result reported with this task.

No secret values, API keys, passwords, or tokens are reproduced in this document. A dedicated credential/secret scan was not part of this inspection.

---

## 4. Technical status

### Dependencies, frameworks, tools, and runtime requirements

Verified:
- Three.js `0.169.0` is loaded at runtime from `https://cdn.jsdelivr.net/npm/three@0.169.0/`.
- `GLTFLoader` is loaded from the matching Three.js addons path.
- The page uses browser ES modules and an import map.
- The page needs WebGL, Canvas 2D (for the moon/sun textures), Pointer Events, and `requestAnimationFrame`.
- The Fullscreen API is optional; failure is handled with an on-screen message.
- The site is static; GitHub Pages serves the repository root.
- Local development is documented as `python -m http.server 8080`. Opening the page through `file://` is not supported in practice because the GLB and PNG assets are fetched over HTTP.

Local inspection environment (not a repository requirement):
- Node.js `v20.19.2` available.
- Python `3.13.5` available.
- npm available but unused by the repository.

No package manager manifest exists, so there is no install step and no pinned local dependency tree. Runtime network access to jsDelivr is required. If the CDN is blocked or unreachable, the menu will not initialize.

### Build, run, and test procedures supported by the repository

Build:
- No build step. `index.html` is served as-is.

Run locally:
```bash
python -m http.server 8080
```
Then open `http://localhost:8080/` in a modern browser.

Deploy:
- Push to `main` or run the `Deploy Ancha House to GitHub Pages` workflow manually.
- GitHub Pages must be configured to use GitHub Actions as its source.

Tests:
- There is no unit/integration test suite.
- The repository has two CI validation workflows, not a test framework.
- `pages.yml` validates file presence, five slot classes, asset paths, the rotate-device prompt, sprite-sheet frame selection code, GLB magic/version, PNG signatures and 3x4 dimensions, and JavaScript syntax via `node --check`.
- `validate-research.yml` validates required documentation files and presence of project-boundary/asset-tracking text.

### Tests executed during this inspection

Executed locally, using the same commands as the workflows:

1. `pages.yml` validation logic (Python portion):
   - Result: passed.
   - `witch-001-NESW.png`: `OK` (`144x256`, 3x4 frames).
   - `witch-002-NESW.png`: `OK` (`144x256`, 3x4 frames).
   - Menu structure, GLB header, and sprite-sheet checks: `OK`.
   - GLB verified as glTF version `2`, size `27,359,444` bytes.
2. `pages.yml` JavaScript syntax check:
   - Extracted the module script to `/tmp/ancha-menu.mjs`.
   - Ran `node --check /tmp/ancha-menu.mjs`.
   - Result: passed (`OK: JavaScript syntax`).
3. `validate-research.yml` checks:
   - Required files existed and were non-empty.
   - `PixelTest` boundary text present.
   - License and source text present in `credits/ASSET-CREDITS.md`.
   - Result: passed.

Not executed or not verifiable here:
- The GitHub Actions workflows were not run on GitHub during this inspection; only their commands were reproduced locally.
- The page was not rendered in a real browser, so visual output, WebGL compatibility, CDN availability, touch behavior, and fullscreen behavior were not verified.
- No performance, memory, mobile-device, or cross-browser testing was performed.
- No multiplayer, backend, or gameplay tests exist.

### Known errors, limitations, compatibility concerns, and unresolved issues

- Main-menu buttons are non-functional placeholders; the menu cannot launch anything.
- The documentation is stale in several places:
  - `README.md` says the day/night cycle is about 12 minutes; `index.html` uses 180 seconds (three minutes).
  - `README.md` says the witch sprites are not wired into the menu; they are wired.
  - `README.md` says there are five blank slots; the fifth is a fullscreen toggle.
  - `docs/MVP-PROTOTYPE.md` says the GLB is missing; it is present.
  - `docs/MVP-PROTOTYPE.md` describes menu entries and modal placeholders that do not exist in `index.html`.
  - `credits/ASSET-CREDITS.md` says no supplied witch sheet is wired into the menu; the sheets are wired.
- The Three.js dependency is CDN-hosted and unpinned beyond the `0.169.0` URL. There is no local fallback.
- The initial renderer buffer size may be lower than the displayed canvas size until a resize/fullscreen event. This is inferred from `index.html`; not browser-verified.
- `#brand` is referenced in CSS but has no corresponding element.
- The unused paving code and unused `starsVisible` variable are minor dead-code issues.
- No `.gitignore` exists. Large binary research files and extracted APK contents are committed directly to Git, making the repository large (`assets-research` about 129 MB; `.git` about 132 MB).
- The Extra Lives APK and extracted resources are third-party commercial material with no recorded license or provenance decision. They create a redistribution/compliance risk if the public repository is treated as a distribution channel.
- No root license has been chosen for original project code.
- No accessibility audit has been performed. The blank buttons have accessible names, but the page has no visible menu labels, headings, or navigation semantics.
- The page's error banner is the only user-facing diagnostic for load failures.
- The intended game architecture, multiplayer design, and asset pipeline remain undecided.

---

## 5. Project progress

### Completed milestones

Verified as present in the repository:
- Repository and project documentation scaffolding.
- Project boundaries recorded: Ancha House is separate from PixelTest; PixelTest must remain untouched.
- Asset-credit tracking document with a structured template.
- Stendhal technical research notes (research only; no integration).
- GitHub Pages deployment workflow and documentation validation workflow.
- Supplied castle GLB added and credited with CC BY 4.0 attribution.
- Supplied Witch on Broomstick v1.2 pack added with license/source files.
- Main-menu 3D presentation built: procedural island, ocean, vegetation, rocks, clouds, sky, stars, lighting, fog.
- Castle model loading, auto-framing, scaling, and orbit.
- Pointer-drag camera orbit with damping.
- Three-minute day/night cycle with sun, moon, lunar phase, and night-only witches.
- Landscape-only presentation with portrait rotate prompt.
- Fullscreen toggle.
- Local validation of menu structure/assets and JavaScript syntax passed.

### Work in progress

- Main-menu information architecture and navigation. The presentation exists, but the menu is still a shell of unlabeled, mostly inert buttons.
- Visual tuning of the menu environment (camera, lighting, island composition, castle framing, day/night timing).
- Asset and license review. The castle and witch pack are documented; the Extra Lives research material is not.
- Documentation consistency. Existing docs predate several menu changes.

### Remaining tasks, ordered by priority and dependencies

1. **Define and implement the real main-menu navigation.** Add visible labels and working actions for the intended entries (for example Enter the House, Characters, Settings, Credits & Sources), or explicitly reduce the menu to the currently supported actions. This is the current stated focus and the dependency for any game-entry flow.
2. **Verify the menu in a real browser on desktop and mobile.** Confirm GLB loading, CDN availability, drag/orbit behavior, day/night visuals, fullscreen, portrait prompt, and performance. This should happen before further visual polish because it validates the only working feature.
3. **Reconcile documentation with implementation.** Correct the stale day/night duration, witch-integration status, slot description, and missing-model statements in `README.md`, `docs/MVP-PROTOTYPE.md`, and `credits/ASSET-CREDITS.md`.
4. **Resolve the Extra Lives research-asset policy.** Decide whether the APK and extracted contents belong in the public repository, record provenance/license status, or remove them. This is a legal/repository-hygiene prerequisite for any public release.
5. **Choose the game architecture and asset pipeline.** Decide the 2D world renderer, map format, sprite format, camera, collision, and data flow before writing gameplay code. This depends on the menu being stable and the asset/licensing direction being clear.
6. **Implement the first playable 2D scene and four-direction movement.** Depends on task 5 and on approved character/environment assets.
7. **Build castle interiors and world content.** Depends on the scene architecture and map pipeline.
8. **Add multiplayer, accounts, chat, social features, and persistence.** Depends on the world model, backend decisions, and security design.
9. **Add audio, settings, credits UI, and final art.** Some of these can proceed in parallel with tasks 5-7, but final audio licensing depends on asset review.
10. **Prepare the first presentable release.** Depends on all prior tasks plus full licensing review.

### Most logical next step and why

The most logical next step is to finalize the main-menu information architecture and make the existing menu buttons real: assign visible labels, implement the supported actions (at minimum Credits & Sources and the fullscreen/settings behavior), and either implement or clearly mark unavailable entries such as Enter the House.

Reason: the project explicitly states that the menu and presentation layer come first, and the only implemented application is the menu shell. The scene, castle, day/night cycle, and witches already exist, but the user cannot navigate anywhere. Making the menu coherent is the smallest high-value step that turns the current demo into a usable shell and unblocks the game-entry flow. It should be paired with browser verification and documentation corrections.

---

## 6. Evidence and confidence

### Verified facts

- Branch, HEAD, and clean working tree: `git status`, `git branch --show-current`, `git log`.
- Remote and sync state: `git remote -v`, `git rev-list --left-right --count origin/main...HEAD`.
- Menu structure and all runtime behavior described above: `index.html`.
- Castle asset presence, size, glTF version: direct file inspection.
- Witch asset presence, dimensions, frame layout, and licensing files: `assets/sprites/witch-on-broomstick/`.
- Research asset contents and application metadata: `assets-research/extralives/`.
- Deployment and validation behavior: `.github/workflows/pages.yml`, `.github/workflows/validate-research.yml`.
- Intended gameplay and scope: `README.md`, `docs/PROJECT-PLAN.md`.
- Asset attribution: `credits/ASSET-CREDITS.md`.
- Test results: locally executed equivalent of the CI validation commands, as listed in section 4.

### Reasonable inferences

- The initial renderer resolution may be lower than the displayed canvas because `renderer.setSize` is called with 62% dimensions and no initial `resize()` invocation follows. This follows from reading the code but was not visually confirmed.
- The current documentation set was written before the castle GLB and witch sprites were integrated, explaining several contradictions.
- `assets-research/extralives/` appears to be reference material for studying an existing game, but the repository does not state the intended purpose. No inference was made about the contents of `Game.swf`.
- Because no build/test configuration exists, the repository's effective quality gate is the two GitHub Actions workflows.

### Unknowns

- Whether the deployed GitHub Pages site currently works in a real browser.
- Whether the GitHub Actions workflows have passed on GitHub recently. Local reproduction passed, but no run history was inspected.
- The exact visual quality, performance, and compatibility of the menu on real devices.
- The intended use, provenance decision, and licensing status of the Extra Lives APK and extracted resources.
- The intended game engine, rendering approach, map format, multiplayer stack, backend, and hosting for gameplay.
- Whether the mentioned "interactive bot" / "Mini App" concept is Telegram-based or something else.
- The final asset list, final licensing decisions, and music selection.
- The meaning and intended use of `Characters.txt` and `Frames.txt` beyond being game-data files; they were not analyzed.
- Whether the project has any private planning, designs, or decisions not stored in this repository.

### Explicitly not done

- `Game.swf` was not opened, inspected, decompiled, or reverse-engineered. JPEXS/FFDec was not used.
- No new tools were installed.
- No MCP servers were configured.
- No source files, assets, configuration, or unrelated documentation were modified.
- No commit, push, publish, or upload was performed.
