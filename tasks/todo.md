# Plan — review fixes (2026-04-25)

Scope agreed with user: only the two highest-impact items from the review.

## 1. Fixed-timestep simulation loop
- [x] Refactor `loop()` to accumulator pattern at 60 Hz (FIXED_DT = 1000/60 ms)
- [x] Clamp frame delta to 250 ms to avoid spiral-of-death after tab-switch
- [x] Make `loop` accept the RAF timestamp; kick off via `requestAnimationFrame(loop)` from `startGame`
- [x] `update()` already early-returns when `!state.running`; no change needed there

## 2. Heart HUD rework
- [x] Drop `player.maxHealth` 100 → 20 (5 hearts × 4 quarters); damage values stay
- [x] Replace `.heart` CSS: `--fill` CSS var + linear-gradient for quarter-fill states
- [x] Split `updateHUD` into one-time `buildHUD()` + per-tick `updateHUD()`
- [x] Call `buildHUD()` from `startGame` before `updateHUD`
- [x] Fix low-health pulse threshold in `drawVignette`: `health <= 4`

## Review

### What changed (index.html)
- **CSS `.hearts` / `.heart`** (~159–178): hearts now 22 px (26 px on mobile), driven by `--fill` CSS variable; linear-gradient splits red/dark fill at the variable's percentage so each heart can render full / ¾ / ½ / ¼ / empty cleanly. Inset highlight via `box-shadow` keeps the cel-shaded look without a per-fill radial gradient.
- **`player.health` / `player.maxHealth`** (549–550): 100 → 20. Existing damage values (moblin/lizalfos/boss = 2, keese = 1) now read as ½ heart and ¼ heart respectively. Roughly 10 moblin hits to die — same survivability as before, but legible on the HUD.
- **`buildHUD()` + `updateHUD()`** (~1613–1637): `buildHUD` runs once on game start and creates 5 heart nodes; `updateHUD` only sets `--fill` per heart on each change. Eliminates the per-tick `innerHTML = ''` + 100× `createElement`.
- **`drawVignette` low-health pulse** (~1605): threshold raised from `<= 2` to `<= 4` so the screen-tint warning fires at 1 heart remaining (the original intent — under the old `<= 2` it was effectively dead code).
- **`startGame`**: calls `buildHUD()` before `updateHUD()`; kicks off the loop via `requestAnimationFrame(loop)` so the RAF timestamp arrives in `loop`.
- **`loop()`** (~1647–1666): rewritten to fixed-timestep accumulator at 60 Hz. Logic ticks (`update()`) decoupled from refresh rate; rendering still runs once per RAF. Frame delta clamped to 250 ms so a long tab-switch doesn't trigger a spiral-of-death catch-up.

### Behavior delta
- 144 Hz / 120 Hz monitors no longer fast-forward gameplay — player speed, stamina drain, attack cooldowns, and enemy AI now run at intended 60 Hz on every refresh rate.
- HUD shows 5 large hearts instead of 100 tiny ones. Damage in ¼-heart steps reads correctly.
- All other behavior preserved: shrine heals to full (20), sword reach/cooldown unchanged, boss fight identical, persistence (Champion's Tunic) untouched.

### Verification
- Re-read all four edit sites; no orphaned references to `maxHealth`-as-100 or to the old `.heart.empty` class.
- No new globals leak — `FIXED_DT`, `lastTime`, `accumulator`, `HP_PER_HEART` all scoped at module level alongside existing module-level state.
- Did **not** open the file in a browser; visual confirmation (heart sprites, screen-shake, sword-arc rendering) is the user's to verify. Recommend a quick playtest: start game, walk into a moblin, watch hearts decrement in halves; dash with Shift to confirm motion feels right.

### Not done (intentional, from review)
- Pre-rendering tiles to offscreen canvas
- Adding sound
- Resolving the async storage race
- Audio, accessibility, pause menu
These remain in the original review for follow-up.
