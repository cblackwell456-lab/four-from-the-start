# Changes

## Versions
- **1.0** (2026-10-15): a late joiner starts with player 1's weapon level; players 3/4 fire the cannon in Impact
  battles; the camera zooms out while three or four players spread. Tested with eight automated scenarios and a
  whole-game survey.
- **0.3** (2026-10-13): small food boxes at the screen edges; the BOSS bar moved below the P3/P4 panels; join as the
  next free player; Sasuke/Yae selectable only while three or four play (journals keep the story's own count).
- **0.2** (2026-10-13): four players from the title screen (2P with three or four controllers); HUD panels for
  players 3 and 4; a food slot per player.
- **0.1** (2026-10-13): drop-in for controllers 3/4 without the 44 passes; players 3/4 kept across area changes;
  all four characters unlocked on a join (fixes a GAME OVER soft-lock).

## The changes, in words
Offsets are for the rommy-decompressed image; the retail patch is "decompress → change these words → recompress",
done with rommy (StopSine/gga's compression tool). Files are those of the game's Nisitenma-Ichigo file
system: file 8 = gameplay code (VRAM `0x801738A0`), file 5 = engine code (`0x800C7B10`), file 9 = Impact battle
code (also loaded at `0x801738A0`, during Impact), file 17 = front end (overlay at `0x08000000`).

**Joining and keeping P3/P4** (file 8; unchanged from v0.1):

| Purpose | VRAM | Decompressed offset | Original | Patched |
|---|---|---|---|---|
| ignore the "4-player mode" flag (`0x80088333`) | `0x8018C384` | `0x5EEDE4` | `81CE8333` `LB T6,-0x7CCD(T6)` | `240E0001` `ADDIU T6,ZERO,1` |
| ignore the 44-pass check | `0x8018C3C8` | `0x5EEE28` | `2881002C` `SLTI AT,A0,0x2C` | `28810000` `SLTI AT,A0,0` |
| join as a normal player, not a guest | `0x8018C5FC` | `0x5EF05C` | `37280100` `ORI T0,T9,0x100` | `37280000` `ORI T0,T9,0` |
| unlock all four characters on join (1/2) | `0x8018C5B0` | `0x5EF010` | `A0600081` `SB ZERO,0x81(V1)` | `240A0004` `ADDIU T2,ZERO,4` |
| unlock all four characters on join (2/2) | `0x8018C5B4` | `0x5EF014` | `A0600082` `SB ZERO,0x82(V1)` | `A10A0009` `SB T2,0x9(T0)` |

The last pair writes 4 to the available-character count `0x80088211`. It reuses the two stores that cleared the
joining player's weapon and food bytes, so a player who re-joins keeps whatever weapon/food that slot had.

**Four from the start** (files 17 and 5). New code is placed in file 5's unused 3×3 matrix-inverse routine
(`0x800D318C`) and its unused determinant helper (`0x800D2CD8`); nothing in the game calls either.

| Purpose | VRAM | Decompressed offset | Change |
|---|---|---|---|
| 1P/2P confirmed: set the cursor count | `0x080056EC` | `0x66458C` | 5 words → call `0x800D318C`: 1 for 1P; for 2P, 2 + 1 if controller 3 is connected + 1 more if controller 4 is too (connected = bit 0 of `0x801736C0 + pad*0x1E`) |
| after the journal is copied in | `0x080002FC` | `0x65F19C` | 5 words → call `0x800D31D4`: with 3 or more cursors, set the character count `0x80088211` to 4 |
| the two routines | `0x800D318C` | `0x58E3AC` | 0x7C bytes of new code |

**HUD** (file 5):

| Purpose | VRAM | Decompressed offset | Change |
|---|---|---|---|
| draw elements for 4 panels, not 2 | `0x800DE028`, `0x800DE058`, `0x800DE094`, `0x800DE1A4` | `0x599248`, `0x599278`, `0x5992B4`, `0x5993C4` | `2A010002` `SLTI AT,S0,2` → `2A010004` `SLTI AT,S0,4` |
| (coins loop) | `0x800DE1D0` | `0x5993F0` | `24110002` `ADDIU S1,ZERO,2` → `24110004` |
| panel draw call → wrapper | `0x800DE578` | `0x599798` | `0C0377F2` `JAL 0x800DDFC8` → `0C034C82` `JAL 0x800D3208` |
| food draw call → wrapper | `0x800DE454` | `0x599674` | `0C0377D9` `JAL 0x800DDF64` → `0C034B36` `JAL 0x800D2CD8` |
| panel wrapper + P3/P4 panel lists | `0x800D3208` | `0x58E428` | 0x2F8 bytes: if stat row 3 (4) has a character with lives left, draw the P3 (P4) panel background; call the original; move each P3/P4 element drawn off-screen right to the top (254 px left, or 508 px for the weapon icon, and 188 px up); drop the elements of an absent row |
| food wrapper + box lists | `0x800D2CD8` | `0x58DEF8` | 0x318 bytes: call the original, then draw the P3/P4 food icons; with row 3 (4) present, switch to a box list with small (20×17 px) boxes at that side's screen edge (P1/P2's above the bottom panel, P3/P4's under the top panel) and draw the icons 14×14 inside them (v0.3) |

**v0.3 additions** (new code over two unused file-5 functions, `0x800DA1B4` and `0x800E6508`, and after the panel
code in the inverse; the determinant tail at `0x800D2FF0` is left free):

| Purpose | VRAM | Decompressed offset | Change |
|---|---|---|---|
| journal hook also records the story's character count | `0x800d31e0` (in `start4_chars`) | `0x58e400` | jump to `story_journal` (`0x800da1b4`): store the loaded count, reset the pad map, then raise the count to 4 with 3+ cursors as before |
| both journal saves → `save_wrap` | `0x800EDEBC`, `0x800EE010` | `0x5a90dc`, `0x5a9230` | `0C033F19` `JAL 0x800CFC64` → `JAL 0x800da204`: when the live count is the raised 4, save max(loaded count, 3 if Sasuke's story flag `0x80088317` bit 7, 4 if Yae's `0x8008831A`), then restore 4 |
| pads re-ordered where they are read | `0x800D8800` | `0x593a20` | `00850019` `MULTU A0,A1` → `JAL 0x800e6508` (6 × `pad_map[pad]`; hides a joining controller's Start until it is released) |
| join as the next free player | `0x8018C354` (file 8) | `0x5eedb4` | `0C0630E0` `JAL player_join` → `JAL 0x800e6568` (`join_next`: lowest empty slot, swap the two `pad_map` entries, then `player_join(slot)`) |
| boss gauge y | `0x801A2EF8` (file 8) | `0x605958` | `240D0017` `ADDIU T5,ZERO,0x17` → `JAL 0x800D3500` (y = 0x17, or 0x17 + 40 with row 3 or 4 present) |
| new code / data | `0x800DA1B4`–`0x800DA284`, `0x800E6508`–`0x800E65D4`, `0x800D3500`–`0x800D354C` | | the story, join and gauge routines (`pad_map` and `suppress` at `0x800D3544`) |

**v1.0 additions** (new code over one more unused file-5 function and in the free space after the v0.3 code):

| Purpose | VRAM | Decompressed offset | Change |
|---|---|---|---|
| a late joiner gets P1's weapon level | `0x8018C5B8` (file 8) | `0x5ef018` | `AFA30018` `SW V1,0x18(SP)` → `JAL 0x800e65d4` (`join_weapon`: does that store and copies row 1's weapon byte `0x80088289` to the joiner's row) |
| Impact: P3/P4 fire the cannon | `0x80177E14`, `0x80177F0C` (file 9) | `0x61ec74`, `0x61ed6c` | `0C03633E` `JAL hold_time` → `JAL 0x800da284` (`gun_hold`: as before, plus a shot for each new P3/P4 Z press read from the input ring when P1 is not firing) |
| camera zoom with 3–4 players | `0x800D87B8` | `0x5939d8` | `0C004569` `JAL osContGetReadData` → `JAL 0x800f69fc` (`zoom_frame`: makes that call, then sets the stage camera's field of view, camera `+0x30`, to the game's value × a zoom of 1.0–1.25 eased toward spread / (0.75 × view width) with 3–4 players, 1.0 otherwise; the co-op leash is the camera's view width, so it widens with it) |
| new code / data | `0x800E65D4`–`0x800E65EC`, `0x800DA284`–`0x800DA310`, `0x800F69FC`–`0x800F6C84` | | the join-weapon, gunner (press state at `0x800E65E8`) and camera-zoom routines (over the unused function `0x800F69FC`) |

The P3/P4 panel backgrounds and the moved food boxes are copies of the game's own P1/P2 display lists with new
screen positions.
