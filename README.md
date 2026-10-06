# SKG2E ART/ART-PT Extractor

Extract the sprites of **Skullgirls 2nd Encore** from its `.art`, `.art-pt` and `.foit` data files to lossless PNG or Mugen-ready PCX.

One Python file, standard library only.

> **Unofficial fan project.** It is not affiliated with, or endorsed by, the developers or publishers of Skullgirls. This repository contains no game assets, and the tool never modifies the game: it only reads data files from your own copy.

## Features

- Rebuilds every frame from the `.foit` records: normal sprites, the extra poses stored in `.art-pt`, and the Skull Heart spirit silhouettes.
- **32-bit RGBA PNG**, lossless, with soft anti-aliased edges. One set per color palette (1p, 2p, ...).
- **8-bit PCX for Mugen**: one shared 256-color palette per character (index 0 transparent) plus one `.act` file per palette.
- Optional **layered upscaling** (`--zoom 2|4|8|16`) that keeps the lineart sharp. It needs numpy. It is plain upscaling: the game has no hidden HD sprites.
- `frames.csv` with the size, block count and `A`/`B` values of every frame.
- Prints how much of each `.art` / `.art-pt` file the `.foit` records cover, a quick sanity check when you try a new character.

## Requirements

- Python 3.8 or newer.
- Optional: `pip install numpy` (only for `--zoom`).
- The game's data files, already unpacked from the `.gfs` archives. A `.gfs` unpacker is planned for this repository.

The script is pure Python with no OS-specific code. It was developed on Termux (Android) and its path handling is written for Windows, Linux and macOS.

## Input files

For each character the tool needs `<name>.foit` and `<name>.art`. It also uses `<name>.art-pt` and the palette files `<name>_1p.dds`, `<name>_2p.dds`, ... when they exist.

```
<base>/
  unpacked/
    characters-art/       *.art
    characters-art-pt/    *.art-pt
    characters-foits/     *.foit
    sprites/sprites/      <name>_1p.dds, <name>_2p.dds, ...
```

The tool searches `--base` recursively by file name, so the exact layout does not matter. Only the palette folder has a default (`unpacked/sprites/sprites`); change it with `--palette-dir`. Palette names are matched ignoring case and underscores, so `boss_marie_1p.dds` is used for `bossmarie`.

## Usage

Run it from the folder that holds your unpacked data, or pass `--base`.

```powershell
# Windows (PowerShell or cmd)
python sg_extract.py bossmarie
python sg_extract.py all --format both
python sg_extract.py bossmarie --base D:\skg_data --out D:\skg_sprites
python sg_extract.py bossmarie --all-palettes
python sg_extract.py bossmarie --zoom 4
python sg_extract.py bossmarie --palette D:\pals\boss_marie_1p.dds --palette D:\pals\boss_marie_2p.dds
```

```sh
# Linux, macOS, Termux
python sg_extract.py bossmarie --base /storage/emulated/0/sg_dump
```

| Option | Description |
|---|---|
| `character` | Internal character name (`bossmarie`, `annie`, `filia`, ...) or `all` |
| `--base DIR` | Folder with the unpacked game data (default: current folder) |
| `--out DIR` | Output folder (default: `<base>/extract`) |
| `--palette-dir DIR` | Folder with the palette `.dds` files (default: `<base>/unpacked/sprites/sprites`) |
| `--palette FILE`, `--pal FILE` | A specific palette `.dds`. Repeatable; the first one is the base palette |
| `--format png\|pcx\|both` | `png` = lossless 32-bit (default), `pcx` = 8-bit for Mugen, `both` |
| `--all-palettes` | Write PNGs for every palette found, not just the first |
| `--zoom 2\|4\|8\|16` | Also write upscaled PNGs (needs numpy). Implies `--format both` if you asked for `pcx` |
| `--version` | Print the version |

## Output

```
<out>/<character>/
  1p/sprites/        frames from the .art            (32-bit RGBA PNG)
  1p/extras/         frames from the .art-pt
  1p/silhouettes/    TAG 0xBA layers (Skull Heart spirit)
  2p/, 3p/, ...      only with --all-palettes
  1p_2x/, 1p_4x/     only with --zoom
  pcx/sprites/       only with --format pcx|both: 8-bit PCX, index 0 transparent
  pcx/extras/
  pcx/silhouettes/
  pcx/<name>_1p.act  one .act per palette
  frames.csv         frame, source, tag, foit_off, width, height, blocks, A, B
```

## Using the output in Mugen

- All PCX frames of a character share one 256-color palette, and index 0 is transparent.
- The `.act` files for 2p, 3p, ... keep the same indices, so a color swap is just a palette swap.
- The `.act` files are written in the reverse order that Mugen reads them.
- To get from 32-bit colors to 255, the palette is built with median-cut plus k-means. On Boss Marie the mean color error is 0.8 out of 255 (99th percentile: 6).
- `.sff` and `.air` files are **not** generated yet. Import the PCX files with a Mugen sprite tool such as Fighter Factory.
- The `A`/`B` columns of `frames.csv` are probably the sprite axis (pivot). That is not confirmed yet.

## How the files are organized

The names below were chosen by this project. They are not official.

| File | Project name | Contents |
|---|---|---|
| `.art` | Archive Sprites 2D Compression | Sprite pixels: 3 planes per image, RLE compressed |
| `.art-pt` | Archive Sprites 2D Compression-Extras | More frames with the same format. What `-pt` stands for is not confirmed |
| `.foit` | Frame Object Information Table | One record per frame (position in the archives, size, block mask), palette band names, animation state names |
| `_Np.dds` | (standard DDS) | Color palette: 16 tone columns by N band rows, uncompressed BGRA |

There is no encryption or obfuscation in any of them.

**Pixel data.** Each image is stored as three planes of `blocks * 256` bytes (blocks are 16x16 pixels). Every plane is RLE compressed separately; the compressed sizes are in the `.foit`.

1. `normal`: almost always `0x88`. Probably lighting data; it is not used to paint.
2. `grad`: `line = g >> 5` (0 to 7, darkening of the lineart) and `tone = (g & 31) / 2` (palette column 0 to 15.5; the fraction is used to interpolate).
3. `band`: the palette row, i.e. the zone of the sprite (hair, skin, dress...).

A pixel is transparent when `band == 0` and `line == 0`. Otherwise its color is `palette[band][tone] * (1 - line / 7)`.

| RLE byte | Meaning |
|---|---|
| `00..7F` | `c+1` literal bytes follow |
| `80..FC` | repeat the next byte `(c & 7F) + 1` times |
| `FD n V` | repeat `V` `n` times |
| `FE hi lo V` | repeat `V` a big-endian `u16` number of times |
| `FF a b c V` | repeat `V` a big-endian `u24` number of times |

The blocks of an image are stored as a horizontal strip of width `W = blocks * 16`: block `k` in row `r` (0 to 15) occupies bytes `[r*W + 16k, r*W + 16k + 16)` of the plane.

**`.foit` image record** (little endian):

```
u32 A | u32 B | PRE 00 | 41 | u16 cells | u16 width | u16 height | u16 blocks | TAG | mask[ceil(cells/8)]
u32 offset | u32 size_normal | u32 size_grad | u32 size_band
```

| Field | Meaning |
|---|---|
| `PRE` | `05`: the image is in the `.art`. `04`: it is in the `.art-pt` |
| `cells` | `width * height`. The frame is a grid of 16x16 cells, so it measures `width*16` by `height*16` pixels |
| `blocks` | Number of blocks in the strip. Equals the number of bits set in the mask |
| `TAG` | `0x3A`: color sprite. `0xBA`: Skull Heart spirit silhouette layer |
| `mask` | One bit per cell, least significant bit first, row by row. The strip blocks are placed, in order, in the cells whose bit is set |
| `offset`, `size_*` | Where the three compressed planes start in the archive, and their compressed sizes |
| `A`, `B` | Probably the pivot of the frame (Boss Marie idle: `953, 630`). Not confirmed |

## Status and known limits

- The format was worked out by studying the data files. The game executable was not reverse engineered or modified.
- Checked against every record of **Boss Marie** (233 of 233 frames: 212 in `.art`, 21 in `.art-pt`) and **Hitspark** (21 of 21). Other characters may show differences. Please report them.
- **Nothing has been tested by loading modified files back into the game.** This repository documents and extracts the formats; it makes no claims about modding the game.
- The animation data in the `.foit` is not decoded yet (only the state names are readable).
- Some `.foit` records are followed by extra signed 32-bit values (hitboxes? offsets?) that are not decoded.
- The game's own shadows are not stored in these files; they are probably generated at runtime.

## Roadmap

- `.gfs` unpacker.
- Repacking edited sprites into `.art` / `.art-pt` and the `.foit`.
- Decoding the animation tables.
- Automatic `.sff` / `.air` generation for Mugen.

## Reporting problems

Open an issue with the character name, your OS and Python version, and the `frames: ... (covers .art X%, .art-pt Y%)` line the tool prints. Coverage far below 100% on a character usually means a record variant that is not handled yet.

## Legal

This project is not legal advice. It does not include or distribute any game files, and you need your own copy of the game to use it. Skullgirls and related names are trademarks of their respective owners, and the sprites you extract remain their property, so do not redistribute them. Check the game's EULA and the laws of your country before using or sharing anything derived from the game.
