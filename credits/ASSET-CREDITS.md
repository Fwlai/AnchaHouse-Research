# Ancha House - Asset Sources, Credits and Obligations

This document tracks third-party assets, source code, music, and other resources used or considered for Ancha House. Nothing listed as unreviewed should be treated as approved for redistribution until its exact source and license have been verified.

## Tracking template

For every selected asset, record:

- Asset name:
- Local path:
- Original source URL:
- Creator(s):
- License stated by the original source:
- Required attribution text:
- Modifications made:
- Redistribution requirements:
- Source-file or license-document path:
- Verification status: Unverified / Needs review / Verified / Rejected
- Intended use in Ancha House:
- Notes:

## In use

### Castle 3D Model
- Local path: `assets/castle_3d_model.glb`
- Original source URL: https://sketchfab.com/3d-models/castle-3d-model-23eb94cac33d4099b7185a81bdc038e4
- Creator: JoshGarber (https://sketchfab.com/JoshGarber)
- License stated by the original source and inside the file metadata: Creative Commons Attribution 4.0 (CC BY 4.0), http://creativecommons.org/licenses/by/4.0/
- Required attribution text: “Castle 3D Model” (https://skfb.ly/ooxSL) by JoshGarber is licensed under Creative Commons Attribution (http://creativecommons.org/licenses/by/4.0/).
- Modifications made: none to the file; the menu scales and re-centers it at runtime.
- Verification status: Verified against the Sketchfab download page and the glTF asset metadata.
- Intended use: 3D castle in the main menu.
- Notes: glTF 2.0, about 60k triangles, 42 PNG textures (about 20 MB). Can be compressed later (WebP/Draco); record any such change here.

### Witch on Broomstick (v1.2)
- Local path: `assets/sprites/witch-on-broomstick/` (license files and README included)
- Original source URL: https://opengameart.org/node/86131
- Creators: Svetlana Kushnariova (Cabbit) and Jordan Irwin (AntumDeluge)
- License stated by the original source: OpenGameArt.org Attribution (OGA BY) 3.0 or later, and Creative Commons Attribution (CC BY) 3.0 or later
- Required attribution text: "Witch on Broomstick" by Svetlana Kushnariova (Cabbit) and Jordan Irwin (AntumDeluge), OGA-BY 3.0 / CC BY 3.0, https://opengameart.org/node/86131. Based on work by Svetlana Kushnariova (see `sources.md`).
- Modifications made: no supplied sprite sheet is currently wired into the rebuilt menu; the current menu uses a tiny procedural pixel-art placeholder texture.
- Source-file or license-document path: `assets/sprites/witch-on-broomstick/LICENSE-*.txt`, `README.txt`, `sources.md`
- Verification status: Source pack and license notes inspected; integration into the menu is still pending.
- Intended use: pixel witches flying around the castle in the main menu.
- Notes: sheet layout is 3 columns x 4 rows, rows N/E/S/W in the NESW files. Use the original NESW sheets for Stendhal-style directions, and preserve the supplied license/source files when integrating them.

### Pixel ground tiles
- Drawn procedurally in `index.html` (original work for this project, no third-party art).

## Stendhal (research only, nothing copied yet)

- Source repository: https://github.com/arianne/stendhal
- Official website: https://arianne-project.org/game/stendhal.html

Potential areas to inspect:
- `data/sprites/outfit/`
- `data/sprites/monsters/`
- `data/maps/tileset/`
- `data/maps/interiors/kalavan/`
- `data/maps/interiors/semos/`
- `data/music/`
- `doc/sources/graphics-outfit.txt`
- `doc/sources/graphics-tileset.txt`
- `doc/sources/graphics-entity.txt`
- `doc/sources/graphics-item.txt`
- `doc/sources/graphics-misc.txt`
- `doc/sources/audio-music.txt`

Licensing and attribution differ between individual assets. Check the relevant documentation and original source before selecting an asset. Entries marked ??? in Stendhal's source records are not usable until their provenance is resolved.

## Other potential sources

### LPC Castle Mega-Pack
- Source: https://opengameart.org/content/lpc-castle-mega-pack
- Creator/reference: bluecarrot16 and potentially other contributors.
- Status: Not reviewed for final asset selection.

### LPC castle
- Source: https://opengameart.org/node/50906
- Contributors listed on the original page include Evert, Sharm, and Xenodora.
- Status: Not reviewed for final asset selection.

### Castle exterior tiles
- Source: https://opengameart.org/content/castle-exterior-tiles
- Creator/reference: Roots.
- Status: Not reviewed for final asset selection.

## Future credits experience

Planned ideas:
- A dedicated credits and licenses section in the interactive bot startup flow.
- Interactive bot messages describing creators, licenses, obligations, and sources.
- A button inside the Mini App that opens the bot using a specific start parameter.
- A credits/legal section inside the Mini App itself (the menu's Credits panel already lists the assets in use).

These are future design ideas, not implemented features.
