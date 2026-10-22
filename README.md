# Ancha House

An independent small multiplayer 2D fantasy castle world for friends. Development starts with the title-screen shell and the visual identity, then moves into the actual world and game systems.

## Current focus: main menu

Open the deployed GitHub Pages site (see below) or serve the folder over HTTP (`python -m http.server 8080`) and open `index.html`. The menu shows the castle GLB (`assets/castle_3d_model.glb`) on a pixel-tile ground, turning slowly (drag to rotate), with two pixel witches on brooms flying around it, and the menu buttons on the right. Music is still a placeholder.

## Deployment

Every push to `main` runs `.github/workflows/pages.yml` and publishes the repository to GitHub Pages. One-time setup: repository Settings, Pages, Build and deployment, Source: GitHub Actions.

## Project boundaries

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
