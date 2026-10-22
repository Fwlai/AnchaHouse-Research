# Ancha House

An independent small multiplayer 2D fantasy castle world for friends. Development starts with the title-screen shell and the visual identity, then moves into the actual world and game systems.

## Current focus: main menu

Open the deployed GitHub Pages site (see below) or serve the folder over HTTP (`python -m http.server 8080`) and open `index.html`. The landscape-only menu loads `assets/castle_3d_model.glb` over a procedural pixel-style island with grass, stone paving, sandy shoreline, and subtly animated block-water tiles. Drag with mouse or touch to rotate the castle; release eases the view toward the front. The menu includes five blank floating slots, a slow day/night color cycle, block-built clouds, and two night-only pixel witch placeholders orbiting with the castle. The supplied Witch on Broomstick sprite sheets are documented in the asset credits but are not yet wired into the current menu build.

## Deployment

Every push to `main` runs `.github/workflows/pages.yml` and publishes the repository to GitHub Pages. One-time setup: repository Settings, Pages, Build and deployment, Source: GitHub Actions.

## Current prototype notes\n\n- Portrait mode hides the menu and displays a rotate-device prompt.\n- The day/night loop is intentionally slow (about 12 minutes per full cycle).\n- The island, water, paving, and clouds are prototype geometry made in Three.js, not final art.\n- The castle is menu decoration only. The eventual playable world remains a separate 2D depth-style game; this menu does not implement gameplay.\n- The witch flight paths are attached to the castle group so they keep their relative position when the castle rotates. The current witch visuals are procedural placeholders; integration of the supplied licensed sprite sheets remains a follow-up.\n\n## Project boundaries

- Ancha House is separate from PixelTest.
- PixelTest must remain untouched.
- PixelTest may be used only as a reference for movement, camera behavior, and rendering ideas.
- Third-party assets must not be treated as approved until their exact source, creator, license, attribution, and redistribution obligations are verified.
- Do not build throwaway movement prototypes before the menu and structure are established.

## Documentation

- [Project plan](docs/PROJECT-PLAN.md)
- [Menu shell notes](docs/MVP-PROTOTYPE.md)
- [Stendhal technical research](docs/STENDHAL-TECHNICAL-NOTES.md)
- [Asset credits and license tracker](credits/ASSET-CREDITS.md)

## Target

First presentable release target: Nowruz 1406 (March 2027).
