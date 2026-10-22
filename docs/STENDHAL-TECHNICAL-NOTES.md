# Stendhal Technical Research Notes

Upstream: https://github.com/arianne/stendhal

These are investigation targets. Confirm implementation details against the current upstream source before adapting anything.

## Character sprites

Potential source directory: `data/sprites/outfit/`

Stendhal uses layered character appearance components such as body, head, hair, eyes, mouth, clothing, hats, masks, and details. The client combines appearance layers to display a character.

A human sprite layout of 48x64 frames with directional rows has been mentioned as a research lead, but do not assume it applies to every sprite. Inspect actual files and current code before implementing a loader.

## Character and animation research

Potential code and search targets:
- `src/games/stendhal/client/sprite/AnimatedSprite.java`
- Find the current location of `OutfitStore`.
- Search for outfit composition, direction mapping, and animation frame selection.

## Camera and rendering

Potential files to verify in the current repository:
- `src/games/stendhal/client/GameScreen.java`
- `src/games/stendhal/client/GameScreenSpriteHelper.java`
- `src/games/stendhal/client/EntityViewManager.java`

Research goals:
- World-to-screen coordinate transformation.
- Camera centering and following.
- Entity drawing order and depth sorting.
- Sprite alignment relative to map tiles.
- Animation timing and direction selection.

## Maps and castle references

Potential TMX maps to inspect:
- `data/maps/interiors/kalavan/castle.tmx`
- `data/maps/interiors/kalavan/castle_1.tmx`
- `data/maps/interiors/kalavan/castle_basement.tmx`
- `data/maps/interiors/semos/wizards_tower_1.tmx`
- `data/maps/interiors/semos/library.tmx`
- `data/maps/interiors/semos/tavern_0.tmx`

Potential tileset areas:
- `data/maps/tileset/ground/`
- `data/maps/tileset/building/`
- `data/maps/tileset/furniture/`
- `data/maps/tileset/plant/`
- `data/maps/tileset/terrain/`

## Next steps

1. Inspect the current upstream source tree.
2. Inspect sprite dimensions and sheet layout with an image tool.
3. Verify direction and animation mapping in source code.
4. Preview relevant castle and interior maps.
5. Select useful assets and document each file's source and license.
6. Design Ancha House's own map and art direction around the chosen assets.
