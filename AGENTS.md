# Mech Vanguard - Project Guide

## Start Here

This project is a TIC-80 Lua action-platformer vertical slice. The working game is a single extensionless Lua file:

- Game code: `C:\Users\yazda\OneDrive\Desktop\Programming\LUA\TIC-80_MEch`
- Design document: `C:\Users\yazda\OneDrive\Documents\Obsidian Vault\2D side scroller mech game.md`
- Cartridge: `C:\Users\yazda\AppData\Roaming\com.nesbox.tic\TIC-80\game.tic`
- Reference cart: `C:\Users\yazda\AppData\Roaming\com.nesbox.tic\TIC-80\rosa.tic`
- Reference Lua: `C:\Users\yazda\OneDrive\Desktop\Programming\LUA\ROSA_example_platformer`
- Extracted cart assets / map previews: `C:\Users\yazda\AppData\Local\Temp\opencode\ticdump`

The project is not a Git repository. Do not run Git commands or commit unless the user explicitly asks.

Last known AI session:

```powershell
cd "C:\Users\yazda\OneDrive\Desktop\Programming\LUA"
opencode --session ses_e90841899ffeAvmLBaa70SKFlO
```

Add `--fork` to avoid modifying the original session.

## Non-Negotiable Constraints

- Target TIC-80: **240x136**, 16-color palette, Lua single-file code.
- Keep the game within TIC-80's free-version 64 KB code limit.
- Keep all tunables, asset IDs, and content data in the top-level singletons:
  `CFG`, `INPUT`, `TILE`, `SPR`, `SFX`, `MECHS`, `WEAPONS`, `ENEMY_TYPES`, `LEVEL1`, `ART`, `GAME`, `WORLD`, `STARS`.
- Do not hard-code asset IDs or tuning values inside gameplay functions.
- Do not overwrite the cartridge's `__gfx__`, `__map__`, `__sfx__`, `__music__`, or `__palette__` sections. Generated art is written to runtime VRAM only.
- Match the existing terse style. Do not add comments unless the user asks.
- Make surgical edits. Do not reformat or rewrite the whole file.
- The design document's code block in section 6.1 is illustrative and its IDs are stale. The actual source of truth is `TIC-80_MEch`.

## TIC-80 Sprite/Tile Memory

TIC-80 has 512 8x8 sprites:

- IDs `0-255`: background tiles, used by the map.
- IDs `256-511`: foreground sprites, used by `spr`.
- Universal sprite/tile address: `0x4000 + id * 32`.
- Generated art must therefore call `load_art(..., 0x4000)`, **not** `0x6000`.
- A 16x16 sprite is a 2x2 composite: draw with `spr(id, x, y, 0, 1, flip, 0, 2, 2)`.
- A 32x16 boss is a 4x2 composite: draw with `spr(id, x, y, 0, 1, flip, 0, 4, 2)`.

Current IDs in the implementation:

- Pilot: `256-259`; grunt: `272-273`.
- Generated mech: idle `408`, walk1 `410`, walk2 `412`, thrust `414`, empty `416`.
- Turret: `440`; drone: `448`.
- Bullets: pistol `450`, vulcan `451`, spread `452`, enemy `453`, rocket `454`.
- Boss: `460`.
- Tiles: empty `87`, solid `129`, generated platform `250`, generated ladder `251`.
- `CFG.use_code_art=true`, `CFG.generate_tiles=true`, `CFG.start_title=true`.

Do not change the generated sprite bases without verifying the 2x2/4x2 blocks do not overlap. Sprite IDs use a 16-column sheet: a base and the next base two IDs later are horizontally adjacent, not overlapping.

## Current Implementation

- Level 1: six 30x17 ASCII screens joined into a 180x17 map, horizontal camera.
- States: `title`, `play`, `dead`, `gameover`, `victory`.
- Player forms: pilot (6x8, 1 HP, pistol, ladders, vents) and mech (16x16, armor, thrust fuel).
- Mechs: `vanguard` (VULCAN) and `bulwark` (SPREAD).
- Enemies: grunt, drone, turret, heavy-tank boss.
- Mechanics: mounting, manual exit, forced ejection, one-way platforms, ladders, pilot-only vent, boss trigger, HUD, particles, screen shake.
- Key functions:
  - `build_level()` parses `LEVEL1.screens` with `mset` and spawns markers.
  - `move_body()` handles tile/platform/ladder collision.
  - `update_player()`, `update_pilot()`, `update_mech()`.
  - `mount_mech()`, `eject_from_mech()`, `try_use()`.
  - `check_boss_trigger()`, `update_boss()`, `kill_enemy()`, `kill_player()`.
  - `draw_game()` and state overlays.

## Recent Fixes - Do Not Regress

- **Float collision edges:** use the right edge `x + w` and bottom edge `y + h`. Do not revert to `x + w - 1` / `y + h - 1`; that causes 1 px sinking and `grounded` flicker.
- **Manual ejection:** `eject_from_mech()` uses `p.x/p.y`, and `update_mech()` syncs `p.mech.x/y`. Do not use the stale chassis position.
- **Generated VRAM:** `load_sprites()` must pass `0x4000` to `load_art()` for IDs `256+`.
- **Boss trigger:** activates at `WORLD.player.x >= 1216`.
- **Mech exit cooldown:** `GAME.use_cd` prevents instant re-mount. Tests must wait about 24 frames before pressing use again.
- **Mount invulnerability:** mounting grants 45 invulnerable frames; damage tests must zero `p.invuln` or wait.

## Testing

There is currently **no TIC-80 executable available** on this machine. Do not claim visual/runtime verification until one is found. The game has been tested through a Python `lupa` mock:

- `lupa` is installed for Python.
- Mock the TIC-80 API: `btn`, `btnp`, `rect`, `rectb`, `line`, `circ`, `circb`, `print`, `spr`, `map`, `sfx`, `cls`, `mget`, `mset`, `peek`, `poke`.
- Initialize `MAP` as a 240x136 table of `87` and `PEEK` as an empty table.
- Execute `TIC-80_MEch`, call `reset_game()`, then simulate frames with `TIC()`.
- Lua functions that return multiple values become Python tuples under Lupa. Example: `m, d = nearest_mech()`, not `m = nearest_mech()`.

Verified so far:

- Script parses and runs without runtime errors.
- Six screens are 30x17; level is 180x17.
- Player spawns as pilot at `x=25`, `y=112`.
- First vanguard chassis at `x=92`; spare bulwark at `x=916`.
- Mounting and manual exit work.
- Mech is blocked by the screen-4 vent; pilot passes through.
- Pits are crossable with pilot and mech jumps.
- Boss activates and can be killed; victory state works.
- Player death transitions to dead, then gameover; restart works.
- Grounded state is stable after the collision fix.

Still unverified:

- Actual TIC-80 rendering, sprite animation, palette, audio, and performance.
- Full end-to-end playthrough in the real runtime.
- Title/gameover/victory visual layout.
- Final balance and audio authoring.

## Next Steps

1. Find or install a TIC-80 runtime and load `TIC-80_MEch`.
   - The file is extensionless; copy it to a `.lua` file if the TIC-80 loader requires an extension.
2. Visually verify generated mech/boss/turret/drone/bullet art, the pilot/grunt cart sprites, tiles `87/129/250/251`, and the HUD.
3. Play through: mount vanguard, cross the low obstacle and pits, enter the vent on foot, pick up the bulwark, trigger the boss, win, then test death/gameover.
4. If a bug is found, fix it surgically and re-run the mock for regressions.
5. Update the GDD only when design or IDs change; keep `TIC-80_MEch` as the implementation source of truth.
