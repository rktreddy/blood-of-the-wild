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

---

# IP Cleanup Pass (2026-04-25)

Goal: strip Zelda-specific names and iconic visuals so the game stops being a Nintendo fan project.

## Renames

| Original (Zelda IP) | Replacement |
|---|---|
| Blood of the Wild | Wake of Embers |
| Hyrule | Aeloria |
| The Calamity | The Sundering |
| Ganon's Remnant | The Hollow King |
| Champion's Tunic | Vowsworn Mantle |
| The Champion | The Vowsworn |
| Rupees / Rupee | Shards / Shard |
| Moblin | Tuskling |
| Lizalfos | Skarn |
| Keese | Vesper |
| Sheikah eye/shrine | Flame sigil on stone shrine |
| `botw_hasWon` localStorage | `embers_hasWon` |

Internal symbol renames followed: `state.rupees` → `state.shards`, `dropRupee` → `dropShard`, `tunicMain/Hi` → `mantleMain/Hi`, etc. `state.blueClothes` left as-is (descriptive of the visual, not IP).

## Visual reskins
- **Player**: dropped pointy ear, dropped elf-cap point. Replaced with hooded silhouette (rounded crown + side drapes). Sheikah eye on chest replaced with three vertical strokes.
- **Sword**: blade pale steel, cross-guard iron grey, grip dark leather, pommel iron — was Master Sword blue+gold.
- **Shrine**: kept teal stone tower, replaced orange Sheikah eye + teardrop with a two-layer flame sigil.

## Verified
Grepped for `Hyrule|Ganon|Sheikah|BOTW|Tunic|Master Sword|Rupee|Champion|Nintendo|botw_|Hylian|Calamity|Moblin|Lizalfos|Keese` (case-insensitive) — zero matches remain in `index.html`.

## Caveats
- localStorage key change wipes any prior win record — the previously-earned Vowsworn Mantle won't carry over from before this pass. Acceptable since the project is brand new.
- Filename / repo name `blood-of-the-wild` and `README.md` unchanged — out of scope; rename outside if desired.
- I did not rebrand the cyan slash-arc trail color since cyan slash trails are a generic action-game effect, not Zelda-specific.

## Lessons (for tasks/lessons.md if you keep one)
- When doing IP cleanup on a clone, don't stop at the obvious user-visible strings — internal variable names, localStorage keys, comments, and enemy type IDs are all visible to anyone who opens devtools and contribute to the "fan game" identity.
- A grep sweep at the end is mandatory; finding the residual `Calamity` and the lowercase enemy type strings only happened because of the post-pass scan.

---

# Plan — second review fixes (2026-04-25)

Scope: the 🟡 "Important" batch from the second review pass. Skipping 🔴 #1 (async storage race) — already deferred in prior plan. Skipping 🟢 cosmetic items.

## Items

### 1. `wanderX` / `wanderY` initialized in `makeEnemy` (`index.html:589–602`, used at `:902–905`)
- [x] Add `wanderX: 0, wanderY: 0, wandering: false` to the object literal returned by `makeEnemy`
- [x] In the idle block (`:897–905`), set `e.wandering = true` when a new direction is rolled
- [x] Change `if (e.wanderX)` → `if (e.wandering)` so a legitimate `0` doesn't skip movement

### 2. Reduce particle burst counts in hot paths
- [x] Per-hit on enemy (`damageEnemy`): 8+6 → 5+4
- [x] On enemy death: 15+10+8 → 8+6+5
- [x] Boss hit: 15+8 → 10+6. Boss-death 60+40+30 cinematic left as-is.

### 3. Centralize `updateHUD()` calls
- [x] Removed from `damageEnemy`, `hurtPlayer`, `updateShardDrops`
- [x] Added single call at end of `update()`

### 4. Cache HUD textContent / width writes
- [x] Module-level `hudCache` introduced
- [x] `updateHUD` skips DOM writes when value unchanged (hearts, stamina, shards, kills)
- [x] `buildHUD` resets cache so a new game forces redraw

### 5. Track and clear cinematic `setTimeout`s
- [x] Module-level `cinematicTimers` array + `clearCinematicTimers()` helper
- [x] All three cinematic `setTimeout`s wrapped via `cinematicTimers.push(...)`
- [x] `clearCinematicTimers()` called from `startGame()`; also reset `game-over-screen` display
- **Discovered during impl:** the only existing restart path is `riseAgain()` → `location.reload()`, which already nukes pending timers. So this is currently future-proofing rather than fixing an active bug. Kept anyway — small, defensive, makes the lifecycle explicit, and stops a future "in-place restart" feature from regressing.

### 6. Delete dead arithmetic in `doSwordHit`
- [x] Removed `hy -= 0;` (up-branch) and `hy += 0;` (down-branch)

## Out of scope (intentionally)
- Particle pooling (item 2 chose count reduction instead — simpler)
- 🟢 batch (camera flooring, screen-shake curve, magic-number constants, world-gen weights) — defer; cleanup-when-touching
- Async storage race (carried over from prior plan; unchanged)

## Verification plan
- After edits, grep for `updateHUD()` to confirm only one call site outside the function definition + `startGame`/`buildHUD` callsite
- Grep `setTimeout(` to confirm only the three cinematic sites use the tracked array
- Grep `e.wanderX` / `e.wanderY` to confirm no remaining `if (e.wanderX)` checks
- Manual visual check is the user's — I'll list what to look for: hearts decrement smoothly, kill kills look less particle-heavy, restart-during-victory works cleanly

## Review

### What changed (`index.html`)
- **`makeEnemy` (~589–602):** added `wanderX: 0, wanderY: 0, wandering: false` to the returned object so idle-state checks have defined values from frame 1.
- **`updateEnemies` idle branch (~907–917):** sets `e.wandering = true` when a direction is rolled; movement check is now `if (e.wandering)` — a legitimately-zero velocity component won't short-circuit movement on the other axis any more.
- **`doSwordHit` (~782–783):** removed `hy -= 0` / `hy += 0` no-ops.
- **`damageEnemy` (~805–814):** particle bursts trimmed (8+6 → 5+4 per hit; 15+10+8 → 8+6+5 on kill); removed `updateHUD()` call from the kill branch.
- **`damageBoss` (~827–828):** boss-hit bursts trimmed 15+8 → 10+6; victory cinematic bursts (60+40+30) preserved. Both cinematic `setTimeout`s now tracked via `cinematicTimers.push(...)`.
- **`hurtPlayer` (~951):** removed `updateHUD()`; death-cinematic `setTimeout` wrapped via `cinematicTimers.push(...)`.
- **`updateShardDrops` (~1006):** removed `updateHUD()` from shard pickup.
- **`update()` (~702):** added one `updateHUD()` at the end so HUD reflects all state changes for the tick.
- **HUD (~1617–1660):** introduced module-level `hudCache` (shards / kills / stamina / per-heart fill). `updateHUD` now compares each value to its cached prior and only writes the DOM on change. `buildHUD` resets the cache when a new game starts.
- **Cinematic timer cleanup (~606):** new `cinematicTimers` array + `clearCinematicTimers()` helper, scoped just above the boss block.
- **`startGame()` (~1712):** calls `clearCinematicTimers()` and resets `game-over-screen` display before rebuilding HUD.

### Behavior delta
- Idle enemies start wandering correctly on first qualifying frame instead of standing still on a (rare) angle that produced `cos(ang) ≈ 0`.
- Particle effects still read as "hit" / "kill" / "boss hit" — just leaner (~50–55% fewer particles in hot paths). Boss-death cinematic unchanged.
- HUD updates once per logic tick from a single site. With the cache, a still moment (no damage taken, no shards picked up, full stamina) does zero DOM writes. During active gameplay, only the values that changed touch the DOM.
- Cinematic timer tracking is currently dormant (restart goes through `location.reload()`); doesn't change behavior today.

### Verification
- `grep updateHUD\(\)` → 3 results: definition (1644), one call site at end of `update()` (713), one in `startGame()` (1723). ✓
- `grep setTimeout\(` → 4 results: 3 cinematic (851/857/976) all wrapped in `cinematicTimers.push`; 1 in `showMessage` (1675) intentionally not wrapped — it self-clears via its own `_t` handle. ✓
- `grep "e\.wanderX|e\.wandering"` → no remaining `if (e.wanderX)` truthy-check; movement gated on `e.wandering`. ✓
- `grep "hy [+\-]= 0"` → no matches. ✓
- Did **not** open the file in a browser. User playtest recommended: kill an enemy and confirm puff-of-smoke still reads as a kill (just smaller); take damage and confirm hearts decrement; pick up shards and confirm count updates.

### Caveats / not done
- Async storage race (🔴 #1) still deferred — same status as prior plan.
- Cinematic timer cleanup is future-proofing today. The actual victory→restart path is `location.reload()`, which obliterates timers anyway. The change still makes the cinematic lifecycle explicit and prevents regressions if anyone adds an in-place restart later.
- Particle burst reduction is a number-tweak — chose this over particle pooling because the pool would be a much larger change and the GC pressure here isn't catastrophic, just wasteful.
- 🟢 batch (camera flooring, screen-shake curve, magic-number constants, world-gen weights, etc.) deferred per scope agreement.

### Lessons
- Don't trust an exploration subagent's bug list at face value — verify each finding against the code. The first review-pass agent reported "shrine bounds-unsafe" and "shrine multi-fire" — both were already guarded (`world[idx]` short-circuits to `undefined`; the heal is gated on `health < maxHealth`). Skipping verification would have led to writing fixes for non-bugs.
- When proposing a fix, sanity-check the *trigger path* before claiming "real-world bug." Item 5's pitch claimed an active bug; while implementing, the only restart path turned out to be `location.reload()`, which made the bug moot. The change is still defensible as future-proofing, but the framing should have been honest from the start.

---

# Plan — PWA scaffolding (2026-04-29)

Goal: make the GitHub Pages site installable on iOS/Android home screens, offline-capable, with a real icon. Step toward eventual Capacitor wrap.

## Steps
- [ ] `tools/generate-icons.html` — self-contained canvas script that renders the flame-sigil icon at 512/192/180 px and provides download buttons. User opens once, downloads 3 PNGs, drops them in repo root.
- [ ] `manifest.json` — name, short_name, icons, `display: fullscreen`, `orientation: landscape`, theme/background colors matching game palette.
- [ ] `sw.js` — minimal cache-first service worker; precaches `index.html`, `manifest.json`, `sw.js`, the 3 icon PNGs. Versioned cache key so updates evict cleanly.
- [ ] `index.html` `<head>` — add `<link rel="manifest">`, `theme-color`, `apple-touch-icon`, `apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style`. ~5 lines.
- [ ] `index.html` `<script>` — register service worker on load (guarded behind `'serviceWorker' in navigator`).
- [ ] User step: open `tools/generate-icons.html` in any browser, click "Download all", move PNGs to repo root.
- [ ] Commit + push.

## Out of scope
- Capacitor / native packaging (separate next step if user wants store presence)
- Audio (would need user-gesture init for iOS WebView)
- Landscape lock enforcement beyond manifest hint (manifest is advisory; iOS Safari ignores `orientation`)

## Verification
- Open `index.html` in a Chrome desktop tab → DevTools → Application → Manifest: no errors, all 3 icons resolve, no SW errors.
- On a phone: Add to Home Screen → launch from icon → confirms fullscreen, custom icon, and offline (toggle airplane mode and relaunch).
