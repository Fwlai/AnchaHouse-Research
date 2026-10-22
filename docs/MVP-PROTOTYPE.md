# Ancha House: first playable prototype

## What exists now

The repository includes a self-contained browser prototype in `index.html`. It is deliberately dependency-free so it can be opened locally on a phone or desktop browser without installing a framework.

### Included
- A scrollable top-down fantasy castle courtyard with walls, towers, a central gate, paths, trees, and a small garden.
- A placeholder player character with four-direction movement.
- Keyboard controls: WASD or arrow keys.
- On-screen directional controls for touchscreens.
- Collision against the outer walls, castle building, and towers.
- A small interaction button / Space key with contextual placeholder messages.
- Responsive canvas rendering with pixelated presentation.

### Not included yet
- Multiplayer, accounts, server, persistence, chat, or Telegram integration.
- The final character sprites, licensed art, sound, or music.
- Interior rooms, floors, inventory, minigames, or a real door transition.
- The uploaded 3D castle model or the rotating 3D main menu.

## Run it

Open `index.html` in a modern browser. No build step or package installation is needed. On Android, download the repository files and open the HTML file in a browser, or serve the folder using any simple local HTTP server.

## Controls

| Input | Action |
| --- | --- |
| WASD / arrow keys | Move |
| Space | Interact |
| On-screen arrows | Move on touchscreens |
| INTERACT button | Interact |

## Technical direction

This is a first playable sketch, not the production architecture. Keep the game loop, world data, collision rules, and rendering easy to replace as the project grows. Before adding multiplayer, define a small authoritative server protocol and decide how player positions, identity, reconnects, and chat are validated. Do not treat client-sent positions as trusted.

## Next milestones

1. Test this prototype on Android and desktop; fix movement/collision issues.
2. Split rendering, map data, player input, and interactions into separate modules once the first scene is accepted.
3. Review the supplied 3D castle model and decide whether it belongs in the menu, while keeping the actual game world 2D.
4. Audit and select assets with verified license/attribution details.
5. Add interior transitions and a real map format.
6. Design multiplayer architecture and deployment before implementing networking.
