<a id="top"></a>

<div align="center">

# Alundra 2 — reverse engineering notes

**PlayStation · NTSC-U · SLUS-010.17 · Matrix Software, 1999**

[![Language: English](https://img.shields.io/badge/lang-English-1f6feb?style=for-the-badge)](#en)
[![Язык: Русский](https://img.shields.io/badge/язык-Русский-d73a49?style=for-the-badge)](#ru)

![Type](https://img.shields.io/badge/type-documentation%20only-8250df?style=flat-square)
![Platform](https://img.shields.io/badge/platform-PS1%20(MIPS%20R3000)-lightgrey?style=flat-square)
![Maps](https://img.shields.io/badge/maps-512-2da44e?style=flat-square)
![Script%20commands](https://img.shields.io/badge/script%20commands-236-2da44e?style=flat-square)
![Models](https://img.shields.io/badge/models-555%20%2B%203260%20on%20maps-2da44e?style=flat-square)

**🇬🇧 [English](#en) &nbsp;·&nbsp; 🇷🇺 [Русский](#ru)**

</div>

---

## At a glance · Коротко

| | |
|---|---|
| 🗺️ | **Maps / Карты.** 512-entry `GRP.GP`, 24 slots each: textures, geometry, objects, scripts, text, sound banks |
| 🧩 | **Scripts / Скрипты.** 236-command event language plus a separate bytecode VM for enemy AI |
| 🧍 | **Models / Модели.** Rigid-part models with per-frame absolute poses, no bone hierarchy |
| 🔤 | **Text & fonts / Текст и шрифты.** 16-bit message format, 12×24 2-bpp font, a code-level route to Cyrillic |
| 🎵 | **Audio / Звук.** CD-XA tables, a MIDI-like sequencer with SPU samples, global sound-effect banks |
| 🎮 | **Game systems / Системы игры.** Damage, items, shops, medals, save-card format, three arcade games |
| 🛠️ | **Developer leftovers / Наследие разработчиков.** Debug HUD, menus and `a2profile.ini`, still in the retail build |

**EN:** A documentation-only repository. Formats, addresses and rules, each marked with how well it has been verified. No code, no game data.

**RU:** Репозиторий только с документацией: форматы, адреса и правила, у каждого помечена степень проверенности. Без кода и без данных игры.

---

<a id="en"></a>

# 🇬🇧 English

## Table of contents

1. [What this is](#en-1)
2. [Disc and executable](#en-2)
3. [Containers: PAK and GRP.GP](#en-3)
4. [Compression and image format](#en-4)
5. [Maps: the 24-slot entry](#en-5)
6. [Map geometry (slot 11)](#en-6)
7. [Objects, types and scripts (slot 10)](#en-7)
8. [The script language](#en-8)
9. [Text, dialogs and fonts](#en-9)
10. [3D models and animation](#en-10)
11. [Model materials](#en-11)
12. [Enemies, AI bytecode, bosses, damage](#en-12)
13. [Player state, items, shops, saves](#en-13)
14. [Audio](#en-14)
15. [Mini-games](#en-15)
16. [Cutscenes (EVENT00, PAKVCAST)](#en-16)
17. [Developer leftovers: debug HUD, menus, a2profile.ini](#en-17)
18. [Overworld graph](#en-18)
19. [Open questions](#en-19)
20. [Notes on method and legal](#en-20)

---

<a id="en-1"></a>
## 1. What this is

A reverse-engineering write-up of **Alundra 2** (PlayStation, Matrix Software, 1999). Target build: **NTSC-U, SLUS-010.17**.

The goal was to understand the game well enough to rebuild its tooling: unpack every container, read maps, scripts, models, animations, text, audio, and know how the engine ties them together. This repository is documentation only. There is no game data and no code here. You need your own copy of the disc.

How to read the confidence marks used below:

| Mark | Meaning |
|---|---|
| ✅ | Checked against all the data (counts given) and/or against the game code, and in many cases against a running copy of the game (RAM and VRAM dumps) |
| 🟡 | Read from code or consistent with data, but not seen working in the game |
| ❓ | Guess or unknown |

Addresses are PlayStation RAM addresses (`0x8xxxxxxx`). "Slot" always means a numbered block inside a map entry unless stated otherwise.

---

<a id="en-2"></a>
## 2. Disc and executable

### 2.1 Files

| File | Size | What it is |
|---|---|---|
| `SLUS_010.17` | 305 152 | Main PS-X EXE. Load `0x80017000`, entry `0x8003CF4C`, SP `0x801FFFF0` |
| `GAME00.BIN` | 805 144 | Overlay: the game proper (id 4) |
| `EVENT00.BIN` | 488 592 | Overlay: cutscene/event player (id 5) |
| `GAME01.BIN` | 456 376 | Overlay: mini-game 1 (id 6) |
| `GAME02.BIN` | 281 432 | Overlay: mini-game 2 (id 7) |
| `GAME03.BIN` | 173 528 | Overlay: mini-game 3 (id 8) |
| `DATA/GRP.GP` | 207 MB | 512 map entries |
| `DATA/PACK0.PAK` | 13 MB | models, animations, images, fonts, system text, music tracks |
| `DATA/PACK1.PAK` | 0.7 MB | code overlays (title screen, intro, boss modules) |
| `DATA/PAKOMAKE.PAK` | 4.4 MB | bonus content ("omake"), 117 records |
| `DATA/PAKVCAST.PAK` | 79 MB | cutscene data, 311 records |
| `A2STR.STR` | 18.7 MB | intro movie |
| `MONO.XA`, `STEREO.XA` | 22.5 / 180 MB | CD-XA: voices and cutscene music |

Raw image: 2352 bytes per sector, Mode 2, ISO 9660, about 258 244 sectors. The build uses Sony library versions from early 1998 (sys.c 1.140, bios.c 1.86, intr.c 1.75).

Developer path leftovers inside the binaries: `d:\a2\bin\` (`a2.prm`, `a2.frx`, `a2.tim`, `aaa.mbz`, `fardome1.mbz`, `aaa.map`, `fardome1.map`, `light.tmp`) and, on the dev machine, one file per map: `data\grp\grpNNN.gp`.

### 2.2 Overlays ✅

All five overlays are linked to the same address, **`0x80084A98`**, and are loaded one at a time over each other. The first dword of each file is the overlay id. The main EXE holds the file name table (`0x80052170`) and the module scheduler.

Code between `0x80010000` and `0x80017000` is a **separate library overlay** ("AL22"): PACK0 records **76 to 79**, copied to `0x80010000` according to a byte table at `0x800521E8` (the game-mode number picks one of the four). It contains the GTE math library (`RotMatrix`, `MulMatrix`, …) and the **model rasterizer**.

### 2.3 Module state machine ✅

The game is a small state machine. The scheduler (`0x8001A228`) reads the next module number from the word `0x80060888`, loads the file named in the table at `0x80052170` (8 bytes per row: filename pointer, entry address), calls the entry with the argument in `0x800608CC`, and uses the **return value as the next module**. `-1` means "back to the title".

| # | File | Entry | Role |
|---|---|---|---|
| 0 | `game00.bin` | `0x8008968C` | The game itself (map loop) |
| 1 | `event00.bin` | `0x80087684` | Cutscene player; argument = event number |
| 2 | `game00.bin` | `0x80089338` | Title screen (code lives in PACK1 record 34) |
| 3 | `game01.bin` | `0x80084D50` | Mini-game 1 |
| 4 | `game02.bin` | `0x80084D4C` | Mini-game 2 |
| 5 | `game03.bin` | `0x80084CA4` | Mini-game 3 |
| 6 | `game00.bin` | `0x8008939C` | Picture sequence (PACK0 24 to 28) and sound, probably credits 🟡 |
| 7 | `game00.bin` | `0x80089404` | Intro movie (PACK1 record 35, plays `A2STR.STR`) |

Boot order: module 7 (intro), then 2 (title, with a 900-frame idle timeout that returns `-1` and replays the intro), then the module-0 loop. Starting a new game goes to event **101** (module 1). The scheduler only reads the module word *before* loading an overlay, so poking a new number mid-game does nothing; it has to be there from the first frame.

The scheduler is also where mini-games are entered from maps: script command 198 with code 18/19/20 starts modules 3/4/5 (from maps **494, 493, 495**).

### 2.3.1 Memory regions worth knowing

| Address | What |
|---|---|
| `0x8011D5C0` | Object array: 60 objects × 804 bytes (index 0 is the player) |
| `0x80066BC8` | Saved game state, `0xC88` bytes (see §13) |
| `0x80066C08` | Story flag bitfield |
| `0x8006740C` | Item counts, 100 bytes |
| `0x80134E90` | 260-byte map entry header, as loaded |
| `0x80134EE0 + 8k` | Pointers to map slots 10 to 14 |
| `0x8012C3A0` | Dialog state |
| `0x800EE6F4` | Script command handler table (236 entries) |
| `0x80082830` | Sound bank table (127 × 12 bytes) |

---

<a id="en-3"></a>
## 3. Containers: PAK and GRP.GP

### 3.1 PAK files ✅

`PACK0`, `PACK1`, `PAKOMAKE`, `PAKVCAST` share one layout:

- Header: exactly **768 slots × u32**. Each slot is an absolute file offset, `0` = empty.
- Non-zero offsets ascend. Record size = next offset minus this one; the last record runs to end of file.
- Filled slots: PACK0 693, PACK1 37, PAKOMAKE 117, PAKVCAST 311.
- PACK0 contains many byte-identical duplicates (45 groups, about 1.86 MB wasted).

The game reads PAK records through `0x8001BD70(dest, first, last, maxsize)` after selecting the pak with `0x8001BD1C`.

**What lives where in PACK0:**

| Records | Content |
|---|---|
| 14, 15 | system notices, character and boss names |
| 23 to 28 | full-screen pictures (title, 768×488 sequence) |
| 29 to 38 | hero/misc textures (32 to 38: seven variants of the hero page, 256×256, 4-bit) |
| 39, 40 | `PC00_KEN` and `PC00_TATE`: five swords and five shields |
| 68 to 75 | hero models and animation banks (see §10) |
| 76 to 79 | AL22 code overlay |
| 299, 300, 516 | item names, option text, shop text |
| 304 to 310ff | event images (256×256, 6 palettes) |
| 375 to 402 | image sets in fours (256×256 image, table, 256×16 image, table) |
| 537 to 552 | fonts (see §9.4) |
| 557, 559 | mini-game 2 and 3 data |
| 581 onward, in threes | music tracks: sequence, bank header, samples (§14.2) |

About 212 PACK0 records start with a 32-byte ASCII name (`PC00_KEN`, `O01_01`, `M01`, `TA05`, …) and are models or animation banks.

### 3.2 GRP.GP ✅

- Header: `0x800` bytes = **512 × u32** absolute offsets. Offsets need not be sector-aligned (the CD reader keeps the `offset & 0x7FF` remainder).
- **Entry index = map number.** There is no other mapping. `startmap = 213` means entry 213.
- Empty slots: 236, 289, 292, 299, 304, 423, 496, 497. Entry 0 is special (0x3620C bytes, mostly zero).
- Each non-empty entry is a **260-byte header** followed by the slot blocks back to back.

**Entry header (260 bytes):**

| Offset | Content |
|---|---|
| `0x00` | 24 pairs `(offset, size)`, 8 bytes each. Offsets are relative to *start of data* (entry + 260) and are just running sums of the sizes |
| `0xC0` | 4-byte area tag: two letters + two digits (`hi00`, `pa00`, …) |
| `0xE8` (half-word 116) | PACK1 record number holding this map's boss code (non-zero on 15 arena maps) |
| rest | more fields up to 260 |

Area tags (count of entries): pm 65, ki 43, pz 34, cg 28, gy 28, st 28, ud 25, yk 24, di 23, hi 21, pd 18, sn 17, fc 16, yo 16, aw 14, fm 12, pb 9, re 9, oo 8, fb 7, pa 6, se 6, bu 5, tb 5, nd 4, sh 4, fa 3, pc 3, ya 3, ha 2, ta 2. Special entries: 400 `pmwa`, 498 `over`, 507 `star`, 510 `uso`, 511 `chec`, and ten `test` maps (499 to 506, 508, 509), which are full small maps.

The loader (`0x800AE4A8`, `0x800AF1F0` onward) reads the 260-byte header, then reads slots in groups (0-3, 4-7, 10-13) into a work buffer, then rewrites the header's offset fields into **RAM pointers**. A byte table at `0x800F095C` picks the buffer: `0x80` for 152 big maps (`0x8019C000`, `0x50800` bytes), `1` for 342 normal ones (`0x801AB000`, `0x41800`), `0` for the empty slots.

### 3.3 Slot overview ✅

| Slot | Content | Present in |
|---|---|---|
| 0 | Main textures (type-10 image container, up to 10 streams × 256×256, 10 palettes) | all |
| 1 | One 128×256 8-bit image | 47 outdoor maps |
| 2 | One 256×128 4-bit image, 2 palettes | all |
| 3 | Three 256×256 4-bit images, 3 palettes | all |
| 4, 5, 6, 8, 9 | empty | none |
| 7 | Sound-effect banks | all |
| 10 | Objects, object types, script streams | all |
| 11 | World geometry chunks | all |
| 12 | Fixed 5764 bytes, purpose unconfirmed ❓ | the same 47 maps as slot 1 |
| 13 | Fixed 184 bytes, lighting 🟡 | 383 maps |
| 14 | Map text (or a 28-byte stub) | all |

---

<a id="en-4"></a>
## 4. Compression and image format

### 4.1 Image container ✅

Used by PACK0 images and map slots 0 to 3.

```
u32 type        number of streams (1, 2, 3 or 10)
u32 palettes    number of 256-colour palettes
u32 size        width | (height << 16)
u32 sizes[...]  compressed size of each stream; total header = 0x34 bytes
palettes        palettes × 512 bytes, 256 × PS1 15-bit colours (bit 15 = semi-transparent flag)
streams         one after another, whole record padded to 4
```

A stream of size 0 is a skipped slot (the loader still advances the VRAM x position). Checked: 207 of 207 PACK0 streams, 2781 of 2781 slot-0 streams, 2059 of 2059 slot 1-3 streams decode to exactly width×height (or half that at 4-bit) with zero input left over.

### 4.2 Stream methods (`0x8001C860` dispatcher) ✅

First stream byte = method.

| Byte 0 | Method | Notes |
|---|---|---|
| `01` | RLE | tokens as below, but `0x80` means *repeat the next byte n times* |
| `02` | LZ | used by everything found on the disc |
| `03` | raw | data after the 4-byte header is copied as is |

Methods 1 and 3 exist in code but no data on the disc uses them.

Header of method 1/2:

- Byte 1: colour depth. `0` = 4 bits per pixel (output = w×h/2 bytes), anything else = 8 bits (output = w×h).
- Byte 2: if `0`, the first output byte is stored in byte 4 and tokens start at byte 5; otherwise tokens start at byte 4.
- Byte 3: unused.

Token `t`:

- Length `n = (t & 0x3F) + 1`. If bit `0x40` is set the length is 14-bit: `((t & 0x3F) << 8 | next byte) + 1`.
- Bit `0x80` set: **copy**. Next byte + 1 is the distance back into already produced output; copy `n` bytes one at a time (overlap allowed, which gives repeats).
- Bit `0x80` clear: `n` literal bytes.

The game decodes into a 16 KB window (`0x80061F28`) and uploads in 16 384-byte pieces with `LoadImage`. Type-2 images (two streams) put the second one to the right of the first in VRAM.

### 4.3 Where each map image goes in VRAM ✅

This is the layout the map loader produces (verified: for map 213, 225 024 of 225 024 words match a VRAM dump of the running game):

| Slot | Palettes | Pixels |
|---|---|---|
| 0 | block (768, 488), 256 wide, one row per palette (`488 + j`) | stream k at x = 320 + 64·k, y = 0 (4-bit = 64 words wide, 8-bit = 128) |
| 1 | row (768, 503) | 128×256 8-bit at (960, 0) |
| 2 | rows (768, 498) and (768, 499) | 256×128 4-bit at (512, 384), the bottom half of texture page `0x18` |
| 3 | one 64×16 block at (896, 464); palette p = four rows `464 + 4p …` | three 256×256 4-bit streams at (320 + 64·k, 256) |

An 8-bit stream takes two page columns, which is why a 4-bit stream after it is sometimes missing.

### 4.4 The third scheme: cutscene textures ("MOD") ✅

`PAKVCAST` texture blocks begin `01 'M' 'O' 'D'`, then 256 palette words, then a stream (458 of 458 blocks decode):

- Skip one zero byte.
- Read a flag byte, **most significant bit first**. Bit 0 = one literal byte. Bit 1 = a two-byte pair `(a, c)`: copy `c + 3` bytes starting at position `(row − 1) · 256 + a`, byte by byte, where `row` is the row of the current output position.
- A virtual row of zeros sits above row 0.
- Result is always 256×256, 8 bits.

Blocks tagged `00 'M' 'O' 'D'` hold palette sets (a raw 5120-byte block is ten palettes of 256).

---

<a id="en-5"></a>
## 5. Maps: the 24-slot entry

Quick recap of what a map is: textures (slots 0-3), sound (7), objects + scripts (10), 3D world (11), lighting-ish data (13), a sky-like block (1, 12) and text (14). The next sections go through the important ones.

### 5.1 Slot 14: map text ✅

```
u32  map number
u32  n
i32  offsets[n + 1]      (-1 = no message)
text                     16-bit units
```

Message k starts at `8 + 4·(n+1) + offsets[k]`. Details of the unit encoding are in §9. Checked on all 511 blocks: every offset lands inside its block and exactly one message of each map starts at offset 0.

### 5.2 Slot 7: sound banks ✅

```
u32 n                     number of banks (1 to 11)
u32 offsets[n]            first is always 68 (0x44)
bank:
    prefix   (28 + 8k bytes)
    "SShd" header          at prefix + 12
    SPU-ADPCM data         length = u32 at header + 4
```

Checked on 2163 of 2163 banks (27.8 MB of ADPCM, all blocks structurally valid).

Prefix: `w0` = offset of the header in the combined buffer, `w1` = w0 + header length, `w2 = 12 + 8k`, then `k` entries of 8 bytes `(u16 id, u16 sample index, 4 bytes)`, `0xFFFFFFFF`, 16 bytes of tail. The sound `id` is **global**: every map that uses id 3011 carries the same sample set. Across all maps there are only 186 distinct sample sets and 586 distinct ids. See §14.3.

### 5.3 Slot 13 and 14 stubs 🟡

Slot 13: 184 bytes = 92 half-words. First six numbers are two RGB triples (for example `128,128,128` and `150,121,109`), then signed fixed-point vectors. Combined with the dev-time file name `light.tmp` (60 bytes, loaded into the slot 13 pointer in the developer build) this is lighting: ambient and directional colours plus directions. Outdoor maps use neutral white `255,255,255`; interior maps use warm tones.

Slot 12: 5764 bytes = 1 word + 96 × 60-byte records. It only exists on maps that have slot 1, and the debug path name `fardome1` fits a distant sky dome. This is a guess ❓. Slot 1 itself is a 128×256 8-bit picture; it is probably a background/sky image but nobody has watched the game use it.

---

<a id="en-6"></a>
## 6. Map geometry (slot 11) ✅

Slot 11 is the world mesh, split into chunks.

### 6.1 Slot table (560 bytes = 140 words)

- Word 0 = 560 (start of data).
- Words 1 to 44: chunk offsets (relative to start of slot), filled from the front; count varies from 0 to 44 per map.
- Words 126 to 129, 132 to 136, 139: fixed pointers (always filled, always valid; the last is close to the end of the slot). Their meaning is unknown ❓.
- All others are zero.

### 6.2 Chunk header

| Offset | Type | Meaning |
|---|---|---|
| `+0` | u16 | Flags. `0xC000` normally; bit `0x8000` = visible (scripts toggle it) |
| `+2` | 6 × s16 | Local bounding box min/max |
| `+14` | 3 × s16 | World anchor x, y, z |
| `+20` | 9 × s16 | 3×3 orientation matrix, 4096 = 1.0 |
| `+40` | u16 | Polygon count |
| `+42` | u16 | Vertex count |
| `+44` | u32 | Offset of the vertex array (relative to slot start) |
| `+52` | 32 bytes | Optional name string (used by script commands 194 to 196) |
| `+84` | | Start of the polygon stream |

Only 275 of the 5399 chunks have a name (`door1_1`, `neji01`, `wing02`…`wing07`, `paati1`…`paati4`, `tru02`). Named chunks are the ones scripts show, hide, move or rotate.

### 6.3 Polygons

44 bytes each, or 80 if bit `0x04` is set in the tag (a second 36-byte overlay packet follows, used for decals and shadows). These are ready-made PS1 GPU packets:

| Offset | Meaning |
|---|---|
| `0..3` | DMA tag; byte 0 carries flags |
| `4..7` | vertex indices v0..v3 (triangle: v3 = v2) |
| `8..10` + `11` | colour 0 (RGB) and the GPU command (`0x3C` G4, `0x34` G3, `0x38` GT4, `0x30` GT3, and `+2` variants for semi-transparency) |
| `12..14`, `16..18`, `20..22` | colours 1 to 3 |
| `24..35` | UV and CLUT / TPAGE words: `u0 v0 clut`, `u1 v1 tpage`, `u2 v2`, `u3 v3` |
| `36..39` | `E2` texture-window word |
| `40..43` | DMA terminator `00 FF FF FF` (or `03 FF FF FF`) |

A quad is any command with bit `0x10` set (`0x3C 0x38 0x3E 0x3A`). Quads are stored in **contour order** 0-1-2-3, so split them as (0,1,2) and (0,2,3). Splitting in GPU "Z" order produces bow-ties or holes.

### 6.4 Vertices

12 bytes each: `x` at +0, `y` at +4, `z` at +8 (s16). The other half-words are runtime scratch. World position = local + chunk anchor. Y is height.

**The chunk layout is exact:** polygon stream ends precisely at the vertex array, and the vertex array ends precisely where the next chunk starts, on 5399 of 5399 chunks.

---

<a id="en-7"></a>
## 7. Objects, types and scripts (slot 10) ✅

### 7.1 Header (28 bytes)

| Word | Meaning |
|---|---|
| 0 | Offset of the **type table** (44 bytes per record) |
| 1 | Offset of the **placement list** (16 bytes per record, ends with a record whose first byte is `0xFF`) |
| 2 | Offset of the **script catalog** |
| 3 | Slot size |
| 4 | Slot size − 8 |
| 5 | `28` (header size) |

Between byte 28 and the catalog lie the script streams. After the placement list there is usually a small tail; the type table and the model/animation blobs sit in the rest.

### 7.2 Placement record (16 bytes)

| Offset | Meaning |
|---|---|
| `+0` | Type index into the type table (`0xFF` = end of list) |
| `+1` | Bits 0-2 × 64: facing/variant; mostly even values |
| `+2, +4, +6` | x, y (height, stored negated), z as s16, shifted left by 12 when spawned (20.12 fixed point) |
| `+8` | Byte, minus 1 goes to object field `+0x30C` |
| `+10, +11, +13` | Three parameter bytes (object `+0x2AC, +0x2AD, +0x2AF`). Door/exit objects read their own parameters here |
| `+14` | u16 → object `+0x2B4` (also the save-flag number for toggled objects) |

Statistics over 6127 records on 503 maps: 12.2 placements per map on average, maximum 31, 24 maps with none. Object slot = placement index + 1, so slots 1 to 31 come from maps; **slots 32 to 59** are reserved for runtime objects (projectiles, effects, the player's helpers). An already-active slot is not spawned again.

### 7.3 Type record (44 bytes)

| Offset | Meaning |
|---|---|
| `+0` | u16 object id (`0xFFFF` = end) |
| `+2` | Class: 2 enemy, 4 boss, 11 item, 12 NPC / trigger / effect, 14 "D" object |
| `+4` | Flags word. Bits 11-13 = variant (→ object `+0x30D`); bits 14-15: `0x8000` = no model setup, `0x4000` = also load an animation bank |
| `+28` | Model offset in the slot |
| `+32` | Animation bank offset |
| `+36` | Start index into the tpage table |
| `+37` | Start index into the clut table |
| `+38, +39` | u and v shift applied to every vertex (64, 128, 192 are common) |
| `+40` | Extra block (32 pointers) |

**Id ranges:** A 1 to 87 enemies, B 101 to 126 bosses, N 200+ NPCs, I 300 items, T 401 to 499, D 501 to 524 objects, E 600+ (projectiles, boss parts, effects).

**Models live here.** Enemies, bosses and NPCs are not in the PAK files at all. Each map embeds the models of the objects it uses, together with their animation banks, inside slot 10. Over all maps there are 3429 embedded models, 546 distinct. Parts parse exactly on 4433 of 4442 models.

Object structure in RAM (804 bytes): `+0` active, `+0x0A` type id, `+0x0C` class, `+0x10` flags, `+0x18/1C/20` position (20.12, Y negated), `+0x38` HP, `+0x3A` max HP, `+0x14/+0x16` animation, `+0x7C` speed, `+0x248..+0x258` AI state (§12.2).

### 7.4 Script catalog

`0x80096C68(group, number)`: catalog word `[group + 1]` points at a table of u32; entry `number` is the stream offset (0 = none, −1 = disabled). Groups 0 to 8 (504 of 509 maps have all nine). 5106 streams on all maps, and the first byte of every stream is below 236 (the number of commands).

---

<a id="en-8"></a>
## 8. The script language

### 8.1 Interpreter ✅

Entry `0x8009E17C`. Each step reads command byte `c` (0 to 235), calls `handler[c](objectArray, cmdPtr)` from the table at `0x800EE6F4`, and the handler returns **the next command pointer**. Return `0` = stop this script; return the same pointer = wait here (used by every "wait" command). Command **0** is the terminator (length 1), followed by zero padding to 4 bytes.

Command lengths are known for 232 of 236 commands. With them, 4539 of 5064 streams parse exactly (terminator reached, only zeros after). Conditional jumps have two lengths (fall-through and jump); linear parsing takes the short one.

### 8.2 Object selector ✅

Most commands take a selector byte as their first argument (`0x80096E64`):

| Value | Selects |
|---|---|
| `0xFF` | nobody |
| `0x00..0x7F` | the object in that slot (if active) |
| `0x80` | the script's own object |
| `0x81` | all active objects of class 2, 3, 4 |
| `0x82` | all of class 9 |
| `0x83` | all active except class 5 |

Selected pointers go into an array at `0x8012C328`; the command then applies to each.

### 8.3 Commands with confirmed meaning

| # | Len | Meaning |
|---|---|---|
| 0 | 1 | End of script |
| 1 | 3 | Wait N frames (u16) |
| 2 | 7 | System message (text from the main EXE table) |
| 3 | 4 | Show map message: `[3][message no][speaker group][flags]`. Flag `0x80` = don't block |
| 5, 6 | | Branch on dialog answer (two-choice) |
| 8 | | goto (s16) |
| 9 | | call (return stack of 4) |
| 10 | 1 | return |
| 11 | 1 | restart this stream from its beginning (the usual last command; 1143 uses on 284 maps) |
| 12 / 13 / 14 | 3 | set / clear / toggle story flag (u16) |
| 15 to 22 | 5 | jump / call / return conditioned on a flag; 22 is a flag-gated block |
| 23 to 27 | | object variables (16 × u16): set, add, subtract, a+b, a−b |
| 29 | 4 | wait for flag value |
| 30, 217 | 3, 5 | open a 2-choice / 4-choice dialog (texts by number) |
| 31 | 3 | return player from a special state (`0xFFFF` skips) |
| 32 | 1 | reset player mode |
| 33 / 34 | | give / pay money (0 = use the saved work amount) |
| 35 / 36 | | give / remove one item (by id) |
| 37 to 39 | | heal HP / heal 2nd counter / full heal |
| 40, 41 | | cinematic-mode toggles 🟡 (41 also clears the 12-entry floating number table) |
| 42 | | fade (RGB, frames, mode) |
| 53 | | store number of selected objects |
| 56 | | condition: object is in selection |
| 61 | | condition over selected objects (sets state, used with 70/71) |
| 64 | | condition: money ≥ N |
| 70, 71 | 3 | jump if state ≠ 0 / = 0 |
| 72 to 77 | | call / return / block forms of the same state test |
| 78, 79 | 1 | IF / IFNOT prefix: run the next command only if state allows |
| 87 | | condition: dialog active (most common unnamed command, 1861 uses) |
| 88 | 2 | spawn the object from placement index (byte − 1); error stop if it fails |
| 89, 96 | | put object at position (×4096) |
| 91, 99, 112 | | set object mode bytes |
| 92, 98 | | mode bits |
| 93 | 3 | per-object helper call (animation parameter) |
| 94 | 4 | set animation/frame for a group of objects (s16 value; −1 = use default) |
| 95, 119, 124 | | animations by number |
| 100, 101 | 6 | set / clear bits in object flag word (32-bit mask) |
| 103 / 104 | | "switch on" / "switch off" a toggleable object (saves a flag, plays frame 130 or 131) |
| 105, 106 | | sound by object kind / stop sound |
| 109 | | height offset |
| 113 | | set speed field |
| 114 | 8 | set group offset vector (three s16 × 16 into `+0xB8/BC/C0`) |
| 115 | | player state number (`+0xB60`) |
| 116 | 9 | place at another object's position + rotated offset |
| 120 | | one-shot enable for object kinds ≥ 501 |
| 128 | 4 | continuous rotation |
| 152 to 158 | | camera control: target, distance, angle (smoothed) |
| 167, 168 | | camera reset / mode |
| 180, 181 | 3 | wait for flag set / clear |
| 190, 191 | | set / clear a global switch |
| 192 | 6 | **play cutscene**: event number − 100 in bytes 1-2, return map in bytes 3-4 |
| 193 | 2 | layer/mode byte |
| 194, 195, 196 | | **show/hide, move, rotate a named map chunk** (32-byte name follows) |
| 198 | 2 | **switch module**: code 18/19/20 → mini-game 1/2/3; 15 and 22 → game with a mode field; others are save/load requests |
| 200 | | condition: two half-words equal over selection |
| 220 | 1 | back to title |
| 221 | | wait on state flag |
| 222 | | set `worldcnt` |
| 225 | 1 | hand in all item-68 crystals as score points |
| 226 | | play a sound from a small table |
| 228 | 2 | set combo level directly |
| 229 | | add medals (u16) |
| 230, 232 | | play sound with `id + 100`; set sound volume |

Roughly 140 of 232 commands (short object operations and camera) are still unnamed. A complete per-command table with handler address, length and usage count exists but is too long for this file.

### 8.4 Flags ✅

A single bitfield at `0x80066C08` (`byte = n >> 3`, `bit = n & 7`, low bit first). Helpers: check `0x8001D398`, set `0x8001D3D4`, clear `0x8001D414`, toggle `0x8001D454`.

About 7000 flag numbers are actually used (0 to 511: 3090 references; 512 to 1023: 683; 1024 to 2047: 389; 2048 to 6927: 2849). Flags are grouped by 256 into **areas**: blocks 0 and 2 are global story state, the rest belong to groups of neighbouring maps (block 5: maps 20 to 25, block 6: 50 to 59, block 9: 70 to 85, block 10: 106 to 127, block 19: the medal merchant and the three arcade maps, block 27: only map 220, and so on). Some low flags (0, 1, 2) are temporary and flip between saves.

### 8.5 Cutscene hand-off

Command 192 calls `0x80098A18(event, map)`: it sets module := 1, argument := event, and stores the map to return to (`+0xB14`, plus `+0xB16 := 2`). The EVENT00 player does the rest (§16).

---

<a id="en-9"></a>
## 9. Text, dialogs and fonts

### 9.1 Message encoding ✅

16-bit units. `0xFF00 | c` is the printable ASCII character `c`. Anything else is a control code. Messages end on unit `0` or `1`. Verified: all 2855 messages in the maps end on 0 or 1 (2542 / 313), and messages sit back to back with no gap on 2344 of 2344 table entries.

Interpreter: 32 line states of 32 bytes at `0x80068E40`; unit `< 18` jumps through the table at `0x80017978`.

| Code | Meaning |
|---|---|
| 0, 1 | End of message (two window behaviours) |
| 2 | Wait for button / page break |
| 3 | New line; if the window is already full, behaves like 2 |
| 4 + param | Line style (colour), params like 7, 1, 3, 6, 4 |
| 5 + param | Print speed delay |
| 6 | Speed from user setting (table `0x80055C64`) |
| 7 + param | Second style field (param always 0 in data) |
| 8 | Centre the next line: `x = (320 − width) / 2` |
| 9 | End of inserted string |
| 10 | Insert system message 10 (hero name) |
| 11, 12 | Insert a number from game variables |
| 13 + param | Insert a string from 64-byte slots at `0x80069290` |
| 14 to 17 | Button glyphs (12 px wide) |

Codes 4, 5, 7 and 13 take **one parameter unit**. A parser that stops at the first `1` will cut messages in half. Five units `171 214 151 222 190` plus one of `20..23` appear in 292 maps; they are not ASCII and are a leftover of the Japanese build that the US build probably never shows ❓.

### 9.2 Window rules ✅

Window width is 288 px (`0x120`). Latin letters are 6 px each. The original allows three lines per page, each ≤ 282 px. **There is no automatic word wrap**; every break is a code 3. All 3303 original messages (maps plus system text) satisfy these rules.

### 9.3 System text ✅

Same record format as slot 14: `u32 id, u32 count, i32 offsets[count + 1], units`.

| PACK0 record | Content |
|---|---|
| 299 | 257 entries: items, weapons, shields, key items, relics |
| 300 | 28 entries: options and settings |
| 14 | 30 entries: notices (chests, doors, …) |
| 15 | 99 entries: character and boss names |
| 516 | 62 entries: shops, token exchange, element lords |

More strings (menus, mini-game labels) are inside the EXE and the overlays and have not been catalogued.

### 9.4 Fonts ✅

| PACK0 record | Content |
|---|---|
| 537 to 540 | Japanese-build fonts |
| **552** | **Dialog font used by the US build**: 96 glyphs × 73 bytes |
| **551** | **Small/menu font**: 96 glyphs × 49 bytes |
| 538 | 320 glyph × 72 byte block never loaded by a constant in the US build (free space) |
| 541 to 550 | alternative styles (bold, script, narrow…) |

Glyph layout (large): 72 bytes of bitmap, **12×24 pixels, 2 bits per pixel, 3 bytes per row, MSB-first**, then **one width byte at the end** (offset 72; offset 48 for the small font, 12×16). Width 0 or ≥ 12 means the default of 6; the small font is proportional (5 to 10).

Loader `0x80021638` fills `0x8014C880`; US path loads record 552 to `0x8014C888` (instruction at `0x800216B4`, `addiu s0,zero,552`) and record 551 to `0x800693A0`. Blitter `0x8002A230` writes into VRAM at (960, 472) page. Width lookup `0x8001FAE0`, per-glyph address `stride × c − 2264`.

The global at `0x80060854` is `1` at start; with it the text mode (`0x80067930`) becomes **3 (US)**. In mode 3 the code uses `unit & 0xFF` as the character and `char − 32` as glyph index, and the range check accepts **32 to 255**. The engine therefore already supports **224 glyphs** like the PAL builds, but the US font record only supplies 96.

**Cyrillic path (code-verified, untested in an emulator 🟡):** write a 224-glyph font with CP1251 positions (`0xC0..0xFF`, Ё `0xA8`, ё `0xB8`, « » `0xAB/0xBB`, — `0x97`, … `0x85`, № `0xB9`) into record 538 (23 040 bytes, plenty), and change one instruction in the EXE from record 552 to 538 (`0x24100228` → `0x2410021A`). File sizes stay the same; only 13 sectors change. The area after the font buffer (up to `0x80157708`) has no absolute code references, so 16 352 bytes of extra glyphs fit. The small font cannot be widened the same way: `0x8006A800` sits 512 bytes after it.

Rewriting map text can grow a record: GRP.GP entries are addressed by byte offset, so the table can be rebuilt and the file moved to the end of the disc image (EDC/ECC for Mode 2 Form 1 must be recomputed; those checks were verified on 3240 of 3240 real sectors).

---

<a id="en-10"></a>
## 10. 3D models and animation

### 10.1 One thing to unlearn first ✅

**There is no skeleton hierarchy in this engine.** Models are split into parts (4 to 20 depending on the model, 16 for both heroes). Every part is a rigid piece modelled around its *own* origin. For each frame, the animation gives each part its own absolute translation, rotation and scale, with no parent-child relationship and no matrix stack. Names like "forearm", "shin" are not stored anywhere. Any assembly of the character that uses fixed joint offsets is wrong; the pose comes only from an animation frame.

### 10.2 Model record ✅

```
+0x00   name[0x20]               ASCII (e.g. PC00, O01_01)
+0x20   u32 part offsets[...]    terminated by 0xFFFFFFFF
+0x128  u32 external vertex count
+0x12C  u32 part count
```

Part counts: heroes 16, PLAYER 15, TARGET 16, TO01 16, M01 9, M02 4, O01 13, O02 12, O03 13, O04 14, M07 17.

**Part header:** 9 × s16 = (0, bbox min xyz, bbox max xyz, polygon count, vertex count); `u32` at `+0x14` = absolute offset of the vertex block; elements start at `+0x18`. **Vertices** (the last `nvert × 16` bytes of the part): `s16 x, y, z`, `s16 nx, ny, nz` (4096 = 1.0), byte 12 = **external vertex export slot** (see 10.4), byte 13 = a flag whose meaning is unknown ❓, 2 more bytes.

Y points **down**. A hero standing on the ground has its feet at y ≈ 0 and the top of the hair at −138.

All 555 parts in PACK0 and PAKOMAKE parse, with the polygon count matching the header; the bounding box in the header equals the real extent of the vertices on all of them.

**Element (11 words):** flags | indices (4 bytes, `v2 == v3` = triangle) | four colour words `0x00BBGGRR`, the first carrying the GPU command in its top byte (`0x30/0x38` Gouraud untextured, `0x34/0x3C` textured, `+2` = semi-transparent; seen `30 32 34 36 38 3A 3C 3E`) | three UV words | an `E1/E2` word | terminator `0xFFFFFF0t`.

- Quads are stored in contour order (0-1-2-3). Check: with 0-1-3-2 only 1 of 257 hero quads is convex, with 0-1-2-3 all 257 are.
- Face winding is **not consistent**; the game draws without culling for many faces. Export as double-sided.
- Flags (low 16 bits): bit 2 (value 4) = a **decal packet** follows (80 bytes instead of 44), confirmed on 1156 elements with no exceptions in either direction; bits 0-1 = depth-sort method (0: average of 4 vertices, 1 and 2: min/max pair, 3: fixed `0x1FFB`) 🟡; bit 3 (8) = skip the back-face test 🟡; bit 9 (512) set on about half the elements, meaning unknown ❓.
- Decal: 9 words, no flags and no indices (reuses its parent's `idx`): 4 colours, 3 UV words, `E1/E2`, terminator. For heroes, bodies are plain Gouraud and the face, eyes and bandages are textured decals on top. In an exporter, nudge decals 0.5 unit along the normal to avoid z-fighting.
- Terminator `0xFFFFFF0t`: the low byte `t` is the **material record number** (0 = none); the game multiplies it by 4 as an index into a pointer table in the part. Found values: 0 (10033), 1 (779), 2 (98), 3 (29) on elements; 0, 1, 2, 4 on decals.
- Texture binding: a `(tpage, clut)` pair per polygon, in model space they are **small table indices**, not VRAM words (hero uses `tpage 0x0100`, `clut 0, 1, 2, 5`). §11 explains the real values.

### 10.3 Animation banks ✅

A bank sits in the record right after its model (hero 68 → 69, 70 → 71, 72 → 73, 74 → 75; also PLAYER, monsters, bosses, `TA*`, `TO*`, `USI_T1..4`). 55 banks, 533 table entries (485 distinct blocks), 6629 bone tracks, **66 290 channels, every one parsed exactly**.

```
+0x00   name
+0x1E   u8   bone (part) count; equals the model's part count in 45 of 45 pairs
+0x20   u32[256]  animation block offsets (0xFFFFFFFF = none; several may share a block)
+0x420  u32  offset of the collision shape table
+0x446  u16[256] per-animation flags (0x4000, 0x6000, 0x7000, 0xC000, 0xE000)
```

**Animation block** at `o`:

```
u16 n               index of last frame (frames = n + 1)
u16 0
u16 track[nb]       track offset from o, 0xFFFF = this part is not animated
4 × (u16 frame, u16 event)   event code is emitted when the frame counter hits `frame`
extra channel: pairs (u8 length, u8 value) covering n+1 frames; value = index into the collision shape table
tracks
```

**Track:** `u16 channel[10]` (offsets from `o`). Channels 0-2 translation x y z, 3-5 rotation rx ry rz (4096 = full turn), 6-8 scale (4096 = 1.0), 9 visibility.

**Channel 0 to 8 stream:** a word `w`; the value is `int16(w & 0xFFE0)`; `n = w & 0x1F` is how many frames the segment holds (minimum 1); if `n ≥ 3` the next `int16` is a **slope** added every frame (units of 1/8). Translation = value >> 5 (whole model units), rotation and scale = value as is. **Channel 9 (visibility):** bytes with bit 7 = flag and bits 0-6 = run length, padded to even. **1 = hidden.** Hero parts 14 and 15 are hidden in 8 of 8 animations of the event bank (those are the weapon and shield).

Checks: segment lengths add up to the frame count in every one of the 66 290 streams, and each stream ends exactly where the next starts, except for channel 9's 1-byte padding.

Some animations (two on the Flint bank and copies) have an extra channel that sums to 29 instead of 30; a harmless tooling bug in the original build.

**Hero banks.** Models 68, 70 and 74 are byte-identical in geometry, colours and UVs. Bank 69 has 108 animations, 71 has 100, 75 has 66. **Banks 73 (Alexia) and 75 are byte-identical**, with numbering shifted by 41: idle is animation 3 on Flint (15 frames, 141 high) and 44 on Alexia. Animation lengths for Flint: 0:30, 3:15, 4:29, 5:24, 6:19, 8:24, 9:18, 10:20, 32:60, 33:40, 37:50, ….

### 10.4 Posing a model ✅

Per part: `v_object = R · S · v + t`, then multiply by the object's base matrix.

- `S = diag(sx, sy, sz) / 4096`.
- `R = Rz(rz) · Ry(ry) · Rx(rx)` through the library's `RotMatrix`. This was confirmed by running the library routine in a small MIPS+GTE interpreter (matches to 0.0004 on five angle sets; the other five orders miss by 0.2 to 0.93). With an extra yaw argument the code builds `Rz · Rx · Ry` by hand instead; it is not known why.
- Pose code: `0x80027B90` / `0x80027C6C`; frame advance `0x80026394`; track reader `0x80025D5C`; start `0x80025F00`; output `0x800260F0`; events `0x8002628C`.

Not verified against a game picture: the unit of the per-frame slope, and the animation rate (30 fps is assumed).

### 10.5 External vertices ✅

Bosses and some models (O01 to O04, PLAYER, M07, M08, TA11, USI_T) have polygons whose indices are **≥ 60**. These read from a shared buffer filled by *other* parts.

- Rule: when the part rasterizer transforms a vertex whose byte 12 is non-zero, slot `k = byte − 60`, it also stores that transformed vertex into the shared external buffer at `s7 + 16·k` (`0x800133A4…0x800133B4`). Index ≥ 60 (`idx << 4 ≥ 960`) reads from that buffer, anything lower from the part's own buffer.
- Check: in 118 of 120 models, the set of exported byte-12 values is exactly `60 … 60 + n − 1`, with `n` = the word at `0x128`, each value having one source vertex. Two `O02_01` records have `n = 38` but only 36 sources (two slots unused).
- Practical result: the seams between parts (shoulders, hips, necks) are polygon strips between a part's own vertex ring and a ring that belongs to the neighbouring part, positioned by the neighbour's pose. An exporter must follow the byte-12 table. Guessing "closest vertex of the neighbour" gets it right only about half the time.

### 10.6 Collision and hit shapes ✅

Table at `+0x420`: `u32[N]` pointers. Element 0 has no tag: three s16 (w, h, d), the default body box. Flint 25×138×25, bosses 30×200×30. Other elements are tag-lists until zero: **tag 1 = box (centre xyz, size whd), 14 bytes; tag 2 = point, 8 bytes; tag 3 = sphere (centre xyz + radius), 10 bytes.** 59 lists in 55 banks with only these tags (36 boxes, 16 points, 47 spheres), and all 542 extra-channel indices land in the table. Flint's attack frames add a sphere of radius 55 to 60 about 135 units in front (−Z), which looks like the hit zone 🟡.

### 10.7 The renderer ✅

AL22 (`0x80012E70`): for each part there is an 80-byte record (flags, part number at `+2`, model pointer `+0x3C`, external buffer `+0x48`, material table `+0x44`, matrix `+0x20..0x30`, translation `+0x34..0x38`). Vertices are rotated with GTE `MVMVA 0x4A480012`, normals the same way, then `RTPS`. The routine that sets up an object calls `0x800B2040` / `0x800B2708` (these only build the part matrices through `0x80027C6C`; they do not draw).

### 10.8 Weapon and shield

Part 14 (29 vertices, blade along +Z from −21 to 102) and part 15 (18 vertices, thin plate) on heroes match, in shape and local frame, the five `PC00_KEN` sword variants (23/29/27/29/37 vertices) and the five `PC00_TATE` shield variants (18/18/18/18/27). They are almost certainly swapped in by the game, but what selects the variant has not been found ❓.

---

<a id="en-11"></a>
## 11. Model materials ✅

A model file only contains small numbers (`tpage 0..n`, `clut 0..n`). The real VRAM address is decided when the model is **set up for use**, by `0x80024DC4(model, tpageTable, clutTable, du, dv)`.

For each polygon with material `t = 0`:

```
tpage_word = (T[(word & 0xFF) + startT] & 0xFF9F) | ((word >> 3) & 0x60)
clut_word  =  C[clut + startC]
u += du;  v += dv        (byte arithmetic, per vertex, decals too)
```

The low byte of the model's tpage picks a table row; bits 8-9 become the blend mode (so model `tp0100` = "row 0, mode 1"). After that `t` is multiplied by 4.

For `t ≠ 0`, the drawing code (AL22 `0x80013C4C`) computes the same lookups at draw time, with two shifts from the material record (s16 at offsets 14 and 16). **Where these records are stored in the file is not known**, so the exporter treats such polygons with zero shifts ❓. They look like state channels (flash, freeze, …).

### 11.1 Which tables

| Case | tpage table | clut table | du, dv |
|---|---|---|---|
| Map models, class normal | `0x80052480`: `0x15,0x16,0x17,0x18` = pages (320..512, 256) 4-bit | `0x80052488`: 4 palettes × 16 rows, `0x7438…`= x 896.., y 464..479 | from type record |
| Map models, class 14 ("D" objects) | `0x80052588`: `0x18, 0` | `0x8005258C`: `0x7CB0 + i` | du 0, dv 128 |
| Map models, class 4 (bosses) | `0x800522D0`: `0x85,0x87,0x89,0x8B,0x8D` = 8-bit pages from slot 0 | `0x800522E8`: `0x7A30…` rows y = 488.. | 0, 0 |
| Heroes (PACK0 68 to 75) | `0x8005244C`: `0x0019` = (576, 256) 4-bit | `0x80052460`: `0x7D30 + i` = (768 + 16i, 500) | 0, 0 |
| Effect records 39 to 67 | first the hero tables (39, 40), then GAME00 tables `0x800F056C / 0x800F050C` | | |
| EVENT00 objects | tpage `0x800D27C6 + 2v`, clut `0x800E9410 + 16v` (filled at run time) | | |

This is why a single model, for example `T02`, shows up on 159 maps with 118 different (tpage, clut, du, dv) combinations: **colouring belongs to the placement/type record on the map, not to the model.**

### 11.2 Verification

- RAM dump of the running game on map 213: the game rewrote polygon words in memory; the rule matched **94 of 94** `t = 0` elements byte for byte (the other 20 with `t ≠ 0` had UV shifted, clut/tpage untouched).
- All 3260 models on all maps, 96 565 textured polygons and decals: every UV rectangle (after shift) falls inside the VRAM that the map itself provides, and every referenced palette entry exists. Zero misses.
- An exported page + palette image was bit-identical to the real VRAM render in all four cases tried.
- Boss (class 4) and "D" objects (class 14) are confirmed by code and coverage but not by a game screenshot.

### 11.3 Hero texture page 🟡

The hero page at (576, 256) is loaded in GAME00 around `0x800AFD70` from **PACK0 record `32 + n`**, `n = (s16 at 0x8006746E) − 39`, which gives seven variants (records 32 to 38). Those seven only differ in one UV region (what the `clut 5` decals cover), so they look like equipment or look variants. Who writes `0x8006746E` is not known ❓.

---

<a id="en-12"></a>
## 12. Enemies, AI bytecode, bosses, damage

### 12.1 Stat table ✅

`0x800A6924(id)` returns a 14-byte record:

| id | Base address |
|---|---|
| < 100 (A) | `0x800EEF48 + 14·id` |
| 100 to 129 (B) | `0x800EEF58 + 14·id` |
| 400 to 499 (T) | `0x800EE0A4 + 14·id` |
| 600 to 759 (E) | `0x800EDB3C + 14·id` |
| other | shared empty record `0x8012CC18` |

Record: `+0` element/class (0 to 6), `+1` defence, `+2` base attack, `+3` flags (`0x40` damage scales with level, `0x80` double chance, `0x04` halves damage taken), **`+4` u16 HP**, then three `(condition key, damage)` pairs (`+6/+7`, `+8/+9`, `+10/+11`; `255` = none). The spawn routine (`0x800AE754`) copies HP to `+0x38/+0x3A`. HP matched in-game measurement on all 40+ objects checked. 213 records exist.

Samples: A01 15, A02 50, A17 220, A54 280, A58 300; bosses: B01 200, B02 600, B11 150, B12 250, B13 200, B17 300, B18 400, B20 1000, B21 880, B22 200, B23 600, B25 1500, B26 300; E147 500, E146 360, **E148 4320** (the highest in the game). B03 and B24 have empty records; on arena 408 (B24) E148 and eight E146 sit next to it, so HP is probably held by the parts 🟡. The player has 200 HP.

### 12.2 Damage formula 🟡

From code `0x800A8478…0x800A86C8`:

1. Elements: `0x800A8314` compares classes on a ring 1 to 4 (±1 / ±3 give advantage / disadvantage). The player's class comes from the weapon.
2. `base = attack` (or the pair value if the key matches), scaled by `0x800A6BE0` (flag `0x40`: × (level + 1); flag `0x80`: ×2 on a roll), minus defence, floor 0, halved against flag `0x04`.
3. Element multiplier ×16 / ×8 / ×10 then ÷10 (1.6 / 0.8 / 1.0). If the player attacks and `0x8012CAD8` > 0: ×(v + 10) / 10. Add random 0 to 3.
4. Difficulty: half-word `0x80066BD4` is `0` on a fresh game, in which case the player deals half and takes double; otherwise normal. Cap 999, player minimum 1.

The combo level `comblvl` (word `0x80067726`, 0 to 4) is a **combo chain length limit**: `0x800A2334` compares it with the current chain step (`0x8012CAD8`), and each step adds +10 % in the formula above. Level goes up with points from item-68 crystals (command 225): threshold = `3·(L+1)·(L+2)` points.

### 12.3 AI bytecode ✅

Regular enemies (A1 to A99, class 2) do not use map scripts. Their behaviour is a **separate bytecode** in GAME00 at `0x800F5A7C` (about 17.5 KB). Layout: 4-byte empty program, a table of **140 half-words** (type → program offset, `0xFFFF` = none), then programs. Interpreter `0x800EA7D8`: a byte ≥ `0x80` is a command (handler table `0x800F4828`, index = byte − `0x80`, 128 entries; `0x800EA870` = unknown with debug print); lower bytes are data. A handler returning non-zero ends the frame.

Object registers: `+0x250` entry, `+0x252` restart point, `+0x254/+0x256` return stack (2 levels), `+0x258` PC, `+0x25A` condition register, `+0x24C` timer (decremented each frame), `+0x25C..+0x25F` child object slots (0/255 = none), `+0x24D` mode (high nibble set by `8F`, low nibble used by `BF`).

**Main opcodes:** `F8` goto, `F9` call, `FF` return, `FD` yield, `FC` restart, `FA` 4-way switch on zone, `E0/E1` `jeq/jne B,W`, `F4/F5` jnz/jz, `F6/F7` jge/jlt, `F0` wait for any event mask, `F1` wait for all, `80/81/82` animation (82 = default from the table), `90 to 93` timers (93 = base + random), `95` sound, `B0/B1/B2` stop/walk/run, `A0..A7` turns (toward player, ±, 180°), `BA/BB` velocity vector (fixed / random), `D1` spawn child, `D2/D3` child commands, `DD` blend two skeleton poses (smooth transitions), `E9` random chance B/256, `EB` random & B, `AC/AD/E4/E5/E6` point-target operations, `BF/EE` variable-length branches (word list until `0000`), **`C0 BB` query** (about 60 subqueries: arithmetic, distance to nearest non-enemy `0x84`, distance to player `0x90`, "player is attacking" `0xC0`, …). RNG: LCG `0x41C64E6D + 12345`, seed at `0x8007D770`.

**Events** (word `0x80148E4A`, recomputed each frame by `0x800E5A90`): `0x8000` animation finished, `0x4000` timer expired, `0x0100` turn finished, `0x0080/0x0040/0x0020` vertical speed positive/negative/zero, **`0x0010/0x0008/0x0004/0x0002` = player in near / middle / far zone / outside**.

**Vision table** (`0x800F4C08`, 36 bytes per type A1..A99): squared distance `(dx >> 12)² + (dz >> 12)²` compared with the parameter squared, plus a cone angle (degrees → 0..511 with 360° = 512). Example A1: zones 120 / 300 / 600 with a 180° cone; A2: 200 / 400 / 900 with a narrower far zone. Other fields: walk/run speed (×256 into `+0x7C`), timer base and random part (`+18/+19`), sound base `1001 + 10·type`, extra sound `+34` (mostly 1003), vertical reach ×100.

**Example (A1):** four branches by zone: near, a lunge with animation `0x52`; middle, a pause then one of several random moves (sidesteps with turns of ±128 or 180°, a dash, a strike); far, a slow approach; out of range, idle.

Boss programs: slot index ≠ id for ids ≥ 100: `120 → 120`, `710 → 121`, `711 → 122`, `712 → 123`, **`112 → 124`**, `715 → 125`, `716 → 126`, **`122 → 127`**, `720..731 → 128..139`. So **B12 (250 HP), B20 (1000) and B22 (200) are bytecode bosses**, not PACK1 modules (their PACK1 records are empty 4-byte stubs). B20 builds fights on distance bands and spawns 645 / 710 / 711 / 712; B22 spawns 720 / 721 / 723 (chain 723 → 724 → 725 → 726).

### 12.4 Boss modules in PACK1 ✅

Half-word 116 in a map header = PACK1 record with that arena's boss code (loaded at `0x801493B0`, 12 KB limit, loader `0x800AFCB8`). Only 15 maps have one:

| Map | Boss | PACK1 | Size |
|---|---|---|---|
| 337 gy028 | B01 | 1 | 8424 |
| 288 di022 | B02 | 2 | 7364 |
| 303 di021 | B03 | 3 | 4 (empty) |
| 401 hi020 | B11 | 11 | 7256 |
| 154 fc014 | B12 | 12 | 4 |
| 400 pmwa | B13 | 13 | 4596 |
| 402 yk100 | B17 | 17 | 8008 |
| 403 pd100 | B18 | 18 | 7716 |
| 406 st100 | B20 | 20 | 4 |
| 405 pz100 | B21 | 21 | 7792 |
| 404 ki039 | B22 | 22 | 4 |
| 407 st101 | B23 | 23 | 5356 |
| 408 st102 | B24 | 24 | 9280 |
| 409 st103 | B25 | 25 | 11 308 |
| 48 aw017 | B26 | 26 | 6984 |

Module layout: first word = a small id, then a table of 17 to 36 event handler pointers (many repeats), then code, and at the end a **state table** (one function pointer per state: B01 7, B02 6, B11 7, B13 4, B17 4, B18 4, B21 6, B23 4, B24 6, B25 8 + 3 helpers, B26 5). State word `+0x248`, state timer `+0x24C`, sub-mode `+0x250`. Sound ids: `10000 + 100·boss + n` (B25: 12501 to 12518).

**B01** state 0 chooses by squared planar distance: ≤ 90 000 (≈ 300 units) → state 1 (close attacks), ≤ 202 500 (≈ 450) → state 2, otherwise approaches (anim 151 above 360 000, anim 133 above 202 500). A counter starting at 120 sends it to state 3 (special attack) when it reaches zero.

**B25 final boss** (map 409, 1500 HP head with horns, arms E149/E150 at 0 HP that take no damage, three floating E147 targets at 500 HP each). The arena script (stream `@0xA0`) sets flags 1, 2, 3 one after another, apparently as the three targets die 🟡. The boss code (phase variable `0x8014BFA2`, 8 phases, table `0x8014BD90`) reads them in phase 0: each flag is cleared and answered by phase 2 (animation 136, a projectile E152 after 120 frames); the third also sets flag 4, after which `0x8014BFD8` becomes 1, and then the head slides along Z from −4 505 600 to about −2 720 000 in 32 frames and `0x8014BFA0` = 1 ("head out"). Phase 7 (death) pulls it back and picks phase 4 or 5 (attacks). Verified by setting `0x8014BFD8 = 1` by hand. Whether the exposed head is vulnerable, what phases 4 and 5 do beyond animation ids, and the damage values are not established. A player who does nothing loses about 0.33 HP per frame and dies in roughly 600 frames; after death the game goes to map 498.

---

<a id="en-13"></a>
## 13. Player state, items, shops, saves

### 13.1 State block ✅

The save is a plain RAM image. `0x80066BC8` for `0xC88` bytes is "everything that persists". Field map (offset from `0x80066BC8`):

| Offset | Meaning |
|---|---|
| `+0x00` u16 | save counter (cap 999), written on save |
| `+0x02` u16 | "place index" = position of the current map in a 40-entry table at `0x800F2CD8`; used for the slot's label |
| `+0x04/06/08` | hours, minutes, frames (60 per second) of play time; `+0x0A` pauses it |
| `+0x0C` | the half-word `0x80066BD4` mentioned in §12.2 |
| `+0x40…+0x3A2` | story flags (6928 bits) |
| `+0x844` | item counts (100 bytes, index = item id) |
| `+0x8EC` u32 | `1` on a new game |
| `+0x90C…+0xB0B` | 16 "bags" × 0x20 bytes |
| `+0xB0C` u16 | mask of passed nodes in the overworld (§18) |
| `+0xB10` u16 | `worldcnt` (0 to 24) |
| `+0xB14` u16 | map to return to / start map (213 on new game) |
| `+0xB16` | cutscene return marker |
| `+0xB1C…B22` | 165, 177, 214, 157 on new game (meaning unknown) |
| `+0xB3C` u32 | money (cap 999 999) |
| `+0xB40` u32 | **medals** (cap 99 999) |
| `+0xB44` u32 | level points |
| `+0xB48…+0xB4E` | max HP, HP, max of the second gauge, the second gauge (default 200 each) |
| `+0xB50/54/58` | player x, y, z as s32, 20.12 fixed point |
| `+0xB5C` u16 | facing (12-bit angle) |
| `+0xB5E` u16 | combo level |
| `+0xB60` | player state number |
| `+0xB66…+0xB76` | 9 × u16 button masks (defaults `0x80,0x20,0x40,0x10,0x01,0x02,0x08,0x04,0x100`) |
| `+0xB78…+0xBEF` | shop stock (15 × 8 bytes) |
| `+0xBF8` u32 | mini-game 1 best time (default `0xEA24` = 999.0 s in frames) |
| `+0xBFC / +0xC04` | mini-game 2 / 3 high scores |

The game also keeps a second copy of the block at `0x80065F38`.

### 13.2 Memory card format ✅

Files are named `BASLUS-01017NN` (one block, 8192 bytes). Block layout: `SC`, icon flag `0x11`, 1 block, a 64-byte Shift-JIS title (play time and slot digits as full-width characters), palette at `+0x60`, icon at `+0x80`, **a copy of the 3208-byte RAM image at `+0x100`**, and then a **u16 checksum at `+3464` = sum of u16 over the first 3464 bytes**, modulo 65536 (`0x800C9690`). The remaining ≈ 118 bytes up to the written length are leftover buffer and are not covered. Verified on all five slots of a real card (counter, place index, time fields, HP all match the screen).

### 13.3 Items ✅

- Counts: `0x8006740C` (100 bytes); new game zeros it and sets items 31 and 39 to 1 via table `0x80056314`.
- Definition table `0x80055DB4`: 86 records × 16 bytes. Fields: `+0` group, `+2` flags (`0x40` has a use-effect, `0x0800` = group 31 to 38 weapons, `0x1000` = group 39 to 46 armour, `0x2B` consumable), `+4` value, `+8` effect type, `+9` subtype, `+0xA`, `+0xC` magnitudes.
- Use routine `0x80023D78(id)`. Effect types: **1** heal HP, **2** heal the second gauge, **3** raise max HP, **4** raise max of second gauge, **5** money. 0x34/0x36/0x37 heal 100/300/600; 0x39/0x3A restore 200/500 of the second gauge; 0x05/0x3E/0x3F/0x45 give +25 max HP; 0x0B/0x42 give +100 to the second gauge max; 0x50 to 0x55 give money 100/200/500/5/10/20. Types 6 to 10 not decoded.
- 16 starting bags (`0x800674D4`, 0x20 bytes): `u16 count` then ids, then flags (ids 0x34, 0x36, 0x37, 0x39, 0x3A and 0x20 to 0x2A, 0x45, 0x0B, 0x05).
- **Crystals (item 68)** are turned into points by command 225; thresholds for combo levels 0, 1, 2, 3 are 6, 18, 36, 60 points.
- Items never given by scripts, shops or starting bags: 2, 3, 7, 8, 13, 19, 20, 37, 38, 45 to 50, 59, 60, 63, 64, 71, 73, 75 to 77, 80 to 85. Some of them (80 to 85, money pickups) are probably picked up from placed objects instead.

### 13.4 Shops ✅

The 15-slot table is a **shop stock**: `+0` flags (high nibble = shelf position, bit 0 = sold/special, bit 1 = new), `+1` item id, `+2` available, `+4` u16 price, `+6` u16 second number. Buying (GAME00 `0x800E5160`): check money, check stock against a cap (9 for items with first u16 of 0 or 1, otherwise 99), then `sound 2`, add item, take money; otherwise sound 63. First group (8 shelves, from `0x80056424`): items 23 to 26 at 2500 / 1500 / 1000 / 2000, item 4 at 30 000, 11 at 5000, 74 at 9000, 56 at 2000. Second group (7 shelves, `0x80056464`): consumables 52 / 54 / 55 / 22 / 56 at 10 / 100 / 250 / 300 / 1500, items 70 and 72 at 2000 / 4000 and 2500 / 5000. The table is saved, so "sold" persists.

### 13.5 Medals ✅

The second counter is **medals**, used by the three arcade games. Command 229 adds medals. The medal merchant on map 414 sells at **5 money per medal**: 2000 → 400, 500 → 100, 100 → 20.

### 13.6 Maps that give lots of money 🟡

Maps 485 and 487 repeatedly hand out and take back weapons/armour and big sums (487 gives 100 000 on entry, 485 gives 30 000). Map 502 gives and takes 30 000. They look like developer test rooms; the 487 payout was confirmed by jumping there directly.

---

<a id="en-14"></a>
## 14. Audio

The game has three separate audio systems.

### 14.1 CD-XA (cutscenes and voices) ✅

- `STEREO.XA`: Level B stereo, 37 800 Hz, 4-bit ADPCM, **8 interleaved channels** (sector step 8).
- `MONO.XA`: Level B mono, 37 800 Hz, **16 channels** (step 16).
- Two master tables in the main EXE: `0x80056EF8` = **125 music tracks**, `0x800576C8` = **52 voice clips**. Record (16 bytes): `start sector, channel (0..7 or 0..15), sector count, name pointer`. The sectors of a track are `file LBA + start + channel + i × step`. The channel number in each sector's sub-header agrees on 125 of 125 and 52 of 52.
- Names are in the EXE: `MX_12` (5950 sectors ≈ 5.3 min), `MX_11S`, `001AS/BS/CS`, `030AS/BS`, `JX_04A`, `JX_07A`, `JX_09`, `JX_10`, `JX_11`, `SAL01/02`, `SEI01/02`, `DRI01/02`, `PIX01/02`, `CARDV01..21` (card-game dealer voices), and story clips like `002AM`, `021AM`, `040BM`.
- **This is cutscene and event audio, not map background music.** The XA names live inside the PAKVCAST records for their scene (number prefix = scene, suffix `S` stereo music, `M` voice). `MX_12` appears only in a PACK1 module (probably the finale); `023B` is referenced nowhere.
- Playback by name: `0x80030430(const char*)` (search `0x80030254`). Callers include a debug key, EVENT00 (name from the script stream), and GAME00. XA playback from scene commands (`xaplay`, `xaseek`) is a stub in the retail EVENT00.
- ADPCM decode is the standard CD-XA filter set with `f0 = [0, 60, 115, 98]` and `f1 = [0, 0, −52, −55]`.

### 14.2 Background music: sequencer ✅

Normal in-game music is not XA. It is a **MIDI-like sequencer** with `SSsq` (sequence), `SShd` (instrument header) and **SPU-ADPCM samples**. There are 30 tracks. The map → track table is at `0x80056C20`. PACK0 holds each track as a triple: **record 581 + 3·(k−1) = SSsq, +1 = SShd, +2 = ADPCM.** Unused sequences (no table entry): 563, 567, 572, 584, 626, 641, 647, 650, 656, 677.

- `SSsq` header: dword `0x01E00064`, `N` (48 to 128), 0, magic `SSsq`; then `N` 16-byte records.
- Event handler at `0x800489F8` (note off/on, aftertouch, control change, program change, pitch bend, system; meta `0x2F` = end), track header parser `0x80025D5C` (10 tracks × 20 bytes). The sound driver is the range `0x8004B000…0x80050000`; `SSsq` check at `0x8004F03C`.
- Pitch = `(note − centre) + fine/16 + bend/64·2` semitones; volume = `velocity × tone volume × volume`; pan from tone + channel + program; ADSR from the tone's registers. Loop controller CC 99: value `0x14` marks the loop start and `0x1E` the end.
- Cross-check on track 34 against a running game: pitch correlation 0.89, timing ratio 1.000, loop period 64.65 s.
- Not verified: pan by program, percussion programs, exact sustain behaviour, and the sound as judged by a human ear.

### 14.3 Sound effects (slot 7) ✅

Bank loader `0x8004EBE0(adpcm, header, size)` checks 4-alignment, size ≤ `0x80000`, multiple of 16 and the `SShd` magic at `header + 12`; it registers the bank in the 127-slot table `0x80082830` and uploads to the SPU (`0x8004E2E0`). Play by id: `0x8002DF78`; volume table of 64 entries × 10 bytes at `0x8006ADF0`. Script ids carry an offset of 100. Header after `SShd`: `+0` length, `+4` ADPCM size, `+12` magic, `+20` the number 128, then mini-sequences (9 bytes per sound ending `… ff 2f 00`) and a table of 16-byte records at 0x100 steps. That tone table is **not decoded** ❓.

Most shared sets: id 3011 (4432 bytes, 194 maps), 1001 to 1007 (24 320 bytes, 187 maps), 3021, 3071/4611, 2551, 2461, 2681, 2171 to 2173. Enemy sound ids follow `1001 + 10·type`; boss ids follow `10000 + 100·boss + n`.

The main EXE also has a fixed system bank (`0x8006B1A0`, 4112 bytes of ADPCM) and a 14-page debug audio screen (`0x8002E474`).

---

<a id="en-15"></a>
## 15. Mini-games

All three cost **10 medals per play** and live in GAME01 to GAME03 (modules 3 to 5). Entered from maps 494 / 493 / 495 through command 198.

### 15.1 GAME01 (balloon/ball run) 🟡

Collect everything in each stage within a time limit; stages go in order and the reward grows with them. State word `0x8008F4B4`: 7 intro/input, 8 "start?" question, 9 pay (medals `[0x80067708]` ≥ price `[0x8008C61C] = 10`, deduct with `0x8001D338`), 10 play, 13 pause, 6 after, 12 not-enough-medals. Texts 7301 to 7311.

Stage table `0x8008C4AC`: 12 rows of (seconds, pool): (40,20) (30,20) (30,20) (50,20) (30,30) (40,30) (40,30) (40,30) (30,30) (40,30) (30,30) (50,30). Time left = `seconds × 60 − frame counter`. The pool drains 3 per frame into the reward word `0x800A6C94`; total across 12 stages = 320. Best time lives in the save at `+0xBF8` (starts at 999.0 s). Not verified: how medals are finally paid out. In test runs the game only reached the "start?" screen.

### 15.2 GAME02 (shooter, 50 stages) ✅/🟡

Screen 256×~160. Stage data: `u16 N`, then `N × (u16 kind 1..9, u16 x, u16 y)`; table of 51 pointers at `0x800AE688` (stages 0 and 1 share data). Score cap 9 999 999. After a stage, remaining time + HP + stage bonus are added (stage bonus around 1000 + 100·n; every fifth stage is big: 15 000, 25 000, 35 000, … up to 65 000 on stage 50).

Per-kind tables: points `0x8008BB30` (kinds 1 to 3: 3, kind 4: 80, 5 to 15: 10), toughness `0x8008BB70` (kind 4: 8 hits, rest 1), size `0x8008BBB4` (kind 4: 20, rest 12). Medals at the end = score ÷ 1000.

Measured on the running game (speed unit 4096 = 1 px/frame): kind 1 walks to the player at 0.25 px/frame; kind 2/3/8/9 wander at 1.0 px/frame and change heading about every 64 frames; kind 4 is stationary (a wall of 57 on stage 20); kind 5 converges on the player and stops in a clump; kind 6 bursts out at about 3.1 px/frame and decays ×127/128 per frame; kind 7 fans out of a row at 1.0 px/frame. Player shots use 7 rows of 24 bytes at `0x8008BA6C` (triple shots at 120°). Player starts at (128, 80), three tries per run, one point of "FLINT" lost per try.

Not decoded: the turn rule for kinds 2/3/8/9, how kind 4 shoots, field `+0x28`.

### 15.3 GAME03 (grid arcade) 🟡

A tile-based arcade with 16-pixel tiles: a field of stumps, plants and one chaser. 12 stage rows of 28 bytes at `0x8008A970`: `w0` kind 0 to 3, `w1,w2` field size in cells of 48 px (3×3 on all 12 rows), then **two start cells** `(w3,w4)` and `(w5,w6)` of moving objects (−1 = none). Objects live in 32-byte records at `+0x2DB4` of the state struct, advanced every frame by `0x80089828`. Medals = score ÷ 10. Ladder tables: `0, 100, 200 … 1100` and `(100,1) (200,2) … (1100,11)`. Player starts at (112, 80) and moves in steps of 16. Rules are only partly understood.

---

<a id="en-16"></a>
## 16. Cutscenes (EVENT00, PAKVCAST)

### 16.1 Event table ✅

EVENT00 table `0x800B00DC`, 36-byte records, ends with id 9999. **104 events**: 75 numbered 101 to 278 (mostly 101 to 189), 24 "B" variants numbered +400 (501, 502, 506, …), and 5 odd ones (740 to 754, 954). A record: event number, four PAKVCAST record indices `[textures, scene data, sound effects, music or −1]`, then four path strings (`\vcast\vcast_NNN\vcasttex.NNN`, `vcast.NNN`, `vcast_NNN.se`, `vcast_NNN.bgm`). The table references 286 of the 311 PAKVCAST records. Lookup: `0x800880B0`. New game starts at 101; the debug page VCast and script command 192 both use the same entry.

### 16.2 Record contents

A record in the pack is a container: first word = header length, then block offsets.

- **Texture record:** `01 MOD` blocks (§4.4) five at a time, plus palette sets (`00 MOD`).
- **Scene data record:** a table of u32 offsets (68, 476, 1328, …), then sub-blocks of two kinds: **object descriptions** (they contain model names like `wing2`, `neji00`) and **command streams**. Also `FRITTO` and `SENTEI` blocks: **3D scene models**. Block = a directory of offsets stepping 540 bytes, each "part" = 540 bytes: six s16 (position/rotation), a 3×3 matrix at `+0x14` (4096 = 1.0), a 16-byte name at `+0x34`, then ready-made GPU packets (four indices, colours with `0x3E`, UVs, palette/page words). Vertex list, animation format and texture-to-model links are not decoded ❓.
- **Sound record:** one block at `0x44` with `SShd`.

The magic strings (`MOD`, `FRITTO`, `SShd`) are not compared anywhere in the code; the format is defined by block order.

### 16.3 Scene command language ✅/🟡

Interpreter in EVENT00 (loop at `0x80089544`). Command = `u16 frame` (runs when the scene frame counter `[0x800E980C]` reaches it) + `u16 code (0..76)` + arguments. Jump table `0x80084B5C` (77 words); out of range prints "BlockID Error". Scene state: 16 slots of 1728 bytes at `0x800D27F8`. Stream ends with **code 22**.

Fitting by data gave **134 streams and 3165 commands**. Lengths (bytes): 1→12, 3→6, 5→6, 6→10, 7→10, 10→8, 13→10, 16→12, 25→12, 26→8, 27→6, 28→8, 30→10, 31→14, 49→10, 53→10, 55→8, 56→16, 57→6, 61→10, 68→10, 70→8, 71→10, 73→6, 76→6; codes 38/39/41/43/65/66 carry an inline insert: `12 + even(u16 @10)` (38, 41, 43), `6 + even(u16 @4)` (39, 65), `8 + even(u16 @6)` (66).

Meaning, from code: **6** = camera path (curve block by index, duration in frames, bit 15 = flag; evaluated with the spline routine `0x80029884`, result ×4096 into the slot at `+0x6C..+0x74`); **7** = the same for the look-at point; **1** = set global scene parameters, sub-code in the first argument (2: three globals, 1: a signed pair, 8, 10/11 (set/reset), 6 (three angles → a direction vector, probably light), 3); **10** = play skeleton animation (loop in bit 0 of the second arg); **13** = choose current background/model (110 uses); **26** = switch an object's data block (alternates 56 and 57, blinking); **30/31** = fades and flashes 🟡; **49** = sound; codes 47/48/60 = stubs with debug strings (`xaplay`, `xaseek`, `Depart`).

About 35 of the 77 codes are still unnamed, and ten scenes (ids 119, 147, 602, 601, 604, 228, 671, 272, 672, 277) did not split into streams with these lengths.

---

<a id="en-17"></a>
## 17. Developer leftovers

Nearly all of the developer debug code is still in the retail disc, kept off by one word and one missing file.

### 17.1 The lock ✅

Word `0x800608D4` is written once at `0x8001A9F8` as `lui v0,0 ; addiu v0,v0,1` (value 1). It is only read in GAME00 at `0x80089A0C` (the HUD/menu call from the main loop `0x8008968C`, only if the word is `0`) and `0x8008A100` (pause/step). Patch the instruction at `0x8001A9F4` from `0x24420001` to `0x24420000` and the debug layer wakes up (file offset `0x41F4` in `SLUS_010.17`).

### 17.2 `a2profile.ini` ✅

Parser `0x80019BB8…0x8001A060`. It opens `a2profile.ini` **without a `cdrom:` prefix**; on the retail console the open fails and every flag stays 0. If the file opens and a key is missing, the flag turns ON. Format: `key=number`.

| Key | Destination | Meaning |
|---|---|---|
| `itoinf` | `0x80060828` | HUD enabled |
| `jmp` | `0x80060829` | DebugJmp page mode |
| `itodebug` | `0x8006082A` | menu enabled |
| `memhd` | `0x8006082B` | Save/Load device: "HD" or "MEM" |
| `vcast` | `0x8006082C` | cutscene flag |
| `childno` | `0x8006082D` | start item of the start menu |
| `dmenu` | `0x8006082E` | show the module start menu |
| `xa` | `0x8006082F` | XA test |
| `datacd` | `0x80060830` | read data from CD vs dev host |
| `bgmvol`, `sevol` | `0x80060022`, `0x80060024` | volumes |
| `startmap` | `0x800676DC` | start map (default 213) |
| `worldcnt` | `0x800676D8` | overworld step |
| `item=a,b,c` | | give items by id (effects apply immediately) |
| `hp=N`, `ep=N` | `+0xB48/B4A`, `+0xB4C/B4E` | HP and second gauge |
| `comblvl=N` | `0x80067726` | combo level (default 4 when absent) |

(`childno` takes a number but the *start map* is `startmap`.)

### 17.3 Second-controller HUD and menu ✅ code / 🟡 behaviour

Controller structs `0x80061EF8` (pad 1) and `0x80061F10` (pad 2), filled by `0x8001BF30`; the debug code reads **pad 2**. `SELECT` toggles the HUD and `START` the menu. The HUD prints `CPU%03d GPU%03d %03d`, `OBJ%03d`, player position, `ATB%04x`, and "Prim Over" when the primitive buffer is nearly full. The menu has 13 pages (up/down), `Square` runs:

0 EvtMsg · 1 VCast · 2 Camera Control · 3 Camera Mode (Normal/Yoko/Sita) · 4 Light (loads `bin\light.tmp`) · 5 Snap Shot (writes `d:\a2\bin\a2.tim`) · 6 Touch & Atk · 7 Save · 8 Load · 9 Audio (the 14-page audio screen) · 10 DebugJmp · **11 Flg** (4-digit flag number, Square toggles it through `0x8001D454`) · **12 Map Change** (3-digit map number through `0x80095520`).

Pause/step: hold `R1` on pad 2 to freeze and redraw, `R2` advances one frame. The button bit layout is assumed standard libpad.

### 17.4 Start menu (`dmenu`) 🟡

`0x800888C0` on pad 1: up/down choose a module, `Cross` confirms, the typed number lands in `0x800608CC` and becomes the module argument (for item 1, event no. should be `0101…0274`). The list prints the file names from the module table. Patch for "always on": GAME00 `0x80089714`, `0x9043082E` → `0x24030001`.

---

<a id="en-18"></a>
## 18. Overworld graph ✅/🟡

The object handled by `0x800C3040` (23 states, object `+0x64`) is a **cursor on a 24-node map**, not an ordinary door. It reads the d-pad, looks up a 4-byte neighbour record (0/4/8/12 by direction) in a 16-byte node row (pointer table `0x800F263C`, data `0x800F233C`, second node set `0x800F24BC`): `b0` neighbour node (`0xFF` = no path; open only if its bit is set in the mask `+0xB0C`), `b1` direction along the path, `b2` path number (offsets in `0x800F21E0`, 23 paths, spline evaluated by `0x80029884` from a buffer at `0x801DD1B0`), `b3` **exit index** (→ `obj+0x70`).

Exit index (0 to 37) goes through `0x800954B8`, which fills the map-change struct `0x8012B748` from two tables: area number from `0x800EE600` (`t1`) and (map, entry) from `0x800EE64C` (`t2`). For all 33 real edges, `t1[exit]` equals the neighbour node number. Map/entry pairs 0 to 37:
(20,1) (20,1) (30,1) (32,1) (12,1) (20,1) (43,1) (42,1) (50,1) (50,2) (50,3) (334,1) (295,1) (295,2) (296,1) (39,1) (20,1) (298,1) (389,1) (294,1) (425,1) (207,1) (422,1) (20,1) (203,1) (20,1) (216,1) (483,1) (415,1) (221,1) (410,1) (285,1) (490,1) (140,1) (214,1) (218,1) (340,1) (2,0).

The second node set only fills nodes 8 to 12 (and a link to 19). Note the usual map-border transitions are not done through this table, since it has only 38 entries; how ordinary neighbouring-map exits are set up is not resolved ❓. Door/exit objects run through `0x800C4BA0(number, mode)`, which sets bit `1 << number` in `+0xB0C`; `worldcnt ≥ 25` resets the chain. `worldcnt` is therefore the step in a 25-stage chain, `+0xB0C` the passed-nodes mask. Two tables at `0x800F2210` (26 × 4 bytes) and `0x800F2274` (8 bytes per step) describe the nodes and the flag masks.

The same loader sets the start position: from a save it copies map `+0xB14`, position `+0xB50/54/58`, facing `+0xB5C`, `+0xB64`; for a new game it takes the map in `0x800676DC`.

---

<a id="en-19"></a>
## 19. Open questions

Things still unknown, so nobody has to guess whether they are covered:

- Who writes `0x8006746E` (hero page variant, 7 values) and what it means; what picks the sword/shield variant (`PC00_KEN` / `PC00_TATE`).
- Where material records for `t ≠ 0` live (≈ 40 % of textured polygons); flag bit 9; byte 13 of vertices; the "mixed winding" cause.
- Meaning of slots 1, 12 and 13 (sky, dome, lighting are inferences); words 126 to 139 of the slot-11 table; the tone table in `SShd`.
- About 140 script commands are unnamed; the remaining 525 streams that do not parse; commands 147, 177, 179, 186 have no known length.
- Remaining bytecode AI opcodes (40+ unnamed), several `C0` subqueries; boss B24 structure; what the hands do on B25; damage numbers measured in play.
- Ordinary map-to-map exits.
- Cutscene scene models (`FRITTO`/`SENTEI`) beyond the part directory, the ten unsplit scenes, and the leftover scene codes.
- Mini-game 1 stalls at the "start?" question in test runs; details of GAME02 enemy turning and GAME03 rules.
- System strings stored in the EXE and GAME01 to 03.
- Not tested in an emulator at all: the Cyrillic patch, the debug-HUD patch and the dmenu patch. They were derived from code only.

---

<a id="en-20"></a>
## 20. Notes on method and legal

Work was done with a home-made MIPS R3000 disassembler, a small MIPS+GTE interpreter, Python parsers for each container, and a PS1 emulator used to take RAM, VRAM and GPU command dumps of the running game. The tools are not part of this repository. Everything here comes from the author's own copy of the disc.

**No game assets, text, textures, models or audio are included**, and none should be added. If you reproduce any of this, work from a disc you own and share patches (diffs) rather than modified images.

*Alundra 2 belongs to its respective rights holders (Activision, Matrix Software). This project is a fan documentation effort and is not affiliated with them.*

---

[⬆ back to top](#top) · [🇷🇺 Русский](#ru)
<a id="ru"></a>

# 🇷🇺 Русский

## Оглавление

1. [Что это](#ru-1)
2. [Диск и исполняемый файл](#ru-2)
3. [Контейнеры: PAK и GRP.GP](#ru-3)
4. [Сжатие и формат картинок](#ru-4)
5. [Карты: запись из 24 слотов](#ru-5)
6. [Геометрия карт (слот 11)](#ru-6)
7. [Объекты, типы и скрипты (слот 10)](#ru-7)
8. [Язык скриптов](#ru-8)
9. [Текст, диалоги и шрифты](#ru-9)
10. [3D-модели и анимация](#ru-10)
11. [Материалы моделей](#ru-11)
12. [Враги, байткод ИИ, боссы, урон](#ru-12)
13. [Состояние игрока, предметы, магазины, сохранения](#ru-13)
14. [Звук](#ru-14)
15. [Мини-игры](#ru-15)
16. [Ролики (EVENT00, PAKVCAST)](#ru-16)
17. [Наследие разработчиков: отладочный HUD, меню, a2profile.ini](#ru-17)
18. [Граф обзорной карты](#ru-18)
19. [Нерешённое](#ru-19)
20. [О методе и юридическое](#ru-20)

---

<a id="ru-1"></a>
## 1. Что это

Разбор **Alundra 2** (PlayStation, Matrix Software, 1999) по результатам реверса. Целевая сборка: **NTSC-U, SLUS-010.17**.

Цель: понять игру настолько, чтобы собрать вокруг неё инструменты. Распаковать все контейнеры, читать карты, скрипты, модели, анимации, текст, звук и понимать, как движок всё это связывает. В репозитории только документация: ни кода, ни данных игры. Нужна своя копия диска.

Пометки уверенности:

| Знак | Значение |
|---|---|
| ✅ | Проверено на всех данных (числа указаны) и/или по коду игры, во многих случаях ещё и на запущенной игре (дампы RAM и VRAM) |
| 🟡 | Прочитано по коду или согласуется с данными, но в работе в игре не наблюдалось |
| ❓ | Догадка или неизвестно |

Адреса — адреса RAM PlayStation (`0x8xxxxxxx`). «Слот» без уточнения — пронумерованный блок внутри записи карты.

---

<a id="ru-2"></a>
## 2. Диск и исполняемый файл

### 2.1 Файлы

| Файл | Размер | Что это |
|---|---|---|
| `SLUS_010.17` | 305 152 | Основной PS-X EXE. Загрузка `0x80017000`, вход `0x8003CF4C`, SP `0x801FFFF0` |
| `GAME00.BIN` | 805 144 | Оверлей: сама игра (id 4) |
| `EVENT00.BIN` | 488 592 | Оверлей: проигрыватель событий/роликов (id 5) |
| `GAME01.BIN` | 456 376 | Оверлей: мини-игра 1 (id 6) |
| `GAME02.BIN` | 281 432 | Оверлей: мини-игра 2 (id 7) |
| `GAME03.BIN` | 173 528 | Оверлей: мини-игра 3 (id 8) |
| `DATA/GRP.GP` | 207 МБ | 512 записей карт |
| `DATA/PACK0.PAK` | 13 МБ | модели, анимации, картинки, шрифты, системный текст, музыкальные треки |
| `DATA/PACK1.PAK` | 0,7 МБ | кодовые оверлеи (титул, вступление, модули боссов) |
| `DATA/PAKOMAKE.PAK` | 4,4 МБ | бонусное содержимое («omake»), 117 записей |
| `DATA/PAKVCAST.PAK` | 79 МБ | данные роликов, 311 записей |
| `A2STR.STR` | 18,7 МБ | вступительное видео |
| `MONO.XA`, `STEREO.XA` | 22,5 / 180 МБ | CD-XA: голоса и музыка роликов |

Образ: 2352 байта на сектор, Mode 2, ISO 9660, около 258 244 секторов. В сборке библиотеки Sony начала 1998 (sys.c 1.140, bios.c 1.86, intr.c 1.75).

Следы путей разработчиков в бинарниках: `d:\a2\bin\` (`a2.prm`, `a2.frx`, `a2.tim`, `aaa.mbz`, `fardome1.mbz`, `aaa.map`, `fardome1.map`, `light.tmp`), а на машине разработчика каждая карта была отдельным файлом `data\grp\grpNNN.gp`.

### 2.2 Оверлеи ✅

Все пять оверлеев собраны под один адрес, **`0x80084A98`**, и загружаются по очереди друг поверх друга. Первый dword каждого файла — id оверлея. Таблица имён файлов (`0x80052170`) и планировщик модулей лежат в основном EXE.

Диапазон `0x80010000…0x80017000` занимает **отдельный библиотечный оверлей** («AL22»): записи PACK0 **76–79**, копируются на `0x80010000` по байтовой таблице `0x800521E8` (номер режима игры выбирает один из четырёх). В нём математика GTE (`RotMatrix`, `MulMatrix` и т.д.) и **растеризатор моделей**.

### 2.3 Автомат модулей ✅

Игра — небольшой конечный автомат. Планировщик (`0x8001A228`) читает номер следующего модуля из слова `0x80060888`, грузит файл из таблицы `0x80052170` (8 байт на строку: указатель на имя, адрес входа), вызывает вход с аргументом из `0x800608CC`, а **возвращённое число — следующий модуль**. `-1` — «вернуться на титул».

| № | Файл | Вход | Роль |
|---|---|---|---|
| 0 | `game00.bin` | `0x8008968C` | Сама игра (цикл карты) |
| 1 | `event00.bin` | `0x80087684` | Проигрыватель роликов; аргумент = номер события |
| 2 | `game00.bin` | `0x80089338` | Титульный экран (код в PACK1, запись 34) |
| 3 | `game01.bin` | `0x80084D50` | Мини-игра 1 |
| 4 | `game02.bin` | `0x80084D4C` | Мини-игра 2 |
| 5 | `game03.bin` | `0x80084CA4` | Мини-игра 3 |
| 6 | `game00.bin` | `0x8008939C` | Серия картинок (PACK0 24–28) и звук, вероятно титры 🟡 |
| 7 | `game00.bin` | `0x80089404` | Вступительный ролик (PACK1 запись 35, играет `A2STR.STR`) |

Порядок запуска: модуль 7 (вступление), затем 2 (титул; если 900 кадров нет нажатий, возвращает `-1` и вступление идёт заново), затем цикл модуля 0. Новая игра отправляет на событие **101** (модуль 1). Планировщик читает слово модуля только *перед* загрузкой оверлея, поэтому запись нового номера посреди игры ничего не даёт; она должна быть с первого кадра.

Мини-игры включаются с карт командой скрипта 198 (код 18/19/20 → модули 3/4/5) на картах **494, 493, 495**.

### 2.3.1 Полезные области памяти

| Адрес | Что |
|---|---|
| `0x8011D5C0` | Массив объектов: 60 × 804 байта (индекс 0 — игрок) |
| `0x80066BC8` | Сохраняемое состояние, `0xC88` байт (§13) |
| `0x80066C08` | Битовый массив флагов сюжета |
| `0x8006740C` | Количества предметов, 100 байт |
| `0x80134E90` | Заголовок записи карты (260 байт) после загрузки |
| `0x80134EE0 + 8k` | Указатели на слоты карты 10–14 |
| `0x8012C3A0` | Состояние диалога |
| `0x800EE6F4` | Таблица обработчиков команд скрипта (236 штук) |
| `0x80082830` | Таблица звуковых банков (127 × 12 байт) |

---

<a id="ru-3"></a>
## 3. Контейнеры: PAK и GRP.GP

### 3.1 PAK-файлы ✅

`PACK0`, `PACK1`, `PAKOMAKE`, `PAKVCAST` устроены одинаково:

- Заголовок: ровно **768 слотов × u32**. Слот = абсолютное смещение в файле, `0` = пусто.
- Ненулевые смещения возрастают. Размер записи = следующее смещение минус текущее; последняя идёт до конца файла.
- Заполнено: PACK0 — 693, PACK1 — 37, PAKOMAKE — 117, PAKVCAST — 311.
- В PACK0 много побайтовых дублей (45 групп, около 1,86 МБ лишнего).

Игра читает записи через `0x8001BD70(куда, первая, последняя, макс. размер)` после выбора пака `0x8001BD1C`.

**Что где лежит в PACK0:**

| Записи | Содержимое |
|---|---|
| 14, 15 | системные сообщения, имена персонажей и боссов |
| 23–28 | картинки на весь экран (титул, серия 768×488) |
| 29–38 | текстуры героя и прочее (32–38 — семь вариантов страницы героя, 256×256, 4 бита) |
| 39, 40 | `PC00_KEN` и `PC00_TATE`: пять мечей и пять щитов |
| 68–75 | модели героев и банки анимаций (§10) |
| 76–79 | кодовый оверлей AL22 |
| 299, 300, 516 | названия предметов, текст настроек, текст магазинов |
| 304–310 и далее | картинки событий (256×256, 6 палитр) |
| 375–402 | наборы картинок по четыре (картинка 256×256, таблица, картинка 256×16, таблица) |
| 537–552 | шрифты (§9.4) |
| 557, 559 | данные мини-игр 2 и 3 |
| с 581, тройками | музыкальные треки: последовательность, заголовок банка, сэмплы (§14.2) |

Около 212 записей PACK0 начинаются с 32-байтного ASCII-имени (`PC00_KEN`, `O01_01`, `M01`, `TA05`…) — это модели или банки анимаций.

### 3.2 GRP.GP ✅

- Заголовок: `0x800` байт = **512 × u32** абсолютных смещений. Выравнивания по секторам нет (читатель CD сохраняет остаток `offset & 0x7FF`).
- **Индекс записи = номер карты.** Другого соответствия нет. `startmap = 213` — это запись 213.
- Пустые слоты: 236, 289, 292, 299, 304, 423, 496, 497. Запись 0 особая (0x3620C байт, в основном нули).
- Каждая непустая запись — **заголовок 260 байт** и следом блоки слотов подряд.

**Заголовок записи (260 байт):**

| Смещение | Содержимое |
|---|---|
| `0x00` | 24 пары `(смещение, размер)` по 8 байт. Смещения отсчитываются от *начала данных* (запись + 260) и равны накопленной сумме размеров |
| `0xC0` | 4-байтная метка области: две буквы + две цифры (`hi00`, `pa00`…) |
| `0xE8` (полуслово 116) | номер записи PACK1 с кодом босса этой карты (ненулевое у 15 арен) |
| остальное | прочие поля до 260 |

Метки областей (число записей): pm 65, ki 43, pz 34, cg 28, gy 28, st 28, ud 25, yk 24, di 23, hi 21, pd 18, sn 17, fc 16, yo 16, aw 14, fm 12, pb 9, re 9, oo 8, fb 7, pa 6, se 6, bu 5, tb 5, nd 4, sh 4, fa 3, pc 3, ya 3, ha 2, ta 2. Особые записи: 400 `pmwa`, 498 `over`, 507 `star`, 510 `uso`, 511 `chec` и десять карт `test` (499–506, 508, 509) — полноценные маленькие карты.

Загрузчик (`0x800AE4A8`, дальше `0x800AF1F0`) читает 260-байтный заголовок, затем слоты группами (0–3, 4–7, 10–13) в рабочий буфер, затем переписывает поля смещений в заголовке в **указатели RAM**. Буфер выбирает байтовая таблица `0x800F095C`: `0x80` у 152 больших карт (`0x8019C000`, `0x50800` байт), `1` у 342 обычных (`0x801AB000`, `0x41800`), `0` у пустых слотов.

### 3.3 Обзор слотов ✅

| Слот | Содержимое | Где есть |
|---|---|---|
| 0 | Главные текстуры (контейнер типа 10, до 10 потоков 256×256, 10 палитр) | везде |
| 1 | Одна картинка 128×256, 8 бит | 47 уличных карт |
| 2 | Одна картинка 256×128, 4 бита, 2 палитры | везде |
| 3 | Три картинки 256×256, 4 бита, 3 палитры | везде |
| 4, 5, 6, 8, 9 | пусты | нигде |
| 7 | Банки звуковых эффектов | везде |
| 10 | Объекты, типы объектов, потоки скриптов | везде |
| 11 | Куски геометрии мира | везде |
| 12 | Фиксированные 5764 байта, назначение не подтверждено ❓ | те же 47 карт, что и слот 1 |
| 13 | Фиксированные 184 байта, освещение 🟡 | 383 карты |
| 14 | Текст карты (или заглушка 28 байт) | везде |

---

<a id="ru-4"></a>
## 4. Сжатие и формат картинок

### 4.1 Контейнер картинок ✅

Используется картинками PACK0 и слотами 0–3 карт.

```
u32 type        число потоков (1, 2, 3 или 10)
u32 palettes    число 256-цветных палитр
u32 size        ширина | (высота << 16)
u32 sizes[...]  сжатый размер каждого потока; весь заголовок = 0x34 байта
palettes        palettes × 512 байт, 256 цветов PS1 по 15 бит (бит 15 = полупрозрачность)
streams         подряд, вся запись выровнена до 4
```

Поток размера 0 — пропущенный слот (загрузчик всё равно сдвигает координату x в VRAM). Проверено: 207 из 207 потоков PACK0, 2781 из 2781 потоков слота 0, 2059 из 2059 потоков слотов 1–3 распаковываются ровно в ширина×высота (или вдвое меньше при 4 битах), входных байт не остаётся.

### 4.2 Методы потока (диспетчер `0x8001C860`) ✅

Первый байт потока = метод.

| Байт 0 | Метод | Заметки |
|---|---|---|
| `01` | RLE | токены как ниже, но `0x80` = *повторить следующий байт n раз* |
| `02` | LZ | используется всем, что нашлось на диске |
| `03` | без сжатия | данные после 4-байтного заголовка копируются как есть |

Методы 1 и 3 есть в коде, но данных с ними на диске нет.

Заголовок методов 1/2:

- Байт 1: глубина цвета. `0` = 4 бита на пиксель (выход = w×h/2 байт), иначе 8 бит (выход = w×h).
- Байт 2: если `0`, первый выходной байт лежит в байте 4, а токены начинаются с байта 5; иначе токены с байта 4.
- Байт 3: не используется.

Токен `t`:

- Длина `n = (t & 0x3F) + 1`. Если задан бит `0x40`, длина 14-битная: `((t & 0x3F) << 8 | следующий байт) + 1`.
- Бит `0x80` задан: **копирование**. Следующий байт + 1 = расстояние назад в уже выданных данных; копируем `n` байт по одному (перекрытие разрешено, получаются повторы).
- Бит `0x80` не задан: `n` литеральных байт.

Игра распаковывает в окно 16 КБ (`0x80061F28`) и заливает порциями по 16 384 байта через `LoadImage`. Картинки типа 2 (два потока) кладут второй поток правее первого в VRAM.

### 4.3 Куда в VRAM попадают картинки карты ✅

Раскладка, которую строит загрузчик карты (проверено: для карты 213 совпали 225 024 из 225 024 слов со слепком VRAM запущенной игры):

| Слот | Палитры | Пиксели |
|---|---|---|
| 0 | блок (768, 488), 256 в ширину, строка на палитру (`488 + j`) | поток k в x = 320 + 64·k, y = 0 (4 бита = 64 слова в ширину, 8 бит = 128) |
| 1 | строка (768, 503) | 128×256, 8 бит, в (960, 0) |
| 2 | строки (768, 498) и (768, 499) | 256×128, 4 бита, в (512, 384): нижняя половина текстурной страницы `0x18` |
| 3 | один блок 64×16 в (896, 464); палитра p = четыре строки `464 + 4p …` | три потока 256×256, 4 бита, в (320 + 64·k, 256) |

8-битный поток занимает два столбца страниц, поэтому за ним 4-битный поток иногда отсутствует.

### 4.4 Третья схема: текстуры роликов («MOD») ✅

Блоки текстур в `PAKVCAST` начинаются с `01 'M' 'O' 'D'`, дальше 256 слов палитры, дальше поток (458 из 458 блоков распаковываются):

- Пропустить один нулевой байт.
- Читать байт флагов, **со старшего бита**. Бит 0 = один литеральный байт. Бит 1 = пара байт `(a, c)`: скопировать `c + 3` байт, начиная с позиции `(строка − 1) · 256 + a`, по одному байту; `строка` — строка текущей позиции вывода.
- Над строкой 0 лежит виртуальная строка нулей.
- Результат всегда 256×256, 8 бит.

Блоки с меткой `00 'M' 'O' 'D'` хранят наборы палитр (сырой блок 5120 байт — десять палитр по 256).

---

<a id="ru-5"></a>
## 5. Карты: запись из 24 слотов

Коротко, из чего состоит карта: текстуры (слоты 0–3), звук (7), объекты и скрипты (10), 3D-мир (11), данные освещения (13), блок «неба» (1, 12) и текст (14). Важнейшие разобраны ниже.

### 5.1 Слот 14: текст карты ✅

```
u32  номер карты
u32  n
i32  смещения[n + 1]      (-1 = сообщения нет)
текст                     16-битные единицы
```

Сообщение k начинается в `8 + 4·(n+1) + смещения[k]`. Кодирование единиц описано в §9. Проверено на всех 511 блоках: каждое смещение лежит внутри своего блока, у каждой карты ровно одно сообщение начинается со смещения 0.

### 5.2 Слот 7: звуковые банки ✅

```
u32 n                     число банков (1…11)
u32 смещения[n]           первое всегда 68 (0x44)
банк:
    префикс  (28 + 8k байт)
    заголовок "SShd"       по префикс + 12
    SPU-ADPCM              длина = u32 по заголовок + 4
```

Проверено на 2163 из 2163 банков (27,8 МБ ADPCM, все блоки структурно корректны).

Префикс: `w0` = смещение заголовка в составном буфере, `w1` = w0 + длина заголовка, `w2 = 12 + 8k`, затем `k` записей по 8 байт `(u16 id, u16 номер сэмпла, 4 байта)`, `0xFFFFFFFF`, 16 байт хвоста. `id` звука **глобальный**: любая карта с id 3011 несёт один и тот же набор сэмплов. Всего в игре 186 разных наборов и 586 разных id. См. §14.3.

### 5.3 Слоты 12 и 13 🟡

Слот 13: 184 байта = 92 полуслова. Первые шесть чисел — две тройки RGB (например `128,128,128` и `150,121,109`), дальше знаковые векторы с фиксированной точкой. Вместе с именем файла разработчиков `light.tmp` (60 байт, в отладочной сборке грузится в указатель слота 13) это освещение: окружающий и направленные цвета плюс направления. Уличные карты используют нейтральный белый `255,255,255`, интерьеры — тёплые тона.

Слот 12: 5764 байта = 1 слово + 96 записей по 60 байт. Есть только у карт со слотом 1; отладочное имя `fardome1` подходит под далёкий купол неба. Это догадка ❓. Сам слот 1 — картинка 128×256, 8 бит; вероятно, фон или небо, но использования в игре никто не наблюдал.

---

<a id="ru-6"></a>
## 6. Геометрия карт (слот 11) ✅

Слот 11 — сетка мира, разбитая на куски (chunks).

### 6.1 Таблица слота (560 байт = 140 слов)

- Слово 0 = 560 (начало данных).
- Слова 1–44: смещения кусков (от начала слота), заполнены с начала; число от 0 до 44 на карту.
- Слова 126–129, 132–136, 139: фиксированные указатели (всегда заполнены и корректны, последний близко к концу слота). Смысл неизвестен ❓.
- Остальные — нули.

### 6.2 Заголовок куска

| Смещение | Тип | Смысл |
|---|---|---|
| `+0` | u16 | Флаги. Обычно `0xC000`; бит `0x8000` = виден (его переключают скрипты) |
| `+2` | 6 × s16 | Локальный bounding box min/max |
| `+14` | 3 × s16 | Мировая привязка x, y, z |
| `+20` | 9 × s16 | Матрица ориентации 3×3, 4096 = 1,0 |
| `+40` | u16 | Число полигонов |
| `+42` | u16 | Число вершин |
| `+44` | u32 | Смещение массива вершин (от начала слота) |
| `+52` | 32 байта | Необязательное имя (используют команды скрипта 194–196) |
| `+84` | | Начало потока полигонов |

Имя есть лишь у 275 из 5399 кусков (`door1_1`, `neji01`, `wing02`…`wing07`, `paati1`…`paati4`, `tru02`). Именованные куски — те, что скрипты показывают, прячут, двигают и вращают.

### 6.3 Полигоны

По 44 байта, либо 80, если в теге задан бит `0x04` (следом идёт второй 36-байтный пакет-наложение: для наклеек и теней). Это готовые пакеты GPU PS1:

| Смещение | Смысл |
|---|---|
| `0..3` | DMA-тег; байт 0 несёт флаги |
| `4..7` | индексы вершин v0..v3 (треугольник: v3 = v2) |
| `8..10` + `11` | цвет 0 (RGB) и команда GPU (`0x3C` G4, `0x34` G3, `0x38` GT4, `0x30` GT3, варианты `+2` — полупрозрачные) |
| `12..14`, `16..18`, `20..22` | цвета 1–3 |
| `24..35` | UV и слова CLUT / TPAGE: `u0 v0 clut`, `u1 v1 tpage`, `u2 v2`, `u3 v3` |
| `36..39` | слово `E2` (окно текстуры) |
| `40..43` | терминатор DMA `00 FF FF FF` (или `03 FF FF FF`) |

Четырёхугольник — любая команда с битом `0x10` (`0x3C 0x38 0x3E 0x3A`). Квады хранятся **по контуру** 0-1-2-3, режьте на (0,1,2) и (0,2,3). Порядок «зетом», как у GPU, даёт «бабочки» и дыры.

### 6.4 Вершины

По 12 байт: `x` в +0, `y` в +4, `z` в +8 (s16). Остальные полуслова — рабочие значения. Мировая позиция = локальная + привязка куска. Y — высота.

**Раскладка кусков точная:** поток полигонов заканчивается ровно на массиве вершин, а массив вершин — ровно там, где начинается следующий кусок. Это сошлось на 5399 из 5399 кусков.

---

<a id="ru-7"></a>
## 7. Объекты, типы и скрипты (слот 10) ✅

### 7.1 Заголовок (28 байт)

| Слово | Смысл |
|---|---|
| 0 | Смещение **таблицы типов** (по 44 байта на запись) |
| 1 | Смещение **списка размещений** (по 16 байт, конец — запись с первым байтом `0xFF`) |
| 2 | Смещение **каталога скриптов** |
| 3 | Размер слота |
| 4 | Размер слота − 8 |
| 5 | `28` (размер заголовка) |

Между байтом 28 и каталогом лежат потоки скриптов. После списка размещений обычно небольшой хвост; таблица типов и блобы моделей/анимаций — в остальной части.

### 7.2 Запись размещения (16 байт)

| Смещение | Смысл |
|---|---|
| `+0` | Индекс типа в таблице типов (`0xFF` = конец списка) |
| `+1` | Биты 0–2 × 64: направление/вариант; в основном чётные |
| `+2, +4, +6` | x, y (высота, хранится со знаком минус), z как s16; при создании сдвигаются влево на 12 (фикс. точка 20.12) |
| `+8` | Байт, минус 1 уходит в поле объекта `+0x30C` |
| `+10, +11, +13` | Три байта параметров (объект `+0x2AC, +0x2AD, +0x2AF`). Объекты-двери/выходы читают собственные параметры отсюда |
| `+14` | u16 → объект `+0x2B4` (он же номер флага сохранения у переключаемых объектов) |

Статистика по 6127 записям на 503 картах: в среднем 12,2 размещения на карту, максимум 31, 24 карты без них. Слот объекта = индекс размещения + 1, поэтому слоты 1–31 приходят из карт, а **слоты 32–59** зарезервированы под объекты времени выполнения (снаряды, эффекты, помощники игрока). Уже активный слот повторно не создаётся.

### 7.3 Запись типа (44 байта)

| Смещение | Смысл |
|---|---|
| `+0` | u16 id объекта (`0xFFFF` = конец) |
| `+2` | Класс: 2 враг, 4 босс, 11 предмет, 12 NPC / триггер / эффект, 14 объект «D» |
| `+4` | Слово флагов. Биты 11–13 = вариант (→ объект `+0x30D`); биты 14–15: `0x8000` = модель не настраивать, `0x4000` = ещё грузить банк анимаций |
| `+28` | Смещение модели в слоте |
| `+32` | Смещение банка анимаций |
| `+36` | Начальный индекс в таблице tpage |
| `+37` | Начальный индекс в таблице clut |
| `+38, +39` | Сдвиг u и v для каждой вершины (часты 64, 128, 192) |
| `+40` | Доп. блок (32 указателя) |

**Диапазоны id:** A 1–87 враги, B 101–126 боссы, N 200+ NPC, I 300 предметы, T 401–499, D 501–524 объекты, E 600+ (снаряды, части боссов, эффекты).

**Модели лежат здесь.** Врагов, боссов и NPC нет в PAK-файлах вообще. Каждая карта несёт модели нужных ей объектов вместе с банками анимаций внутри слота 10. Всего по картам 3429 встроенных моделей, 546 различных. Части разбираются точно у 4433 из 4442 моделей.

Объект в RAM (804 байта): `+0` активен, `+0x0A` id типа, `+0x0C` класс, `+0x10` флаги, `+0x18/1C/20` позиция (20.12, Y со знаком минус), `+0x38` HP, `+0x3A` макс. HP, `+0x14/+0x16` анимация, `+0x7C` скорость, `+0x248…+0x258` состояние ИИ (§12.3).

### 7.4 Каталог скриптов

`0x80096C68(группа, номер)`: слово каталога `[группа + 1]` указывает на таблицу u32; элемент `номер` — смещение потока (0 = нет, −1 = выключен). Группы 0–8 (у 504 из 509 карт заполнены все девять). Всего 5106 потоков на всех картах, и первый байт каждого потока меньше 236 (числа команд).

---

<a id="ru-8"></a>
## 8. Язык скриптов

### 8.1 Интерпретатор ✅

Вход `0x8009E17C`. Каждый шаг читает байт команды `c` (0–235), вызывает `handler[c](массив объектов, указатель на команду)` из таблицы `0x800EE6F4`, а обработчик возвращает **указатель на следующую команду**. Возврат `0` = остановить скрипт; возврат того же указателя = ждать здесь (так устроены все команды-ожидания). Команда **0** — терминатор (длина 1), дальше нули до 4 байт.

Длины известны для 232 из 236 команд. С ними точно разбираются 4539 из 5064 потоков (дошли до терминатора, дальше одни нули). У условных переходов две длины (провал и переход); линейный разбор берёт короткую.

### 8.2 Селектор объектов ✅

Большинство команд принимают первым аргументом байт-селектор (`0x80096E64`):

| Значение | Выбирает |
|---|---|
| `0xFF` | никого |
| `0x00..0x7F` | объект с этим номером слота (если активен) |
| `0x80` | собственный объект скрипта |
| `0x81` | все активные объекты классов 2, 3, 4 |
| `0x82` | все объекты класса 9 |
| `0x83` | все активные, кроме класса 5 |

Выбранные указатели кладутся в массив `0x8012C328`; команда применяется к каждому.

### 8.3 Команды с подтверждённым смыслом

| № | Длина | Смысл |
|---|---|---|
| 0 | 1 | Конец скрипта |
| 1 | 3 | Ждать N кадров (u16) |
| 2 | 7 | Системное сообщение (текст из таблицы основного EXE) |
| 3 | 4 | Показать сообщение карты: `[3][номер сообщения][группа говорящего][флаги]`. Флаг `0x80` = не блокировать |
| 5, 6 | | Ветвление по ответу в диалоге (два варианта) |
| 8 | | goto (s16) |
| 9 | | call (стек возврата на 4) |
| 10 | 1 | return |
| 11 | 1 | перезапуск этого потока с начала (обычная последняя команда; 1143 использования на 284 картах) |
| 12 / 13 / 14 | 3 | поставить / снять / инвертировать флаг сюжета (u16) |
| 15–22 | 5 | переход / вызов / возврат по флагу; 22 — блок под флагом |
| 23–27 | | переменные объекта (16 × u16): присвоить, +, −, a+b, a−b |
| 29 | 4 | ждать значения флага |
| 30, 217 | 3, 5 | открыть диалог на 2 / 4 варианта (тексты по номерам) |
| 31 | 3 | вернуть игрока из особого состояния (`0xFFFF` — пропуск) |
| 32 | 1 | сбросить режим игрока |
| 33 / 34 | | выдать / заплатить деньги (0 = взять сохранённую рабочую сумму) |
| 35 / 36 | | выдать / убрать один предмет (по id) |
| 37–39 | | лечить HP / вторую шкалу / полное лечение |
| 40, 41 | | переключатели «кинорежима» 🟡 (41 ещё очищает таблицу 12 всплывающих чисел) |
| 42 | | затемнение (RGB, кадры, режим) |
| 53 | | запомнить число выбранных объектов |
| 56 | | условие: объект входит в выбор |
| 61 | | условие над выбранными (ставит состояние, работает с 70/71) |
| 64 | | условие: денег не меньше N |
| 70, 71 | 3 | прыжок, если состояние ≠ 0 / = 0 |
| 72–77 | | call / return / блок на основе того же теста состояния |
| 78, 79 | 1 | префикс IF / IFNOT: следующая команда выполняется, только если состояние позволяет |
| 87 | | условие: диалог идёт (самая частая безымянная, 1861 использование) |
| 88 | 2 | создать объект из размещения (байт − 1); при неудаче ошибка и останов |
| 89, 96 | | поставить объект в позицию (×4096) |
| 91, 99, 112 | | байты режима объекта |
| 92, 98 | | биты режима |
| 93 | 3 | вспомогательный вызов объекта (параметр анимации) |
| 94 | 4 | задать анимацию/кадр группе объектов (s16; −1 = по умолчанию) |
| 95, 119, 124 | | анимации по номеру |
| 100, 101 | 6 | установить / сбросить биты в слове флагов объекта (32-битная маска) |
| 103 / 104 | | «включить» / «выключить» переключаемый объект (сохраняет флаг, играет кадр 130 или 131) |
| 105, 106 | | звук по виду объекта / остановка |
| 109 | | смещение по высоте |
| 113 | | поле скорости |
| 114 | 8 | вектор смещения группы (три s16 × 16 в `+0xB8/BC/C0`) |
| 115 | | номер состояния игрока (`+0xB60`) |
| 116 | 9 | поставить в позицию другого объекта + повёрнутое смещение |
| 120 | | однократное включение для видов ≥ 501 |
| 128 | 4 | непрерывное вращение |
| 152–158 | | управление камерой: цель, дистанция, угол (плавно) |
| 167, 168 | | сброс камеры / режим |
| 180, 181 | 3 | ждать, пока флаг установлен / снят |
| 190, 191 | | поставить / снять глобальный переключатель |
| 192 | 6 | **запустить ролик**: номер события − 100 в байтах 1–2, карта возврата в байтах 3–4 |
| 193 | 2 | байт слоя/режима |
| 194, 195, 196 | | **показать/скрыть, сдвинуть, повернуть именованный кусок карты** (дальше идёт имя на 32 байта) |
| 198 | 2 | **смена модуля**: код 18/19/20 → мини-игра 1/2/3; 15 и 22 → игра с полем режима; остальные — запросы сохранения/загрузки |
| 200 | | условие: два полуслова равны у выбранных |
| 220 | 1 | вернуться на титул |
| 221 | | ждать флаг состояния |
| 222 | | задать `worldcnt` |
| 225 | 1 | сдать все кристаллы (предмет 68) в очки |
| 226 | | сыграть звук из небольшой таблицы |
| 228 | 2 | прямо задать уровень комбо |
| 229 | | добавить медали (u16) |
| 230, 232 | | сыграть звук с `id + 100`; задать громкость звука |

Примерно 140 из 232 команд (короткие объектные операции и камера) пока безымянны. Полная таблица по каждой команде (адрес обработчика, длина, число использований) есть, но для этого файла слишком длинна.

### 8.4 Флаги ✅

Один битовый массив по `0x80066C08` (`байт = n >> 3`, `бит = n & 7`, младший первым). Функции: проверить `0x8001D398`, поставить `0x8001D3D4`, снять `0x8001D414`, инвертировать `0x8001D454`.

Реально используется около 7000 номеров флагов (0–511: 3090 обращений; 512–1023: 683; 1024–2047: 389; 2048–6927: 2849). Флаги группируются по 256 в **области**: блоки 0 и 2 — общее состояние сюжета, остальные принадлежат группам соседних карт (блок 5: карты 20–25, блок 6: 50–59, блок 9: 70–85, блок 10: 106–127, блок 19: торговец медалями и три аркады, блок 27: только карта 220 и так далее). Часть младших флагов (0, 1, 2) временные и переключаются между сохранениями.

### 8.5 Передача в ролик

Команда 192 вызывает `0x80098A18(событие, карта)`: ставит модуль := 1, аргумент := событие и запоминает карту возврата (`+0xB14`, плюс `+0xB16 := 2`). Дальше работает EVENT00 (§16).

---

<a id="ru-9"></a>
## 9. Текст, диалоги и шрифты

### 9.1 Кодирование сообщений ✅

16-битные единицы. `0xFF00 | c` — печатный ASCII-символ `c`. Всё остальное — управляющие коды. Сообщение заканчивается единицей `0` или `1`. Проверено: все 2855 сообщений карт кончаются на 0 или 1 (2542 / 313), и сообщения идут вплотную без промежутков на 2344 из 2344 записей таблицы.

Интерпретатор: 32 строчных состояния по 32 байта с `0x80068E40`; единица `< 18` ведёт по таблице `0x80017978`.

| Код | Смысл |
|---|---|
| 0, 1 | Конец сообщения (два поведения окна) |
| 2 | Ждать кнопку / конец страницы |
| 3 | Новая строка; если окно уже заполнено, ведёт себя как 2 |
| 4 + параметр | Стиль строки (цвет), параметры вроде 7, 1, 3, 6, 4 |
| 5 + параметр | Задержка скорости печати |
| 6 | Скорость из пользовательской настройки (таблица `0x80055C64`) |
| 7 + параметр | Второе поле стиля (в данных параметр всегда 0) |
| 8 | Центрировать следующую строку: `x = (320 − ширина) / 2` |
| 9 | Конец вставленной строки |
| 10 | Вставить системное сообщение 10 (имя героя) |
| 11, 12 | Вставить число из переменных игры |
| 13 + параметр | Вставить строку из 64-байтных слотов `0x80069290` |
| 14–17 | Значки кнопок (шириной 12 пикселей) |

Коды 4, 5, 7 и 13 берут **одну единицу-параметр**. Разбор, который останавливается на первой `1`, обрежет сообщения посередине. Пять единиц `171 214 151 222 190` плюс одна из `20..23` встречаются в 292 картах; это не ASCII, а остаток японской сборки, которого в US-версии, вероятно, не видно ❓.

### 9.2 Правила окна ✅

Ширина окна 288 px (`0x120`). Латинская буква — 6 px. Оригинал допускает три строки на страницу, каждая ≤ 282 px. **Автоматического переноса слов нет**; каждый перенос — код 3. Все 3303 оригинальных сообщения (карты плюс системные) этим правилам удовлетворяют.

### 9.3 Системный текст ✅

Тот же формат записи, что у слота 14: `u32 id, u32 count, i32 смещения[count + 1], единицы`.

| Запись PACK0 | Содержимое |
|---|---|
| 299 | 257 записей: предметы, оружие, щиты, ключевые вещи, реликвии |
| 300 | 28 записей: опции и настройки |
| 14 | 30 записей: уведомления (сундуки, двери…) |
| 15 | 99 записей: имена персонажей и боссов |
| 516 | 62 записи: магазины, обмен жетонов, лорды стихий |

Остальные строки (меню, подписи мини-игр) лежат внутри EXE и оверлеев и не каталогизированы.

### 9.4 Шрифты ✅

| Запись PACK0 | Содержимое |
|---|---|
| 537–540 | Шрифты японской сборки |
| **552** | **Диалоговый шрифт US-сборки**: 96 глифов × 73 байта |
| **551** | **Малый/меню-шрифт**: 96 глифов × 49 байт |
| 538 | Блок 320 глифов × 72 байта, который US-сборка по константе не грузит (свободное место) |
| 541–550 | альтернативные начертания (жирный, рукописный, узкий…) |

Раскладка глифа (большой): 72 байта растра, **12×24 пикселя, 2 бита на пиксель, 3 байта на строку, MSB-first**, затем **один байт ширины в конце** (смещение 72; у малого шрифта 12×16 — смещение 48). Ширина 0 или ≥ 12 означает значение по умолчанию 6; малый шрифт пропорциональный (5–10).

Загрузчик `0x80021638` заполняет `0x8014C880`; путь US грузит запись 552 в `0x8014C888` (инструкция по `0x800216B4`, `addiu s0,zero,552`), а запись 551 в `0x800693A0`. Блиттер `0x8002A230` пишет в страницу VRAM (960, 472). Ширина — `0x8001FAE0`, адрес глифа `шаг × c − 2264`.

Слово `0x80060854` при старте равно `1`; тогда режим текста (`0x80067930`) становится **3 (US)**. В режиме 3 код берёт `единица & 0xFF` как символ и `символ − 32` как номер глифа, а проверка диапазона пропускает **32–255**. Значит движок уже умеет **224 глифа**, как PAL-версии, но запись шрифта US даёт только 96.

**Путь к кириллице (проверен по коду, в эмуляторе не запускался 🟡):** записать шрифт на 224 глифа с позициями CP1251 (`0xC0..0xFF`, Ё `0xA8`, ё `0xB8`, « » `0xAB/0xBB`, — `0x97`, … `0x85`, № `0xB9`) в запись 538 (23 040 байт, места хватает) и поменять в EXE одну инструкцию, чтобы вместо записи 552 грузилась 538 (`0x24100228` → `0x2410021A`). Размеры файлов не меняются, меняются лишь 13 секторов. За буфером шрифта (до `0x80157708`) абсолютных ссылок из кода нет, так что 16 352 байта дополнительных глифов помещаются. Малый шрифт так же не расширить: `0x8006A800` лежит через 512 байт после него.

Перезапись текста карт может увеличить запись: записи GRP.GP адресуются по байтовому смещению, поэтому таблицу можно пересобрать, а файл перенести в конец образа диска (EDC/ECC для Mode 2 Form 1 нужно пересчитать; проверка на 3240 из 3240 настоящих секторов сошлась).

---

<a id="ru-10"></a>
## 10. 3D-модели и анимация

### 10.1 Сначала отучимся от одного ✅

**Иерархии скелета в этом движке нет.** Модели разбиты на части (от 4 до 20 в зависимости от модели, 16 у обоих героев). Каждая часть — жёсткий кусок, смоделированный вокруг *собственного* нуля. Для каждого кадра анимация задаёт каждой части её собственные абсолютные сдвиг, поворот и масштаб, без связи «родитель–ребёнок» и без стека матриц. Названий вроде «предплечье» или «голень» нигде не хранится. Любая сборка персонажа по фиксированным смещениям суставов неверна; поза берётся только из кадра анимации.

### 10.2 Запись модели ✅

```
+0x00   имя[0x20]                ASCII (например PC00, O01_01)
+0x20   u32 смещения частей[...] завершено 0xFFFFFFFF
+0x128  u32 число внешних вершин
+0x12C  u32 число частей
```

Число частей: герои 16, PLAYER 15, TARGET 16, TO01 16, M01 9, M02 4, O01 13, O02 12, O03 13, O04 14, M07 17.

**Заголовок части:** 9 × s16 = (0, bbox min xyz, bbox max xyz, число полигонов, число вершин); `u32` по `+0x14` — абсолютное смещение блока вершин; элементы начинаются с `+0x18`. **Вершины** (последние `nvert × 16` байт части): `s16 x, y, z`, `s16 nx, ny, nz` (4096 = 1,0), байт 12 = **слот экспорта внешней вершины** (см. 10.5), байт 13 = флаг неизвестного смысла ❓, ещё 2 байта.

Ось Y направлена **вниз**. Герой на земле имеет ступни при y ≈ 0 и макушку волос на −138.

Разбираются все 555 частей в PACK0 и PAKOMAKE, число полигонов совпадает с заголовком; bbox из заголовка равен реальному размаху вершин у всех.

**Элемент (11 слов):** флаги | индексы (4 байта, `v2 == v3` = треугольник) | четыре слова цвета `0x00BBGGRR`, у первого в старшем байте команда GPU (`0x30/0x38` Гуро без текстуры, `0x34/0x3C` с текстурой, `+2` = полупрозрачный; встречались `30 32 34 36 38 3A 3C 3E`) | три слова UV | слово `E1/E2` | терминатор `0xFFFFFF0t`.

- Квады хранятся по контуру (0-1-2-3). Проверка: при 0-1-3-2 выпуклым оказывается 1 из 257 квадов героя, при 0-1-2-3 — все 257.
- Направление обхода граней **не выдержано**; у многих граней игра рисует без отсечения. Экспортируйте двусторонними.
- Флаги (младшие 16 бит): бит 2 (значение 4) = следом идёт **пакет-наклейка** (80 байт вместо 44), подтверждено на 1156 элементах без исключений в обе стороны; биты 0–1 = способ глубины для сортировки (0: среднее по 4 вершинам, 1 и 2: пара min/max, 3: фиксированное `0x1FFB`) 🟡; бит 3 (8) = пропустить проверку обратной грани 🟡; бит 9 (512) стоит примерно у половины элементов, смысл неизвестен ❓.
- Наклейка: 9 слов, без флагов и без индексов (использует `idx` родителя): 4 цвета, 3 слова UV, `E1/E2`, терминатор. У героев тело — обычные гуро-полигоны, а лицо, глаза и повязки — текстурированные наклейки сверху. В экспортёре сдвигайте наклейку на 0,5 единицы по нормали, чтобы не мерцало.
- Терминатор `0xFFFFFF0t`: младший байт `t` — **номер записи материала** (0 = нет); игра умножает его на 4 как индекс в таблицу указателей в части. Значения: 0 (10033), 1 (779), 2 (98), 3 (29) у элементов; 0, 1, 2, 4 у наклеек.
- Привязка текстур: пара `(tpage, clut)` на полигон; в модели это **малые индексы таблиц**, а не слова VRAM (герой: `tpage 0x0100`, `clut 0, 1, 2, 5`). Реальные значения — в §11.

### 10.3 Банки анимаций ✅

Банк лежит в записи сразу после своей модели (герой 68 → 69, 70 → 71, 72 → 73, 74 → 75; также PLAYER, монстры, боссы, `TA*`, `TO*`, `USI_T1..4`). 55 банков, 533 записи таблицы (485 разных блоков), 6629 дорожек костей, **66 290 каналов, все разобраны точно**.

```
+0x00   имя
+0x1E   u8   число костей (частей); равно числу частей модели у 45 из 45 пар
+0x20   u32[256]  смещения блоков анимаций (0xFFFFFFFF = нет; несколько могут делить блок)
+0x420  u32  смещение таблицы форм столкновений
+0x446  u16[256] флаги по номеру анимации (0x4000, 0x6000, 0x7000, 0xC000, 0xE000)
```

**Блок анимации** по смещению `o`:

```
u16 n               индекс последнего кадра (кадров n + 1)
u16 0
u16 дорожка[nb]     смещение дорожки от o, 0xFFFF = эта часть не анимируется
4 × (u16 кадр, u16 событие)   код события выдаётся, когда счётчик кадров доходит до `кадр`
доп. канал: пары (u8 длина, u8 значение) на n+1 кадров; значение = индекс в таблице форм столкновений
дорожки
```

**Дорожка:** `u16 канал[10]` (смещения от `o`). Каналы 0–2 сдвиг x y z, 3–5 поворот rx ry rz (4096 = оборот), 6–8 масштаб (4096 = 1,0), 9 видимость.

**Поток каналов 0–8:** слово `w`; значение = `int16(w & 0xFFE0)`; `n = w & 0x1F` — сколько кадров держится отрезок (минимум 1); если `n ≥ 3`, следующее `int16` — **наклон**, прибавляемый каждый кадр (единицы 1/8). Сдвиг = значение >> 5 (целые единицы модели), поворот и масштаб = значение как есть. **Канал 9 (видимость):** байты, где бит 7 = флаг, биты 0–6 = длина серии, выравнивание до чётного. **1 = скрыто.** Части 14 и 15 героя скрыты в 8 из 8 анимаций событийного банка (это оружие и щит).

Проверки: суммы длин отрезков равны числу кадров во всех 66 290 потоках, и каждый поток заканчивается ровно там, где начинается следующий (кроме однобайтового выравнивания канала 9).

У некоторых анимаций (две в банке Флинта и их копии) доп. канал даёт сумму 29 вместо 30: безвредная ошибка сборки в оригинале.

**Банки героев.** Модели 68, 70 и 74 побайтно одинаковы по геометрии, цветам и UV. В банке 69 108 анимаций, в 71 — 100, в 75 — 66. **Банки 73 (Алексия) и 75 побайтно равны**, нумерация сдвинута на 41: покой — анимация 3 у Флинта (15 кадров, высота 141) и 44 у Алексии. Длины анимаций Флинта: 0:30, 3:15, 4:29, 5:24, 6:19, 8:24, 9:18, 10:20, 32:60, 33:40, 37:50, …

### 10.4 Как ставить позу ✅

Для каждой части: `v_объект = R · S · v + t`, потом умножение на базовую матрицу объекта.

- `S = diag(sx, sy, sz) / 4096`.
- `R = Rz(rz) · Ry(ry) · Rx(rx)` через библиотечную `RotMatrix`. Подтверждено запуском библиотечной процедуры в маленьком интерпретаторе MIPS+GTE (совпадение 0,0004 на пяти наборах углов; остальные пять порядков промахиваются на 0,2–0,93). Если передан дополнительный угол рыскания, код строит `Rz · Rx · Ry` вручную; почему — неизвестно.
- Код позы: `0x80027B90` / `0x80027C6C`; шаг кадра `0x80026394`; чтение дорожек `0x80025D5C`; старт `0x80025F00`; вывод `0x800260F0`; события `0x8002628C`.

Не сверено с картинкой игры: единица наклона за кадр и частота анимации (принято 30 к/с).

### 10.5 Внешние вершины ✅

У боссов и некоторых моделей (O01–O04, PLAYER, M07, M08, TA11, USI_T) есть полигоны с индексами **≥ 60**. Они читаются из общего буфера, который заполняют *другие* части.

- Правило: когда растеризатор части преобразует вершину с ненулевым байтом 12, слот `k = байт − 60`, он ещё и записывает эту преобразованную вершину в общий буфер внешних вершин по `s7 + 16·k` (`0x800133A4…0x800133B4`). Индекс ≥ 60 (`idx << 4 ≥ 960`) читается из этого буфера, меньший — из буфера самой части.
- Проверка: у 118 из 120 моделей множество экспортируемых значений байта 12 в точности равно `60 … 60 + n − 1`, где `n` — слово по `0x128`, у каждого значения одна вершина-источник. У двух записей `O02_01` `n = 38`, а источников 36 (два слота не используются).
- Практически: швы между частями (плечи, бёдра, шея) — полосы полигонов между собственным кольцом вершин части и кольцом, принадлежащим соседней части и стоящим в позе соседа. Экспортёр обязан идти по таблице байтов 12. Подбор «ближайшей вершины соседа» попадает правильно лишь примерно в половине случаев.

### 10.6 Формы столкновений и ударов ✅

Таблица по `+0x420`: `u32[N]` указателей. Элемент 0 без тега: три s16 (w, h, d), тело по умолчанию. Флинт 25×138×25, боссы 30×200×30. Остальные элементы — списки с тегом до нуля: **тег 1 = коробка (центр xyz, размеры whd), 14 байт; тег 2 = точка, 8 байт; тег 3 = сфера (центр xyz + радиус), 10 байт.** 59 списков в 55 банках только с этими тегами (36 коробок, 16 точек, 47 сфер), и все 542 индекса доп. канала попадают в таблицу. В кадрах удара Флинта добавляется сфера радиусом 55–60 примерно в 135 единицах впереди (−Z), похоже на зону удара 🟡.

### 10.7 Рисование ✅

AL22 (`0x80012E70`): на каждую часть — запись 80 байт (флаги, номер части в `+2`, указатель на модель `+0x3C`, буфер внешних вершин `+0x48`, таблица материалов `+0x44`, матрица `+0x20..0x30`, сдвиг `+0x34..0x38`). Вершины поворачиваются командой GTE `MVMVA 0x4A480012`, нормали так же, затем `RTPS`. Процедуры настройки объекта `0x800B2040` / `0x800B2708` только строят матрицы частей через `0x80027C6C` и сами не рисуют.

### 10.8 Оружие и щит

Часть 14 (29 вершин, клинок вдоль +Z от −21 до 102) и часть 15 (18 вершин, тонкая пластина) у героев по форме и локальной системе совпадают с пятью вариантами мечей `PC00_KEN` (23/29/27/29/37 вершин) и пятью вариантами щитов `PC00_TATE` (18/18/18/18/27). Почти наверняка игра подменяет их, но что выбирает вариант, не найдено ❓.

---

<a id="ru-11"></a>
## 11. Материалы моделей ✅

В файле модели лежат только малые числа (`tpage 0..n`, `clut 0..n`). Реальный адрес в VRAM определяется, когда модель **настраивается для использования**, процедурой `0x80024DC4(модель, таблица_tpage, таблица_clut, du, dv)`.

Для каждого полигона с материалом `t = 0`:

```
слово_tpage = (T[(слово & 0xFF) + стартT] & 0xFF9F) | ((слово >> 3) & 0x60)
слово_clut  =  C[clut + стартC]
u += du;  v += dv        (байтовая арифметика, для каждой вершины, наклейки тоже)
```

Младший байт tpage модели выбирает строку таблицы; биты 8–9 становятся режимом смешивания (модельное `tp0100` = «строка 0, режим 1»). После этого `t` умножается на 4.

Для `t ≠ 0` код рисования (AL22 `0x80013C4C`) делает те же подстановки в момент рисования, с двумя сдвигами из записи материала (s16 по смещениям 14 и 16). **Где эти записи лежат в файле, неизвестно**, поэтому экспортёр показывает такие полигоны с нулевыми сдвигами ❓. Похоже на каналы состояний (вспышка, заморозка и т. п.).

### 11.1 Какие таблицы

| Случай | Таблица tpage | Таблица clut | du, dv |
|---|---|---|---|
| Модели карт, обычный класс | `0x80052480`: `0x15,0x16,0x17,0x18` = страницы (320..512, 256), 4 бита | `0x80052488`: 4 палитры × 16 рядов, `0x7438…` = x 896.., y 464..479 | из записи типа |
| Модели карт, класс 14 (объекты «D») | `0x80052588`: `0x18, 0` | `0x8005258C`: `0x7CB0 + i` | du 0, dv 128 |
| Модели карт, класс 4 (боссы) | `0x800522D0`: `0x85,0x87,0x89,0x8B,0x8D` = 8-битные страницы из слота 0 | `0x800522E8`: `0x7A30…` строки y = 488.. | 0, 0 |
| Герои (PACK0 68–75) | `0x8005244C`: `0x0019` = (576, 256), 4 бита | `0x80052460`: `0x7D30 + i` = (768 + 16i, 500) | 0, 0 |
| Эффектные записи 39–67 | сначала таблицы героя (39, 40), потом таблицы GAME00 `0x800F056C / 0x800F050C` | | |
| Объекты EVENT00 | tpage `0x800D27C6 + 2v`, clut `0x800E9410 + 16v` (заполняются во время игры) | | |

Поэтому одна модель, например `T02`, встречается на 159 картах со 118 разными наборами (tpage, clut, du, dv): **окраска принадлежит записи размещения/типа на карте, а не модели.**

### 11.2 Проверка

- Слепок RAM запущенной игры на карте 213: игра переписала слова полигонов в памяти; правило совпало **94 из 94** элементов с `t = 0` побайтно (у остальных 20 с `t ≠ 0` UV сдвинуты, clut/tpage не тронуты).
- Все 3260 моделей на всех картах, 96 565 текстурных полигонов и наклеек: каждый прямоугольник UV (после сдвига) лежит внутри VRAM, которую даёт сама карта, и каждая использованная запись палитры существует. Промахов ноль.
- Выгруженная картинка страница + палитра побитно совпала с реальной отрисовкой VRAM во всех четырёх проверенных случаях.
- Боссы (класс 4) и объекты «D» (класс 14) подтверждены кодом и покрытием, но не снимком игры.

### 11.3 Страница героя 🟡

Страница героя в (576, 256) грузится в GAME00 около `0x800AFD70` из **записи PACK0 `32 + n`**, `n = (s16 по 0x8006746E) − 39`; получается семь вариантов (записи 32–38). Они отличаются лишь одной областью UV (то, что покрывают наклейки `clut 5`), то есть похожи на варианты снаряжения или облика. Кто пишет `0x8006746E`, неизвестно ❓.

---

<a id="ru-12"></a>
## 12. Враги, байткод ИИ, боссы, урон

### 12.1 Таблица параметров ✅

`0x800A6924(id)` возвращает запись из 14 байт:

| id | Базовый адрес |
|---|---|
| < 100 (A) | `0x800EEF48 + 14·id` |
| 100–129 (B) | `0x800EEF58 + 14·id` |
| 400–499 (T) | `0x800EE0A4 + 14·id` |
| 600–759 (E) | `0x800EDB3C + 14·id` |
| прочие | общая пустая запись `0x8012CC18` |

Запись: `+0` стихия/класс (0–6), `+1` защита, `+2` базовая атака, `+3` флаги (`0x40` урон растёт с уровнем, `0x80` шанс ×2, `0x04` режет полученный урон вдвое), **`+4` u16 HP**, затем три пары `(ключ условия, урон)` (`+6/+7`, `+8/+9`, `+10/+11`; `255` = нет). Процедура создания (`0x800AE754`) копирует HP в `+0x38/+0x3A`. HP совпал с замером в игре у всех 40+ проверенных объектов. Всего 213 записей.

Примеры: A01 15, A02 50, A17 220, A54 280, A58 300; боссы: B01 200, B02 600, B11 150, B12 250, B13 200, B17 300, B18 400, B20 1000, B21 880, B22 200, B23 600, B25 1500, B26 300; E147 500, E146 360, **E148 4320** (максимум в игре). У B03 и B24 записи пустые; на арене 408 (B24) рядом лежат E148 и восемь E146, так что HP, вероятно, держат части 🟡. У игрока 200 HP.

### 12.2 Формула урона 🟡

По коду `0x800A8478…0x800A86C8`:

1. Стихии: `0x800A8314` сравнивает классы по кольцу 1–4 (±1 / ±3 дают преимущество / недостаток). Класс игрока берётся из оружия.
2. `база = атака` (или значение пары при совпадении ключа), масштаб `0x800A6BE0` (флаг `0x40`: × (уровень + 1); флаг `0x80`: ×2 при выпавшем условии), минус защита, не ниже 0, вдвое меньше против флага `0x04`.
3. Множитель стихии ×16 / ×8 / ×10 и деление на 10 (1,6 / 0,8 / 1,0). Если бьёт игрок и `0x8012CAD8` > 0: ×(v + 10) / 10. Прибавка случайных 0–3.
4. Сложность: полуслово `0x80066BD4` в новой игре `0`, тогда игрок наносит вдвое меньше и получает вдвое больше; иначе обычный урон. Потолок 999, у игрока минимум 1.

Уровень комбо `comblvl` (слово `0x80067726`, 0–4) — **ограничитель длины цепочки ударов**: `0x800A2334` сравнивает его с текущим шагом цепочки (`0x8012CAD8`), и каждый шаг прибавляет +10 % в формуле выше. Уровень растёт от очков за кристаллы (предмет 68, команда 225): порог = `3·(L+1)·(L+2)` очков.

### 12.3 Байткод ИИ ✅

Обычные враги (A1–A99, класс 2) не используют скрипты карт. Их поведение — **отдельный байткод** в GAME00 по `0x800F5A7C` (около 17,5 КБ). Раскладка: 4-байтная пустая программа, таблица **140 полуслов** (тип → смещение программы, `0xFFFF` = нет), затем программы. Интерпретатор `0x800EA7D8`: байт ≥ `0x80` — команда (таблица обработчиков `0x800F4828`, индекс = байт − `0x80`, 128 штук; `0x800EA870` = неизвестная с отладочной печатью); меньшие байты — данные. Обработчик, вернувший не ноль, завершает кадр.

Регистры объекта: `+0x250` вход, `+0x252` точка рестарта, `+0x254/+0x256` стек возврата (2 уровня), `+0x258` PC, `+0x25A` регистр условия, `+0x24C` таймер (убывает каждый кадр), `+0x25C..+0x25F` слоты дочерних объектов (0/255 = пусто), `+0x24D` режим (старший полубайт ставит `8F`, младший использует `BF`).

**Основные команды:** `F8` goto, `F9` call, `FF` return, `FD` yield, `FC` рестарт, `FA` 4-ветвевой switch по зоне, `E0/E1` `jeq/jne B,W`, `F4/F5` jnz/jz, `F6/F7` jge/jlt, `F0` ждать любое событие из маски, `F1` ждать все, `80/81/82` анимация (82 = по умолчанию из таблицы), `90–93` таймеры (93 = база + случайное), `95` звук, `B0/B1/B2` стоп/шаг/бег, `A0..A7` повороты (на игрока, ±, 180°), `BA/BB` вектор скорости (фикс. / случайный), `D1` создать потомка, `D2/D3` команды потомку, `DD` смешать две позы скелета (плавные переходы), `E9` случайный шанс B/256, `EB` случайное & B, `AC/AD/E4/E5/E6` операции с точкой-целью, `BF/EE` ветвления с переменной длиной (список слов до `0000`), **запрос `C0 BB`** (около 60 подзапросов: арифметика, расстояние до ближайшего «не врага» `0x84`, расстояние до игрока `0x90`, «игрок атакует» `0xC0` и т. д.). ГСЧ: ЛКГ `0x41C64E6D + 12345`, зерно по `0x8007D770`.

**События** (слово `0x80148E4A`, пересчитывается каждый кадр `0x800E5A90`): `0x8000` анимация закончилась, `0x4000` таймер вышел, `0x0100` поворот закончен, `0x0080/0x0040/0x0020` вертикальная скорость положительная/отрицательная/нулевая, **`0x0010/0x0008/0x0004/0x0002` = игрок в ближней / средней / дальней зоне / вне зон**.

**Таблица зрения** (`0x800F4C08`, 36 байт на тип A1..A99): квадрат расстояния `(dx >> 12)² + (dz >> 12)²` сравнивается с параметром в квадрате, плюс угол конуса (градусы → 0..511, 360° = 512). Пример A1: зоны 120 / 300 / 600 при конусе 180°; A2: 200 / 400 / 900 с более узкой дальней зоной. Другие поля: скорость шага/бега (×256 в `+0x7C`), база и случайная часть таймера (`+18/+19`), база звуков `1001 + 10·тип`, доп. звук `+34` (чаще 1003), вертикальная досягаемость ×100.

**Пример (A1):** четыре ветки по зоне: ближняя — выпад с анимацией `0x52`; средняя — пауза, потом один из случайных приёмов (боковые шаги с поворотом ±128 или 180°, рывок, удар); дальняя — медленное приближение; вне зон — стоит.

Программы боссов: индекс слота ≠ id для id ≥ 100: `120 → 120`, `710 → 121`, `711 → 122`, `712 → 123`, **`112 → 124`**, `715 → 125`, `716 → 126`, **`122 → 127`**, `720..731 → 128..139`. То есть **B12 (250 HP), B20 (1000) и B22 (200) — боссы на байткоде**, а не модули PACK1 (их записи PACK1 — пустые заглушки по 4 байта). B20 строит бой на дистанционных поясах и создаёт 645 / 710 / 711 / 712; B22 создаёт 720 / 721 / 723 (цепочка 723 → 724 → 725 → 726).

### 12.4 Модули боссов в PACK1 ✅

Полуслово 116 заголовка карты = запись PACK1 с кодом босса этой арены (грузится в `0x801493B0`, лимит 12 КБ, загрузчик `0x800AFCB8`). Только у 15 карт:

| Карта | Босс | PACK1 | Размер |
|---|---|---|---|
| 337 gy028 | B01 | 1 | 8424 |
| 288 di022 | B02 | 2 | 7364 |
| 303 di021 | B03 | 3 | 4 (пусто) |
| 401 hi020 | B11 | 11 | 7256 |
| 154 fc014 | B12 | 12 | 4 |
| 400 pmwa | B13 | 13 | 4596 |
| 402 yk100 | B17 | 17 | 8008 |
| 403 pd100 | B18 | 18 | 7716 |
| 406 st100 | B20 | 20 | 4 |
| 405 pz100 | B21 | 21 | 7792 |
| 404 ki039 | B22 | 22 | 4 |
| 407 st101 | B23 | 23 | 5356 |
| 408 st102 | B24 | 24 | 9280 |
| 409 st103 | B25 | 25 | 11 308 |
| 48 aw017 | B26 | 26 | 6984 |

Устройство модуля: первое слово — маленький id, затем таблица из 17–36 указателей на обработчики событий (много повторов), затем код, а в конце **таблица состояний** (по указателю на функцию на состояние: B01 7, B02 6, B11 7, B13 4, B17 4, B18 4, B21 6, B23 4, B24 6, B25 8 + 3 вспомогательных, B26 5). Слово состояния `+0x248`, таймер состояния `+0x24C`, подрежим `+0x250`. Звуки: `10000 + 100·босс + n` (B25: 12501–12518).

**B01** в состоянии 0 выбирает по квадрату расстояния на плоскости: ≤ 90 000 (≈ 300 единиц) → состояние 1 (ближние атаки), ≤ 202 500 (≈ 450) → состояние 2, иначе подходит (анимация 151 при > 360 000, анимация 133 при > 202 500). Счётчик с начальным значением 120 при достижении нуля отправляет в состояние 3 (особая атака).

**B25, финальный босс** (карта 409, голова с рогами на 1500 HP, руки E149/E150 с 0 HP, не получают урон, три висящих цели E147 по 500 HP). Скрипт арены (поток `@0xA0`) по очереди ставит флаги 1, 2, 3, видимо, по гибели трёх целей 🟡. Код босса (переменная фазы `0x8014BFA2`, 8 фаз, таблица `0x8014BD90`) читает их в фазе 0: каждый флаг снимается и отрабатывается фазой 2 (анимация 136, снаряд E152 через 120 кадров); третий ещё ставит флаг 4, после чего `0x8014BFD8` становится 1, и голова за 32 кадра выезжает по Z с −4 505 600 примерно к −2 720 000, а `0x8014BFA0` = 1 («голова выдвинута»). Фаза 7 (смерть) возвращает её обратно и выбирает фазу 4 или 5 (атаки). Проверено ручной записью `0x8014BFD8 = 1`. Уязвима ли выдвинутая голова, что делают фазы 4 и 5 помимо номеров анимаций и какой там урон — не выяснено. Игрок, ничего не делающий, теряет около 0,33 HP за кадр и погибает примерно за 600 кадров; после смерти игра переходит на карту 498.

---

<a id="ru-13"></a>
## 13. Состояние игрока, предметы, магазины, сохранения

### 13.1 Блок состояния ✅

Сохранение — прямой образ RAM. `0x80066BC8` длиной `0xC88` байт — это «всё, что сохраняется». Карта полей (смещение от `0x80066BC8`):

| Смещение | Смысл |
|---|---|
| `+0x00` u16 | счётчик сохранений (потолок 999), растёт при записи |
| `+0x02` u16 | «индекс места» = позиция текущей карты в таблице из 40 записей `0x800F2CD8`; нужен для подписи слота |
| `+0x04/06/08` | часы, минуты, кадры (60 в секунду) времени игры; `+0x0A` ставит его на паузу |
| `+0x0C` | полуслово `0x80066BD4` из §12.2 |
| `+0x40…+0x3A2` | флаги сюжета (6928 бит) |
| `+0x844` | количества предметов (100 байт, индекс = id предмета) |
| `+0x8EC` u32 | `1` в новой игре |
| `+0x90C…+0xB0B` | 16 «сумок» × 0x20 байт |
| `+0xB0C` u16 | маска пройденных узлов обзорной карты (§18) |
| `+0xB10` u16 | `worldcnt` (0–24) |
| `+0xB14` u16 | карта возврата / стартовая карта (213 в новой игре) |
| `+0xB16` | маркер возврата из ролика |
| `+0xB1C…B22` | 165, 177, 214, 157 в новой игре (смысл неизвестен) |
| `+0xB3C` u32 | деньги (потолок 999 999) |
| `+0xB40` u32 | **медали** (потолок 99 999) |
| `+0xB44` u32 | очки уровня |
| `+0xB48…+0xB4E` | макс. HP, HP, максимум второй шкалы, вторая шкала (по умолчанию по 200) |
| `+0xB50/54/58` | x, y, z игрока как s32, фикс. точка 20.12 |
| `+0xB5C` u16 | направление взгляда (12-битный угол) |
| `+0xB5E` u16 | уровень комбо |
| `+0xB60` | номер состояния игрока |
| `+0xB66…+0xB76` | 9 × u16 маски кнопок (по умолчанию `0x80,0x20,0x40,0x10,0x01,0x02,0x08,0x04,0x100`) |
| `+0xB78…+0xBEF` | запас магазина (15 × 8 байт) |
| `+0xBF8` u32 | лучшее время мини-игры 1 (по умолчанию `0xEA24` = 999,0 с в кадрах) |
| `+0xBFC / +0xC04` | рекорды мини-игр 2 / 3 |

Игра держит ещё и вторую копию блока по `0x80065F38`.

### 13.2 Формат карты памяти ✅

Файлы называются `BASLUS-01017NN` (один блок, 8192 байта). Раскладка: `SC`, флаг иконки `0x11`, 1 блок, 64-байтный заголовок в Shift-JIS (время игры и номер слота полноширинными цифрами), палитра по `+0x60`, иконка по `+0x80`, **копия 3208-байтного образа RAM по `+0x100`**, а затем **контрольная сумма u16 по `+3464` = сумма u16 по первым 3464 байтам** по модулю 65536 (`0x800C9690`). Остальные ≈ 118 байт до конца записанной длины — остаток буфера, суммой не покрыты. Проверено на всех пяти слотах настоящей карты (счётчик, индекс места, поля времени, HP совпали с экраном).

### 13.3 Предметы ✅

- Количества: `0x8006740C` (100 байт); новая игра обнуляет и ставит предметы 31 и 39 в 1 по таблице `0x80056314`.
- Таблица описаний `0x80055DB4`: 86 записей × 16 байт. Поля: `+0` группа, `+2` флаги (`0x40` есть эффект при использовании, `0x0800` = группа 31–38 оружие, `0x1000` = группа 39–46 защита, `0x2B` расходник), `+4` значение, `+8` тип эффекта, `+9` подтип, `+0xA`, `+0xC` величины.
- Применение `0x80023D78(id)`. Типы эффектов: **1** лечить HP, **2** лечить вторую шкалу, **3** поднять макс. HP, **4** поднять максимум второй шкалы, **5** деньги. 0x34/0x36/0x37 лечат 100/300/600; 0x39/0x3A восстанавливают 200/500 второй шкалы; 0x05/0x3E/0x3F/0x45 дают +25 макс. HP; 0x0B/0x42 дают +100 к максимуму второй шкалы; 0x50–0x55 дают деньги 100/200/500/5/10/20. Типы 6–10 не разобраны.
- 16 стартовых сумок (`0x800674D4`, по 0x20 байт): `u16 число`, затем id, затем флаги (id 0x34, 0x36, 0x37, 0x39, 0x3A и 0x20–0x2A, 0x45, 0x0B, 0x05).
- **Кристаллы (предмет 68)** превращаются в очки командой 225; пороги уровней комбо 0, 1, 2, 3 — 6, 18, 36, 60 очков.
- Предметы, которые не выдают ни скрипты, ни магазины, ни стартовые сумки: 2, 3, 7, 8, 13, 19, 20, 37, 38, 45–50, 59, 60, 63, 64, 71, 73, 75–77, 80–85. Часть из них (80–85, деньги) скорее подбирается с размещённых объектов.

### 13.4 Магазины ✅

15-слотовая таблица — **запас магазина**: `+0` флаги (старший полубайт = место на витрине, бит 0 = продано/особая, бит 1 = новое), `+1` id предмета, `+2` есть в продаже, `+4` u16 цена, `+6` u16 второе число. Покупка (GAME00 `0x800E5160`): проверить деньги, сравнить запас с пределом (9 для предметов, у которых первый u16 равен 0 или 1, иначе 99), затем `звук 2`, добавить предмет, списать деньги; иначе звук 63. Первая группа (8 мест, из `0x80056424`): предметы 23–26 по 2500 / 1500 / 1000 / 2000, предмет 4 за 30 000, 11 за 5000, 74 за 9000, 56 за 2000. Вторая группа (7 мест, `0x80056464`): расходники 52 / 54 / 55 / 22 / 56 по 10 / 100 / 250 / 300 / 1500, предметы 70 и 72 по 2000 / 4000 и 2500 / 5000. Таблица сохраняется, поэтому «продано» переживает сохранение.

### 13.5 Медали ✅

Второй счётчик — **медали**, их тратят три аркадные игры. Команда 229 добавляет медали. Торговец медалями на карте 414 продаёт по **5 денег за медаль**: 2000 → 400, 500 → 100, 100 → 20.

### 13.6 Карты с большими деньгами 🟡

Карты 485 и 487 многократно выдают и забирают оружие/защиту и крупные суммы (487 при входе даёт 100 000, 485 даёт 30 000). Карта 502 даёт и забирает 30 000. Похоже на тестовые комнаты разработчиков; выплата на 487 подтверждена прыжком прямо на эту карту.

---

<a id="ru-14"></a>
## 14. Звук

В игре три отдельные звуковые системы.

### 14.1 CD-XA (ролики и голоса) ✅

- `STEREO.XA`: Level B стерео, 37 800 Гц, 4-битный ADPCM, **8 чередующихся каналов** (шаг секторов 8).
- `MONO.XA`: Level B моно, 37 800 Гц, **16 каналов** (шаг 16).
- Две мастер-таблицы в основном EXE: `0x80056EF8` = **125 музыкальных треков**, `0x800576C8` = **52 голосовых клипа**. Запись (16 байт): `стартовый сектор, канал (0..7 или 0..15), число секторов, указатель на имя`. Секторы трека — `LBA файла + start + канал + i × шаг`. Номер канала в субзаголовке каждого сектора согласуется у 125 из 125 и 52 из 52.
- Имена лежат в EXE: `MX_12` (5950 секторов ≈ 5,3 мин), `MX_11S`, `001AS/BS/CS`, `030AS/BS`, `JX_04A`, `JX_07A`, `JX_09`, `JX_10`, `JX_11`, `SAL01/02`, `SEI01/02`, `DRI01/02`, `PIX01/02`, `CARDV01..21` (голоса крупье карточной игры), и сюжетные клипы вроде `002AM`, `021AM`, `040BM`.
- **Это звук роликов и событий, а не фоновая музыка карт.** Имена XA лежат внутри записей PAKVCAST своей сцены (числовой префикс = сцена, суффикс `S` стереомузыка, `M` голос). `MX_12` упомянут только в одном модуле PACK1 (вероятно, финал); `023B` не упомянут нигде.
- Воспроизведение по имени: `0x80030430(const char*)` (поиск `0x80030254`). Вызывают отладочная клавиша, EVENT00 (имя из потока скрипта) и GAME00. Воспроизведение XA из команд сцен (`xaplay`, `xaseek`) в розничном EVENT00 — заглушка.
- Декодирование ADPCM — стандартные фильтры CD-XA: `f0 = [0, 60, 115, 98]`, `f1 = [0, 0, −52, −55]`.

### 14.2 Фоновая музыка: секвенсор ✅

Обычная музыка в игре — не XA. Это **MIDI-подобный секвенсор**: `SSsq` (последовательность), `SShd` (заголовок инструментов) и **сэмплы SPU-ADPCM**. Всего 30 треков. Таблица «карта → трек» по `0x80056C20`. В PACK0 каждый трек — тройка: **запись 581 + 3·(k−1) = SSsq, +1 = SShd, +2 = ADPCM.** Неиспользуемые последовательности (нет в таблице): 563, 567, 572, 584, 626, 641, 647, 650, 656, 677.

- Заголовок `SSsq`: dword `0x01E00064`, `N` (48–128), 0, метка `SSsq`; дальше `N` записей по 16 байт.
- Обработчик событий `0x800489F8` (note off/on, aftertouch, control change, program change, pitch bend, system; мета `0x2F` = конец), разбор заголовков дорожек `0x80025D5C` (10 дорожек × 20 байт). Звуковой драйвер — диапазон `0x8004B000…0x80050000`; проверка `SSsq` в `0x8004F03C`.
- Высота = `(нота − центр) + fine/16 + bend/64·2` полутонов; громкость = `velocity × громкость тона × громкость`; панорама из тона + канала + программы; ADSR из регистров тона. Контроллер цикла CC 99: значение `0x14` — начало петли, `0x1E` — конец.
- Сверка на треке 34 с запущенной игрой: корреляция высот 0,89, отношение таймингов 1,000, период петли 64,65 с.
- Не проверено: панорама по программе, ударные программы, точное поведение sustain и звучание на слух человека.

### 14.3 Звуковые эффекты (слот 7) ✅

Загрузчик банка `0x8004EBE0(adpcm, заголовок, размер)` проверяет выравнивание 4, размер ≤ `0x80000`, кратность 16 и метку `SShd` по `заголовок + 12`; регистрирует банк в 127-слотовой таблице `0x80082830` и заливает в SPU (`0x8004E2E0`). Воспроизведение по id: `0x8002DF78`; таблица громкостей из 64 записей × 10 байт по `0x8006ADF0`. В скриптах id идут со сдвигом 100. Заголовок после `SShd`: `+0` длина, `+4` размер ADPCM, `+12` метка, `+20` число 128, затем мини-последовательности (по 9 байт на звук, заканчиваются `… ff 2f 00`) и таблица 16-байтных записей с шагом 0x100. Эта таблица тонов **не расшифрована** ❓.

Самые частые общие наборы: id 3011 (4432 байта, 194 карты), 1001–1007 (24 320 байт, 187 карт), 3021, 3071/4611, 2551, 2461, 2681, 2171–2173. Звуки врагов идут по `1001 + 10·тип`, звуки боссов — по `10000 + 100·босс + n`.

В основном EXE есть ещё фиксированный системный банк (`0x8006B1A0`, 4112 байт ADPCM) и 14-страничный отладочный экран звука (`0x8002E474`).

---

<a id="ru-15"></a>
## 15. Мини-игры

Все три стоят **10 медалей за партию** и живут в GAME01–GAME03 (модули 3–5). Запускаются с карт 494 / 493 / 495 командой 198.

### 15.1 GAME01 (забег за шариками) 🟡

Собрать всё на каждой стадии за отведённое время; стадии идут по порядку, награда растёт с ними. Слово состояния `0x8008F4B4`: 7 заставка/ввод, 8 вопрос «начать?», 9 оплата (медали `[0x80067708]` ≥ цены `[0x8008C61C] = 10`, списание `0x8001D338`), 10 игра, 13 пауза, 6 после, 12 не хватает медалей. Тексты 7301–7311.

Таблица стадий `0x8008C4AC`: 12 строк (секунды, пул): (40,20) (30,20) (30,20) (50,20) (30,30) (40,30) (40,30) (40,30) (30,30) (40,30) (30,30) (50,30). Остаток времени = `секунды × 60 − счётчик кадров`. Пул утекает по 3 за кадр в слово награды `0x800A6C94`; сумма по 12 стадиям = 320. Лучшее время лежит в сохранении по `+0xBF8` (начинается с 999,0 с). Не проверено: как в итоге выплачиваются медали. В тестовом запуске дошли только до экрана «начать?».

### 15.2 GAME02 (стрелялка, 50 стадий) ✅/🟡

Экран 256×~160. Данные стадии: `u16 N`, затем `N × (u16 вид 1..9, u16 x, u16 y)`; таблица из 51 указателя по `0x800AE688` (стадии 0 и 1 делят данные). Потолок очков 9 999 999. После стадии прибавляются остаток времени, HP и бонус стадии (бонус около 1000 + 100·n; каждая пятая стадия крупная: 15 000, 25 000, 35 000 … до 65 000 на 50-й).

Таблицы по виду: очки `0x8008BB30` (виды 1–3: 3, вид 4: 80, 5–15: 10), прочность `0x8008BB70` (вид 4: 8 попаданий, остальные 1), размер `0x8008BBB4` (вид 4: 20, остальные 12). Медали в конце = очки ÷ 1000.

Измерено на запущенной игре (единица скорости 4096 = 1 px/кадр): вид 1 идёт на игрока со скоростью 0,25 px/кадр; виды 2/3/8/9 блуждают со скоростью 1,0 px/кадр и меняют курс примерно раз в 64 кадра; вид 4 неподвижен (стена из 57 штук на стадии 20); вид 5 сходится к игроку и останавливается кучей; вид 6 разлетается со скоростью около 3,1 px/кадр и затухает ×127/128 за кадр; вид 7 расходится веером из ряда со скоростью 1,0 px/кадр. Выстрелы игрока — 7 строк по 24 байта по `0x8008BA6C` (тройные выстрелы под 120°). Игрок стартует в (128, 80), на партию три попытки, за каждую теряется одно очко «FLINT».

Не расшифровано: правило поворота видов 2/3/8/9, как стреляет вид 4, поле `+0x28`.

### 15.3 GAME03 (сеточная аркада) 🟡

Тайловая аркада с плитками по 16 пикселей: поле из пней, растений и одного преследователя. 12 строк стадий по 28 байт по `0x8008A970`: `w0` вид 0–3, `w1,w2` размер поля в клетках по 48 px (3×3 во всех 12 строках), затем **две стартовые клетки** `(w3,w4)` и `(w5,w6)` движущихся объектов (−1 = нет). Объекты лежат в 32-байтных записях по `+0x2DB4` структуры состояния, каждый кадр их двигает `0x80089828`. Медали = очки ÷ 10. Таблицы-лестницы: `0, 100, 200 … 1100` и `(100,1) (200,2) … (1100,11)`. Игрок стартует в (112, 80) и ходит шагами по 16. Правила поняты лишь частично.

---

<a id="ru-16"></a>
## 16. Ролики (EVENT00, PAKVCAST)

### 16.1 Таблица событий ✅

Таблица EVENT00 по `0x800B00DC`, записи по 36 байт, кончается id 9999. **104 события**: 75 с номерами 101–278 (в основном 101–189), 24 варианта «B» с номерами +400 (501, 502, 506…) и 5 особых (740–754, 954). Запись: номер события, четыре индекса записей PAKVCAST `[текстуры, данные сцены, звуковые эффекты, музыка или −1]`, затем четыре строки путей (`\vcast\vcast_NNN\vcasttex.NNN`, `vcast.NNN`, `vcast_NNN.se`, `vcast_NNN.bgm`). Таблица ссылается на 286 из 311 записей PAKVCAST. Поиск: `0x800880B0`. Новая игра стартует на 101; отладочная страница VCast и команда скрипта 192 используют тот же вход.

### 16.2 Содержимое записей

Запись пака — контейнер: первое слово = длина заголовка, затем смещения блоков.

- **Запись текстур:** блоки `01 MOD` (§4.4) по пять, плюс наборы палитр (`00 MOD`).
- **Запись данных сцены:** таблица u32-смещений (68, 476, 1328…), затем подблоки двух видов: **описания объектов** (в них имена моделей вроде `wing2`, `neji00`) и **потоки команд**. Ещё блоки `FRITTO` и `SENTEI`: **3D-модели сцены**. Блок = каталог смещений с шагом 540 байт, каждая «часть» = 540 байт: шесть s16 (положение/поворот), матрица 3×3 по `+0x14` (4096 = 1,0), 16-байтное имя по `+0x34`, затем готовые пакеты GPU (четыре индекса, цвета с `0x3E`, UV, слова палитры/страницы). Список вершин, формат анимации и связь текстур с моделями не расшифрованы ❓.
- **Запись звука:** один блок по `0x44` с `SShd`.

Метки (`MOD`, `FRITTO`, `SShd`) нигде в коде не сравниваются; формат определяется порядком блоков.

### 16.3 Язык команд сцен ✅/🟡

Интерпретатор в EVENT00 (цикл `0x80089544`). Команда = `u16 кадр` (выполняется, когда счётчик кадров сцены `[0x800E980C]` дошёл до него) + `u16 код (0..76)` + аргументы. Таблица переходов `0x80084B5C` (77 слов); код вне диапазона печатает «BlockID Error». Состояние сцены: 16 слотов по 1728 байт с `0x800D27F8`. Поток кончается **кодом 22**.

Подбор по данным дал **134 потока и 3165 команд**. Длины (байт): 1→12, 3→6, 5→6, 6→10, 7→10, 10→8, 13→10, 16→12, 25→12, 26→8, 27→6, 28→8, 30→10, 31→14, 49→10, 53→10, 55→8, 56→16, 57→6, 61→10, 68→10, 70→8, 71→10, 73→6, 76→6; коды 38/39/41/43/65/66 несут вставку: `12 + чётное(u16 по +10)` (38, 41, 43), `6 + чётное(u16 по +4)` (39, 65), `8 + чётное(u16 по +6)` (66).

Смысл по коду: **6** = путь камеры (кривая по индексу, длительность в кадрах, бит 15 = признак; считается сплайном `0x80029884`, результат ×4096 в слот по `+0x6C..+0x74`); **7** = то же для точки, на которую смотрит камера; **1** = установка глобальных параметров сцены, подкоманда в первом аргументе (2: три глобала, 1: знаковая пара, 8, 10/11 (задать/сбросить), 6 (три угла → вектор направления, вероятно света), 3); **10** = играть анимацию скелета (зацикливание в бите 0 второго аргумента); **13** = выбрать текущий фон/модель (110 использований); **26** = переключить блок данных объекта (чередуются 56 и 57, мигание); **30/31** = затемнения и вспышки 🟡; **49** = звук; коды 47/48/60 = заглушки с отладочными строками (`xaplay`, `xaseek`, `Depart`).

Около 35 из 77 кодов пока безымянны, а десять сцен (id 119, 147, 602, 601, 604, 228, 671, 272, 672, 277) с такими длинами на потоки не разбились.

---

<a id="ru-17"></a>
## 17. Наследие разработчиков

Практически весь отладочный код разработчиков остался в розничном диске: его держат выключенным одно слово и один отсутствующий файл.

### 17.1 Замок ✅

Слово `0x800608D4` пишется один раз в `0x8001A9F8` из `lui v0,0 ; addiu v0,v0,1` (значение 1). Читается лишь в GAME00: `0x80089A0C` (вызов HUD/меню из главного цикла `0x8008968C`, только если слово равно `0`) и `0x8008A100` (пауза/шаг). Если поменять инструкцию по `0x8001A9F4` с `0x24420001` на `0x24420000`, отладочный слой оживает (смещение в файле `SLUS_010.17` — `0x41F4`).

### 17.2 `a2profile.ini` ✅

Разборщик `0x80019BB8…0x8001A060`. Файл `a2profile.ini` открывается **без префикса `cdrom:`**; на розничной консоли открытие падает, и все флаги остаются нулями. Если файл открылся, а ключа нет — флаг ВКЛЮЧАЕТСЯ. Формат: `ключ=число`.

| Ключ | Куда | Смысл |
|---|---|---|
| `itoinf` | `0x80060828` | HUD включён |
| `jmp` | `0x80060829` | режим страницы DebugJmp |
| `itodebug` | `0x8006082A` | меню включено |
| `memhd` | `0x8006082B` | устройство Save/Load: «HD» или «MEM» |
| `vcast` | `0x8006082C` | флаг роликов |
| `childno` | `0x8006082D` | начальный пункт стартового меню |
| `dmenu` | `0x8006082E` | показывать стартовое меню модулей |
| `xa` | `0x8006082F` | тест XA |
| `datacd` | `0x80060830` | читать данные с CD или с машины разработчика |
| `bgmvol`, `sevol` | `0x80060022`, `0x80060024` | громкости |
| `startmap` | `0x800676DC` | стартовая карта (по умолчанию 213) |
| `worldcnt` | `0x800676D8` | шаг обзорной карты |
| `item=a,b,c` | | выдать предметы по id (эффекты применяются сразу) |
| `hp=N`, `ep=N` | `+0xB48/B4A`, `+0xB4C/B4E` | HP и вторая шкала |
| `comblvl=N` | `0x80067726` | уровень комбо (4, если ключа нет) |

(`childno` принимает число, но *стартовую карту* задаёт `startmap`.)

### 17.3 HUD и меню на втором геймпаде ✅ код / 🟡 поведение

Структуры контроллеров `0x80061EF8` (пад 1) и `0x80061F10` (пад 2) заполняет `0x8001BF30`; отладочный код читает **пад 2**. `SELECT` переключает HUD, `START` — меню. HUD печатает `CPU%03d GPU%03d %03d`, `OBJ%03d`, положение игрока, `ATB%04x` и «Prim Over», когда буфер примитивов почти полон. В меню 13 страниц (вверх/вниз), `Квадрат` выполняет:

0 EvtMsg · 1 VCast · 2 Camera Control · 3 Camera Mode (Normal/Yoko/Sita) · 4 Light (грузит `bin\light.tmp`) · 5 Snap Shot (пишет `d:\a2\bin\a2.tim`) · 6 Touch & Atk · 7 Save · 8 Load · 9 Audio (14-страничный экран звука) · 10 DebugJmp · **11 Flg** (4-значный номер флага, Квадрат переключает его через `0x8001D454`) · **12 Map Change** (3-значный номер карты через `0x80095520`).

Пауза/шаг: удерживать `R1` на паде 2 — кадр замирает и перерисовывается, `R2` — шаг на один кадр. Раскладка битов кнопок принята по стандартному libpad.

### 17.4 Стартовое меню (`dmenu`) 🟡

`0x800888C0` на паде 1: вверх/вниз выбирают модуль, `Крест` подтверждает, набранное число уходит в `0x800608CC` и становится аргументом модуля (для пункта 1 — номер события `0101…0274`). Список печатает имена файлов из таблицы модулей. Патч «всегда включено»: GAME00 `0x80089714`, `0x9043082E` → `0x24030001`.

---

<a id="ru-18"></a>
## 18. Граф обзорной карты ✅/🟡

Объект, которым управляет `0x800C3040` (23 состояния, объект `+0x64`), — **курсор по карте из 24 узлов**, а не обычная дверь. Он читает крестовину, берёт 4-байтную запись соседа (0/4/8/12 по направлению) в 16-байтной строке узла (таблица указателей `0x800F263C`, данные `0x800F233C`, второй набор узлов `0x800F24BC`): `b0` соседний узел (`0xFF` = пути нет; открыт, только если его бит стоит в маске `+0xB0C`), `b1` направление вдоль пути, `b2` номер пути (смещения в `0x800F21E0`, 23 пути, сплайн считает `0x80029884` из буфера по `0x801DD1B0`), `b3` **индекс выхода** (→ `obj+0x70`).

Индекс выхода (0–37) идёт в `0x800954B8`, которая заполняет структуру смены карты `0x8012B748` из двух таблиц: номер области из `0x800EE600` (`t1`) и (карта, вход) из `0x800EE64C` (`t2`). Для всех 33 настоящих рёбер `t1[выход]` равен номеру соседнего узла. Пары карта/вход 0–37:
(20,1) (20,1) (30,1) (32,1) (12,1) (20,1) (43,1) (42,1) (50,1) (50,2) (50,3) (334,1) (295,1) (295,2) (296,1) (39,1) (20,1) (298,1) (389,1) (294,1) (425,1) (207,1) (422,1) (20,1) (203,1) (20,1) (216,1) (483,1) (415,1) (221,1) (410,1) (285,1) (490,1) (140,1) (214,1) (218,1) (340,1) (2,0).

Второй набор узлов заполняет только узлы 8–12 (и связь с 19). Обычные переходы по границам карт идут не через эту таблицу, раз в ней всего 38 записей; как устроены обычные выходы в соседние карты, не выяснено ❓. Объекты-двери работают через `0x800C4BA0(номер, режим)`, которая ставит бит `1 << номер` в `+0xB0C`; `worldcnt ≥ 25` сбрасывает цепочку. Значит `worldcnt` — шаг в цепочке из 25 этапов, а `+0xB0C` — маска пройденных узлов. Две таблицы по `0x800F2210` (26 × 4 байта) и `0x800F2274` (8 байт на шаг) описывают узлы и маски флагов.

Тот же загрузчик задаёт стартовую позицию: из сохранения копирует карту `+0xB14`, положение `+0xB50/54/58`, направление `+0xB5C`, `+0xB64`; в новой игре берёт карту из `0x800676DC`.

---

<a id="ru-19"></a>
## 19. Нерешённое

То, что пока неизвестно, чтобы не гадать, покрыто ли это:

- Кто пишет `0x8006746E` (вариант страницы героя, 7 значений) и что оно значит; что выбирает вариант меча/щита (`PC00_KEN` / `PC00_TATE`).
- Где лежат записи материалов для `t ≠ 0` (≈ 40 % текстурных полигонов); бит 9 флагов; байт 13 вершин; причина смешанной намотки.
- Смысл слотов 1, 12 и 13 (небо, купол, освещение — выводы по косвенным признакам); слова 126–139 таблицы слота 11; таблица тонов в `SShd`.
- Около 140 команд скриптов без имён; оставшиеся 525 потоков, которые не разбираются; у команд 147, 177, 179, 186 длина неизвестна.
- Остальные команды байткода ИИ (40+ безымянных), несколько подзапросов `C0`; устройство босса B24; что делают руки у B25; числа урона, измеренные в бою.
- Обычные переходы между соседними картами.
- Модели сцен (`FRITTO`/`SENTEI`) дальше каталога частей, десять неразобранных сцен и оставшиеся коды сцен.
- Мини-игра 1 в тестовых запусках застревает на вопросе «начать?»; детали поворота врагов GAME02 и правила GAME03.
- Системные строки в EXE и GAME01–03.
- В эмуляторе вообще не проверялись: патч кириллицы, патч отладочного HUD и патч dmenu. Они выведены только из кода.

---

<a id="ru-20"></a>
## 20. О методе и юридическое

Работа велась с самодельным дизассемблером MIPS R3000, небольшим интерпретатором MIPS+GTE, Python-парсерами для каждого контейнера и эмулятором PS1, из которого снимались дампы RAM, VRAM и команд GPU запущенной игры. Инструментов в этом репозитории нет. Всё здесь получено на собственной копии диска автора.

**Ресурсы игры (текст, текстуры, модели, звук) в репозиторий не включены**, и добавлять их не нужно. Если вы повторяете что-то из этого, работайте с купленным диском и делитесь патчами (диффами), а не изменёнными образами.

*Alundra 2 принадлежит соответствующим правообладателям (Activision и Matrix Software). Проект — фанатская документация и никак с ними не связан.*

---

[⬆ наверх](#top) · [🇬🇧 English](#en)
