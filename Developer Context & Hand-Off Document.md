# Developer Context & Hand-Off Document
**Project:** `SOLARIS RUN '89 — Overdrive Edition`  
**File:** `solaris_run_89_overdrive.html` (Single-file HTML5 / Three.js WebGL 3D Arcade Game)  
**Target Audience:** Incoming developer responsible for maintenance, bug fixes, performance tuning, and feature upgrades.

---

## 1. Executive Overview
**SOLARIS RUN '89** is a self-contained, zero-build-step 3D rail/trench arcade shooter inspired by *Star Fox*, *OutRun*, and *Wipeout*. Players pilot an interceptor down an infinite neon synthwave highway, dodging procedural hazards, grazing obstacles for boost, shooting enemy formations and chain-reaction mines, locking on multi-target homing missiles, barrel-rolling to deflect enemy projectiles, and defeating repeating Sector Bosses to pick roguelite-style upgrades.

### Architectural Tenets
1. **Single-File Distribution (`file://` friendly):** All HTML, CSS, procedural textures, 3D meshes, game logic, and Web Audio synthesis live inside one `.html` file. Three.js is loaded as a classic global script (not ES Modules) so the file runs directly from disk without CORS errors or a local dev server.
2. **Zero-Allocation Hot Loop:** Projectiles, particles, shockwaves, and lock-on brackets are pre-allocated in fixed-size object pools at boot to avoid garbage-collection spikes during 60 FPS gameplay.
3. **Zero External Media Assets:** All sprites/textures are generated via offscreen `<canvas>` (`CanvasTexture`), and all sound effects + 108/118 BPM synthwave music are synthesized procedurally via the Web Audio API.

---

## 2. Tech Stack & Runtime Dependencies

| Layer | Technology / Details |
| :--- | :--- |
| **3D Engine** | **Three.js r128** via CDN: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` |
| **Audio** | Native Web Audio API (`AudioContext` / `webkitAudioContext`), lazily unlocked on first `keydown` or `pointerdown`. |
| **Persistence** | `window.localStorage` wrapped in `try/catch` (`loadLS` / `saveLS`). |
| **UI / Styling** | Vanilla CSS3 with CSS variables (`:root`), frosted glassmorphism (`backdrop-filter`), responsive breakpoint at `max-width: 860px`. |
| **Code Structure** | Single `'use strict'` IIFE inside `<script>`—no global namespace pollution. |

### Three.js r128 Compatibility Constraints
If upgrading Three.js in the future (e.g., to r152+), watch for breaking API changes currently relying on **r128**:
- `renderer.outputEncoding = THREE.sRGBEncoding` (changed to `renderer.outputColorSpace = THREE.SRGBColorSpace` in r152+).
- Classic non-module UMD build (`three.min.js`) was deprecated in newer Three.js versions in favor of ES modules / import maps.
- Only r128-safe primitives are used (`BoxGeometry`, `ConeGeometry`, `CylinderGeometry`, `SphereGeometry`, `OctahedronGeometry`, `IcosahedronGeometry`, `TorusGeometry`, `RingGeometry`, `PlaneGeometry`, `EdgesGeometry`). Do **not** introduce `CapsuleGeometry` unless upgrading Three.js (added in r139).

---

## 3. Codebase & Layer Architecture Map

### 3.1 DOM & CSS Z-Index Hierarchy
| `z-index` | Element ID(s) | Purpose |
| :--- | :--- | :--- |
| Base (`0`) | `#gl` | Fullscreen WebGL `<canvas>` (`position: fixed; inset: 0`). |
| `2` | `#scanlines` | Optional subtle CRT scanline overlay (toggled via `body.nocrt`). |
| `3` | `#vig` | Dynamic screen-edge flash vignette (damage, shield, boost, EMP, overclock). |
| `4` | `#locks`, `#popups` | 3D-to-2D projected golden `LOCK` brackets and floating combat score popups. |
| `6` | `#topHud`, `#bossWrap`, `#botLeft`, `#botRight` | Persistent HUD panels (Score/Combo/Wave/Best, Controls toggles, Boss HP bar, Hull/Shield/Boost/Nova/Missile gauges, Speedometer). |
| `7` | `#tutBar`, `#waveBanner`, `#mobBtns` | Interactive first-time tutorial HUD card, Wave/Boss banners, Mobile action buttons (`<= 860px`). |
| `10` | `#overlay` | Modal container holding `#cardStart`, `#cardPause` (with tabbed Pilot Manual), `#cardUpgrade`, and `#cardOver`. |

### 3.2 Logical Sections Inside `<script>`
1. **Utilities & Guard:** `$()`, `clamp()`, `lerp()`, `rnd()`, `V3()`, and missing-Three.js fallback error message.
2. **`MANUAL_TABS`:** Static HTML content for the 3-tab Pilot Manual (`controls`, `combat`, `threats`), rendered into `#manualContent`.
3. **`AUDIO` Module:** Self-contained synthwave step sequencer (`tickMusic`) + 14 procedural SFX methods (`shoot`, `missile`, `lock`, `graze`, `hit`, `deflect`, `explode`, `ring`, `power`, `roll`, `emp`, `damage`, `tutOk`, `wave`).
4. **Scene & Environment Setup:** `renderer`, `scene`, `camera`, 3 lights, `TEX` (`glow`, `spark`, slatted `sun`), `sunGroup`, starfield `Points`, scrolling `gridTex` floor, boundary rails, `THEMES` palette array, 56 flanking wireframe `mountains`, 8 overhead `arches`, and 60 speed `streaks`.
5. **`buildShip()`:** Constructs the multi-part `"AERO-V89"` player starfighter + 3D forward aiming reticle (`retGroup` at `z = -26`).
6. **Object Pools & `SHARED` Assets:** Shared geometries/materials plus pre-allocated pools (`shockPool`, `partPool`, `laserPool`, `misPool`, `eBoltPool`, `lockEls`).
7. **State & Progression Config:** `DIFFS` (3 difficulty presets), `UPGRADE_POOL` (5 ship mods), and the central state object `G`.
8. **UI & Input Listeners:** Keyboard (`keydown`/`keyup`, including double-tap `A`/`D` barrel roll), Pointer/Touch (`pointerdown`/`pointermove`/`pointerup`), and button handlers.
9. **Tutorial Director:** `TUT_INFO`, `setTutStep()`, `finishTutorial()`.
10. **Entity Lifecycle & Combat:** `clearEntities()`, `startGame()`, `gameOver()`, `openUpgradePicker()`, spawners (`spawnObstacle`, `spawnRing`, `spawnRingChain`, `spawnPickup`, `makeEnemyEntity`, `spawnEnemyWave`, `spawnBoss`), and combat handlers (`fireLasers`, `fireMissiles`, `tryRoll`, `tryEmp`, `destroyEnemy`, `damageBoss`, `triggerGraze`, `takeDamage`).
11. **Main Simulation Loop:** `stepFrame(rawDt, now)`, `syncHud()`, `resize()`, and `loop(now)`.

---

## 4. 3D Coordinate System & Spatial Contract

All gameplay math assumes a right-handed Three.js space where the player flies forward along **`-Z`** (implemented by keeping the player at `z = 0` and moving the world along **`+Z`**):

- **Player Plane:** Fixed `z = 0`. Clamped to `x ∈ [-9.2, 9.2]`, `y ∈ [1.4, 8.2]`.
- **Camera:** Positioned at `z = 9.9`, softly tracking `(G.px * 0.52, G.py * 0.5 + 2.35)` and looking at `(G.px * 0.42 + sunSway, G.py * 0.6, -26)`. Dynamic FOV warps from `58°` up to `76°` during Barrel Roll or Hyper-Fever.
- **Corridor Bounds:** Visual rails at `x = ±11.4`; mountain ridges spawn at `|x| ∈ [19.5, 26.0]`.
- **Spawn / Despawn Z-Planes:**
  - World environment wraps seamlessly between `z = +24` and `z = -205`.
  - Hazards, rings, and enemies spawn at `z = -175` to `-196` and despawn at `z > 18`.
  - Player lasers & homing missiles travel along `-Z` and deactivate at `z < -195` or when `life <= 0`.
  - **Boss Hover Plane:** Lerps smoothly to `z = -46` and weaves in a Lissajous path (`x = sin(phase)*6.2`, `y = 4.8 + cos(phase*1.35)*1.8`).

---

## 5. State Machine & Game Modes

The game is governed by `G.mode` (and `G.prevMode` for modal return navigation):

| `G.mode` | Simulation | Music | Overlay Visible | Entry / Exit |
| :--- | :--- | :--- | :--- | :--- |
| `'menu'` | Idle world scroll + orbiting showcase camera | Off | `#cardStart` | Initial boot state. Exits to `'play'` via `startGame()` or `'pause'` via "How To Play". |
| `'play'` | Active flight, combat, spawners, or tutorial (`G.tutStep > 0`) | On | Hidden | Transitions to `'pause'`, `'upgrade'`, or `'over'`. |
| `'pause'` | Completely frozen (`dt = 0`) | Off | `#cardPause` | Entered via `P`/`Esc`/`Pause / Guide` or `How To Play`. `closePauseOrGuide()` restores `G.prevMode`. |
| `'upgrade'` | Completely frozen (`dt = 0`) | Off | `#cardUpgrade` | Entered automatically upon defeating a Boss. Selecting 1 of 3 cards applies the mod and resumes `'play'`. |
| `'over'` | Ship hidden, explosions/particles continue animating | Off | `#cardOver` | Entered when `G.hull <= 0`. Saves Hi-Score to `localStorage`. |

### Interactive First-Time Tutorial (`G.tutStep`)
When `startGame(true)` is invoked (automatically on first run if `localStorage['solaris_run89_tut'] !== '1'`, or manually via "Flight Training" / "Replay Training"):
- **Step 1 (Steer):** Accumulates `G.tutProg += hypot(vx, vy) * dt`; completes at `> 18`.
- **Step 2 (Boost Ring):** Spawns a centered ring at `(0, 4.2, -115)`; respawns if missed.
- **Step 3 (Weapons & Lock-On):** Spawns 2 passive (`fireT = 99`) Scout drones; completes when destroyed.
- **Step 4 (Barrel Roll Deflect):** Keeps `G.boost = 100` and fires a slow centered bolt (`vz = 34`); respawns if missed; completes once `G.deflects > 0`, then calls `finishTutorial(false)` which sets `solaris_run89_tut = '1'` and starts Wave 1.
- **Safety Net:** During `G.tutStep > 0`, `takeDamage()` never reduces Hull/Shield so new players cannot die during training. Pressing `Tab` or clicking `#skipTutBtn` calls `finishTutorial(true)`.

---

## 6. Core Gameplay Mechanics & Tuning Reference

### 6.1 Progression, Waves & Scoring
- **Waves:** Advance every `26s` (`G.waveTimer >= 26`) when no boss is active.
- **Boss Cadence:** Every 3rd wave (`G.wave % 3 === 0`) triggers `spawnBoss()` (`tier = Math.floor(G.wave / 3)`, `maxHp = 46 + tier * 20`).
- **Combo & Hyper-Fever:** Kills, rings, deflects, and grazes increment `G.combo` (capped at `x8`, resets after `6.4s` idle or upon taking hull damage). At **`G.combo >= 6`**:
  - `feverMul = 1.5` applied inside `addScore()`.
  - World speed increases by `+8`, camera FOV widens, engine trails shift magenta, and `AUDIO.tickMusic` accelerates from `108 BPM` to `118 BPM` with a `+1 octave` arpeggio.
- **Difficulty Presets (`DIFFS`):**
  - `0` (`CRUISE`): `baseSpd: 48`, `spawnRate: 1.18`, `dmg: 0.72`, `scoreMul: 0.85`
  - `1` (`ARCADE`): `baseSpd: 58`, `spawnRate: 1.00`, `dmg: 1.00`, `scoreMul: 1.00`
  - `2` (`HYPERDRIVE`): `baseSpd: 70`, `spawnRate: 0.82`, `dmg: 1.30`, `scoreMul: 1.35`

### 6.2 Combat & Collision Rules
- **Barrel Roll (`tryRoll`):** Lasts `0.55s`, costs `G.upg.rollCost` (default `32`) Boost. Grants full obstacle invulnerability, speeds world by `1.72x`, performs a `2π` roll, and reflects any enemy bolt within `G.upg.deflectRad` (default `2.45`) back as a high-damage (`3.5 * dmgMul`) green laser (`DEFLECT! +400`) with `0.09s` micro slow-mo (`G.slowT`).
- **Close-Call Graze (`triggerGraze`):** Triggered once per obstacle/bolt (`grazed` flag) during non-tutorial flight when outside hit radius but inside graze envelope. Awards `+150 * combo`, `+18` Boost, and `+8 * empMul` Nova charge.
- **Multi-Lock Missiles (`fireMissiles`):** Every frame, up to `G.upg.maxLocks` enemies within `z ∈ [-165, -10]` and XY radius `< 7.2` from the ship are pushed into `G.lockedTargets` and projected onto DOM `.lockBox` brackets. Pressing `Q`/`F`/Middle-Click fires curved homing missiles (`dmg = 3.6 * dmgMul`) that track live targets (`target.alive`).
- **Rotating Cross Collision (`type === 'cross'`):** Player `(px, py)` is transformed into the cross's local rotated frame via `cos(-ang)` / `sin(-ang)` to perform exact dual-axis bar AABB checks for both hit and graze zones.
- **Chain-Reaction Mines (`type === 'mine'`):** Destroying a mine spawns a shockwave and recursively calls `destroyEnemy(..., true)` on all enemies within `12.5` units.

### 6.3 Persistent `localStorage` Keys
| Key | Type | Purpose |
| :--- | :--- | :--- |
| `solaris_run89_hi` | Stringified Integer | All-time Hi-Score persisted on Game Over. |
| `solaris_run89_tut` | `'0'` or `'1'` | Tracks whether the pilot completed/skipped first-time Flight Training. |

---

## 7. Performance & Memory Management Contract

Future developers **must** preserve these invariants to keep WebGL frame times flat:

1. **Fixed Object Pools (Never `new` in `stepFrame`):**
   - `partPool` (`PART_MAX = 220`)
   - `laserPool` (`LASER_MAX = 52`)
   - `misPool` (`MIS_MAX = 14`)
   - `eBoltPool` (`EBOLT_MAX = 64`)
   - `shockPool` (`SHOCK_MAX = 5`)
   - `lockEls` (`6` DOM nodes, repositioned via CSS `transform`)
2. **Shared Assets (`SHARED`) vs. Owned Materials:**
   - All geometries and static materials live in `SHARED` and are **never** disposed during gameplay.
   - Only dynamic enemies (`en.ownMat`) and bosses (`boss.ownMat`) instantiate a cloned `MeshStandardMaterial` (to support independent white hit-flashes). Every code path that removes an enemy or boss from `scene` **must** call `ownMat.dispose()`.
3. **HUD Throttling:** `syncHud()` runs on a `60ms` accumulator (`hudAcc >= 0.06`, ~16 Hz) rather than every animation frame, preventing unnecessary DOM layout thrashing.
4. **Scratch Vectors:** `tmpDir` and `projV` are allocated once outside the loop and mutated in place.

---

## 8. Known Issues, Edge Cases & Technical Debt

The following known quirks exist in the current build and are prime targets for the next maintenance pass:

1. **Fever Badge Label vs. Math Discrepancy (Easy Bug Fix):**
   - `#feverBadge` displays `FEVER x2 SCORE`, while `addScore()` currently uses `const feverMul = G.combo >= 6 ? 1.5 : 1.0;`. Align either the multiplier to `2.0` or the badge label to `FEVER 1.5X`.
2. **Unpruned `G.squadAlive` Keys (Minor Memory Hygiene):**
   - In `destroyEnemy()`, `delete G.squadAlive[sid]` is called on squad wipe. However, when a squad member collides with the player or flies past `z > 18`, `G.squadAlive[en.squadId]--` decrements without deleting the key when it hits `0`. Add `if (G.squadAlive[en.squadId] <= 0) delete G.squadAlive[en.squadId];` in both despawn paths.
3. **Immediate Upgrade Modal Interrupts Boss Explosion:**
   - `damageBoss()` calls `openUpgradePicker()` on the exact frame `bs.hp <= 0`, freezing `dt = 0` immediately so the `0.22s` slow-mo and particle burst are paused behind `#overlay`. **Fix:** Introduce a `bossClearTimer` (~`1.1s`) before invoking `openUpgradePicker()`.
4. **Desktop Click-to-Fire Also Pulls Steering:**
   - In `stepFrame()`, `else if (G.mouseSteer || G.ptrDown)` steers the ship toward the cursor whenever the left mouse button is held down (designed for mobile touch-drag). On desktop, players who hold Left-Click to shoot without holding WASD will drift toward their cursor. **Fix:** Track `e.pointerType === 'touch'` separately from mouse clicks so touch always drags while desktop only steers with mouse if `G.mouseSteer` is `ON`.
5. **Unused `G.dist` Stat on Game Over:**
   - `G.dist` is actively accumulated during flight but is currently not rendered on `#cardOver` (replaced by `#goGrazes` in the 6-card grid). Either display it in a 2x4 grid or remove unused tracking.
6. **External CDN Dependency:**
   - If opened completely offline without a cached copy of `three.min.js`, the game displays a fallback error. If offline distribution is required, inline or locally bundle Three.js r128.
7. **Audio Autoplay & Tab-Focus Behavior:**
   - Web Audio requires a user gesture to unlock; if a user returns from a backgrounded tab, `dt` is safely clamped to `0.05s`, but `AUDIO.tickMusic` does not currently auto-pause on `document.visibilitychange`. Adding a `visibilitychange` listener that automatically calls `openPauseOrGuide(false)` when hiding the tab is strongly recommended.

---

## 9. Testing & Verification Protocol

Before shipping any changes to `solaris_run_89_overdrive.html`, run the following checks:

### Automated Checks (Node.js)
1. **Syntax Check:** Extract the inline `<script>` block and run `node --check <extracted.js>` to confirm zero syntax errors.
2. **DOM ID Integrity Check:** Verify every ID referenced via `$('id')` in JS exists as `id="..."` in the HTML markup.
3. **Headless Gameplay & Leak Simulation:** Run a mocked-Three.js Node harness (temporarily exposing `stepFrame`, `G`, `startGame`, `scene` during testing, then removing the export before release) asserting:
   - Full 4-step tutorial progression (`tutStep` `1 -> 2 -> 3 -> 4 -> 0`) and `solaris_run89_tut` persistence.
   - Barrel Roll deflection increments `G.deflects` and spawns a counter-laser.
   - Chain-reaction mine detonation clears adjacent mines.
   - Boss defeat transitions `G.mode` to `'upgrade'`, renders 3 upgrade cards, and clicking one resumes `'play'` on `wave + 1`.
   - Pause (`P`) freezes `G.score`, and Pause/Manual tab switching updates `#manualContent`.
   - **Scene Leak Check:** `scene.children.length` immediately after `startGame(false)` equals the exact baseline child count captured at boot (`488` objects in the current build).

### Manual Browser QA Checklist
- [ ] Desktop Chrome, Firefox, and Safari at 1080p and HiDPI/Retina (`devicePixelRatio >= 2`).
- [ ] Incognito / Private Window (confirms `localStorage` fallback doesn't throw and first-time tutorial triggers).
- [ ] Mobile / DevTools Touch Emulation (`<= 860px`): verify `#mobBtns` (`MISSILE`, `ROLL`, `NOVA`) appear and touch-drag steers smoothly.
- [ ] Audio toggles (`Synth`, `SFX`) and `CRT` toggle work without console warnings.

---

## 10. Recommended Upgrade Roadmap

Ordered by player impact vs. implementation effort:

1. **Quick Wins (High Impact, Low Effort):**
   - Fix the `#feverBadge` text/math alignment (`1.5x` vs `2x`), add `visibilitychange` auto-pause, separate `touch` vs desktop `mouse` pointer steering, and add a `1.1s` victory delay before opening the Upgrade Picker after a Boss kill.
2. **Multi-Phase Bosses & Distinct Archetypes:**
   - Add destructible orbiting shield generators on the Boss (must destroy nodes before damaging the core) and a telegraphed sweeping beam attack on MK-II+.
3. **True Curved Synthwave Horizon:**
   - Inject a lightweight vertex-shader bend (`onBeforeCompile` on floor/obstacle materials) that curves `-Z` geometry gently left/right/down with distance (`offset = z * z * curveStrength`), creating an authentic *OutRun* / *F-Zero* twisting highway feel.
4. **Gamepad API Support:**
   - Poll `navigator.getGamepads()` in `stepFrame()` to map Left Stick -> `ix/iy`, A/RT -> Fire, B/LB/RB -> Barrel Roll, X -> Homing Missiles, Y -> Nova Pulse.
5. **Endless Meta-Progression & Ship Skins:**
   - Store cumulative score / Boss cores in `localStorage` to unlock alternate interceptor colorways (e.g., *OutRun Gold*, *Midnight Stealth*, *Vaporwave Chrome*) directly from `#cardStart`.
