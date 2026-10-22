# Main menu shell and visual foundation

## Current direction

Ancha House starts with its main menu and presentation layer, not a disposable playable prototype. The actual game scene will be built later, once the visual direction and project structure are agreed.

## Current menu shell

The root `index.html` now defines the initial layout:
- Dark fantasy title screen with the castle presentation area on the left and navigation on the right.
- Responsive desktop and mobile layout.
- Main menu entries reserved for Enter the House, Characters, Settings, and Credits & Sources.
- Modal placeholders so buttons have a defined interaction without pretending those systems already exist.
- A Three.js GLB loading stage configured to load `assets/castle_3d_model.glb`, frame the model automatically, light it, and rotate it slowly. Pointer dragging can rotate the model manually.
- A music toggle placeholder that does not claim to play audio before a track is selected.

## Important asset status

The supplied `castle_3d_model.glb` is not yet stored in this repository. The menu expects it at `assets/castle_3d_model.glb`. Until that exact asset is added there, the page displays a clear missing-model notice. Do not replace it with a fabricated castle or an unrelated model.

## Not implemented

- No movement prototype, game world, multiplayer, accounts, chat, or persistence.
- No real character selection, settings, audio playback, or game entry transition.
- No Stendhal assets copied into the repository.
- No final art or licensing decisions.

## Next work

1. Add the exact supplied GLB to `assets/castle_3d_model.glb` using a suitable large-file transfer method.
2. Inspect the model's scale, orientation, origin, materials, and visual quality, then tune the camera and lighting around the actual model.
3. Agree on the title-screen composition, type, palette, and menu behavior before building the world.
4. Investigate whether Stendhal contains an appropriate 3D castle model, and compare its actual license and suitability rather than assuming it does.
5. Review the witch-on-a-broom concept and source an approved sprite/illustration for the menu.
6. Once the menu is established, design the game scene and asset pipeline. Multiplayer comes later.

## Implementation note

Three.js and GLTFLoader are currently loaded from jsDelivr. The GLB is intended to remain a local project asset. For reliable local testing, serve the project over HTTP rather than opening the HTML as a `file://` URL.
