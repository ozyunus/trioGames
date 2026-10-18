# TRIO — iOS-first Flutter Game Design & Implementation Spec
Version: 0.2 draft | 17 October 2026 | Supersedes 0.1

**How to read this document**
- MUST / SHOULD / MAY are used in the RFC sense.
- `[A#]` marks a design assumption that has not been playtested. It is listed in §17 with the metric that will confirm or kill it. Implement it as a constant in `TuningConfig`, never as a magic number.
- `[BLOCKER]` marks something outside the code (accounts, keys, legal) that must exist before release.
- Rules are written so they can be turned directly into unit tests (§16).

---

## 1. Product vision
TRIO is one offline-first mobile block puzzle game with three modes: SHIFT (gravity), ECHO (ghost replay), MICRO (compact quick play). One shared engine, three rule sets. iOS MVP first; keep Android portability. Avoid copying Tetris branding, audiovisual identity or exact presentation.

## 2. Scope and non-negotiables
- Flutter + Dart, iOS only for MVP, portrait only.
- No accounts, login, backend, cloud saves, payments, subscriptions or in-app purchases. The Setgreet purchase/paywall features MUST NOT be used.
- Scores, settings, progress and run snapshots are stored **only on device**.
- **Setgreet** = first-launch onboarding only (SDK integration, no custom onboarding screens). See §15.
- **Firebase Analytics** = anonymous usage events only. See §13.
- **Google AdMob** = the only monetization. End-of-game interstitial only. See §14.
- No custom artwork or asset pipeline: Flutter widgets, `CustomPainter`, system font. No image/audio assets.
- Dependencies are an allow-list (see §11.3). Adding one needs a written reason.
- "No user ID" statement for the privacy policy MUST be worded as: *TRIO does not create accounts or user identifiers. Third-party SDKs (Firebase, Setgreet, AdMob) may generate their own anonymous identifiers.*

## 3. Decision log (owner decisions, v0.2)
| Topic | Decision |
|---|---|
| Engine | One shared pure-Dart engine for all modes |
| Pieces | Original 3–5 cell pieces (catalog in §6.3) |
| Randomness | Seedable PRNG, own implementation (§6.4) |
| Controls | Tap and swipe/drag |
| State management | `ChangeNotifier` (no Riverpod/Bloc) |
| DI | Constructor injection |
| Storage | Versioned JSON files |
| High scores | Separate per mode |
| SHIFT board | 10×10 square; 3 gravity charges at start, max 3 |
| ECHO | Deterministic ghost: after every 5th placement, ghost of the piece placed 3 placements earlier |
| MICRO board | 6×8; drag and drop; bottom usable row closes every 10 placements |
| Ads | In MVP. AdMob. End-of-game interstitial only |
| Onboarding | Setgreet only. "How to Play" is a separate, tiny in-app feature |
| Home visuals | Keep the v0.1 look; fix inconsistencies only |

---

## 4. Design system
### 4.1 Direction
Minimal premium flat. Calm navy canvas, softly outlined grids, crisp geometric blocks, restrained colour. No glassmorphism, textures, 3D, animated backgrounds, image assets.

### 4.2 Colour tokens (unchanged)
| Token | Hex | Use |
|---|---|---|
| canvas | #101827 | Screen background |
| panel | #151F31 | Board/hero grid background (new, was inline in SVG) |
| surface | #192337 | Cards |
| surfaceRaised | #22304A | Pressed/elevated states |
| gridLine | #344158 | Grid strokes (use 50% opacity inside grids) |
| textPrimary | #F5F7FC | Primary text |
| textSecondary | #9CAAC0 | Labels |
| shift | #34D6ED | SHIFT accent |
| echo | #B39AFA | ECHO accent |
| micro | #F7B955 | MICRO accent |
| mint | #5BD5A4 | Positive/extra piece colour |
| danger | #F47C85 | Error/game over/invalid |

Piece fill colour is chosen by piece id from `[shift, echo, micro, mint]` (see §6.3) in **all** modes. The home hero illustrates this palette (it is decorative, so one V3 uses cyan for balance). Mode identity comes from chrome (title, header accent, card border), not from piece colour.

### 4.3 Typography (iOS system font only)
Use the default Cupertino text theme (SF Pro). Never hard-code `fontFamily`.
| Role | Size/weight |
|---|---|
| Wordmark "TRIO" | 40/700 (logo only, not a UI token) |
| Display | 32/700 |
| Page title | 24/700 |
| Section | 18/600 |
| Mode-card title | 16/700 |
| Body | 15/400 |
| Caption | 12/500, letter-spacing up to 2 for all-caps labels |
**Minimum shipped text size is 12.** Menus MUST respect Dynamic Type (test at 1.5× — §16).

### 4.4 Shape, spacing, motion
- 8 px base grid. Allowed gaps: 8, 16, 24, 32. Horizontal screen inset: 20.
- Cards radius 16; controls radius 12; tiles radius 4 (3 allowed inside icons). 1 px borders at subdued contrast; mode cards use a 1.5 px accent outline (the only exception).
- Touch targets ≥ 44×44 pt.
- No shadows (optional subtle overlay scrim behind pause/results).
- Durations: tap feedback 90–140 ms; tile settle 120–180 ms; line clear 180–260 ms; gravity shift 220–320 ms; echo drop 120–180 ms.
- Reduced motion (`MediaQuery.disableAnimations`): replace movement with ≤ 80 ms cross-fades, static How-to-Play illustrations.
- 60 FPS target. Board drawn by one `CustomPainter` inside a `RepaintBoundary`; do not rebuild the widget tree per frame.
- Haptics toggle (default **on**, light impact only), sound toggle (default **off**). Because there are no audio assets, "sound" in MVP = `SystemSound.play(SystemSoundType.click)` on placement. `[A1]` Real audio is deferred.

### 4.5 Home reference
`TRIO_HOME_REFERENCE.svg` (v0.2) is the visual reference for Home. Changes from v0.1, **look unchanged**:
- Text and grid aligned to the 20 px inset; card gap 16; card radius 16; card height 128.
- Hero grid is now 10×10 (matches SHIFT) with cell 28, tile 24, radius 4.
- Hero pieces replaced with original catalog shapes (I3, P5, U5, V3 ×2, X5) instead of T/S/O tetrominoes.
- Unicode glyph icons replaced by geometric icons to be built with `CustomPainter`: SHIFT = double arrow, ECHO = solid square + dashed offset square, MICRO = 2×2 grid with one hollow cell.
- Labels ≥ 12. Developer footer text removed. Settings button added (44×44, top right).
- Fonts in the SVG are the iOS system stack; in Flutter use the Cupertino default.

---

## 5. Navigation and screens
### 5.1 Launch sequence
```
Splash (canvas colour, no logo animation, ≤ 300 ms)
 → ConsentService.resolve()          // UMP if required; analytics consent (§13.3)
 → first launch only: OnboardingGateway.runFirstLaunchFlow()   // Setgreet, §15
 → Home
```
Every step MUST be skippable on failure/timeout so the app is always reachable offline.

### 5.2 Screens
1. **Home** — wordmark, tagline `ONE GRID. THREE RULES.`, hero grid (`CustomPainter`), label `SELECT YOUR MODE`, three equal mode cards, settings button. Entire hero is decorative (`ExcludeSemantics`).
2. **Mode intro** — mode name, one-sentence mechanic, best local score, **Play**, **How to Play**.
3. **How to Play** (modal sheet) — 3 short cards per mode, each one sentence + a small looping `CustomPainter` animation (static if reduced motion). Not an onboarding system, no remote content. Opens from Mode intro and from the Pause overlay.
4. **Gameplay** — header (back/pause, mode title, score, level for SHIFT/ECHO), board, next-piece/tray, controls, pause overlay. Back = pause + "Leave run?" confirmation (run is saved).
5. **Results** — score, personal best (flag "New best"), Replay, Home. Interstitial is shown on the player's *next navigation action* from here (§14), never over the score itself.
6. **Settings** — sound, haptics, analytics consent toggle, privacy policy link, credits, Reset local data (confirmation; also calls `resetAnalyticsData()` and Setgreet `resetUser()` when available `[verify]`).

### 5.3 Copy rules
No developer-facing strings, no version/marketing taglines in shipped UI. All strings in one `strings.dart` (English for MVP; structure allows later localisation).

---

## 6. Shared engine (pure Dart, no Flutter imports)

### 6.1 Conventions
- Board coordinates are `(r, c)`: `r = 0` is the top row, `c = 0` the left column.
- `Direction { down, left, up, right }` with deltas `down (+1,0)`, `left (0,−1)`, `up (−1,0)`, `right (0,+1)`.
- `rotateCW(down)=left`, `rotateCW(left)=up`, `rotateCW(up)=right`, `rotateCW(right)=down` (an arrow pointing down turned 90° clockwise on screen points left). `rotateCCW` is the inverse.
- The engine is **synchronous and authoritative**: an input produces the final resolved state immediately plus an ordered `List<ResolveEvent>` (locks, clears, collapses, ghost events). Animations are presentation only, they replay events. Snapshots are always post-resolution.
- Time enters only through `advance(Duration)`; tests use virtual time.

### 6.2 Core types
`Cell(r,c)`, `Board(rows, cols)` with `List<int>` (0 = empty, 1..4 = colour id), `ShapeDef`, `PieceInstance(shapeId, orientation, cells)`, `GameRules` (Strategy), `GameState`, `ResolveEvent`, `ClearPolicy`, `CollapsePolicy`.

Three collapse policies, selected by mode:
| Policy | Mode | After full lines are removed |
|---|---|---|
| `CellGravity(direction)` | SHIFT | Every settled cell slides along gravity until blocked (per lane, order preserved) |
| `LineShift(down)` | ECHO | Rows above a removed row move down by the number of removed rows below them; cells keep their relative shape |
| `None` | MICRO | Nothing moves |

### 6.3 Piece catalog
Cells are `(r,c)` in orientation 0. Rotation is clockwise: `(r,c) → (c, h−1−r)` where `h` is the current height, then normalise to min = 0. Mirror images are **not** included. All pieces have 3–5 cells.
| id | Name | Cells | Weight | Colour |
|---|---|---|---|---|
| 0 | I3 | (0,0)(0,1)(0,2) | 2 | shift |
| 1 | V3 | (0,0)(1,0)(1,1) | 2 | mint |
| 2 | O4 | (0,0)(0,1)(1,0)(1,1) | 2 | micro |
| 3 | P5 | (0,0)(0,1)(1,0)(1,1)(2,0) | 1 | echo |
| 4 | U5 | (0,0)(0,2)(1,0)(1,1)(1,2) | 1 | micro |
| 5 | V5 | (0,0)(1,0)(2,0)(2,1)(2,2) | 1 | shift |
| 6 | T5 | (0,0)(0,1)(0,2)(1,1)(2,1) | 1 | echo |
| 7 | X5 | (0,1)(1,0)(1,1)(1,2)(2,1) | 1 | shift |
| 8 | W5 | (0,0)(1,0)(1,1)(2,1)(2,2) | 1 | mint |
| 9 | Z5 | (0,0)(0,1)(1,1)(2,1)(2,2) | 1 | mint |
`[A2]` Weights and the 10-piece set are a starting point. The catalog is pure data in one file so it can change without touching rules. Pieces are distinguishable by shape, not only colour.

Distinct orientations per shape are precomputed (I3: 2, Z5: 2, O4: 1, X5: 1, all others 4). MICRO uses these as separate non-rotatable pieces.

### 6.4 Seeded PRNG
- MUST NOT use `dart:math` `Random` (sequence is not guaranteed stable across SDK versions).
- Use **mulberry32** with explicit 32-bit masking (`& 0xFFFFFFFF`):
```
state = (state + 0x6D2B79F5) & M
t = state
t = ((t ^ (t >> 15)) * (t | 1)) & M
t = (t ^ ((t + (((t ^ (t >> 7)) * (t | 61)) & M)) & M)) & M
return (t ^ (t >> 14)) & M
```
- Golden vectors (first five `nextUint32`): seed `1` → `2693262067, 11749833, 2265367787, 4213581821, 4159151403`; seed `12345` → `4207900869, 1317490944, 2079646450, 3513001552, 2187978186`.
- `nextInt(n) = nextUint32() % n` (modulo bias accepted and fixed for determinism).
- Piece pick: weighted by §6.3 using `nextInt(totalWeight)`. If the pick equals the previous **two** picks, re-pick (max 4 attempts, then accept).
- One RNG instance per run for pieces. Its single `uint32` state is stored in the snapshot.

### 6.5 Lifecycle state machine
`idle → playing ⇄ paused → gameOver`. `playing` has sub-state `resolving` (input locked while animations from the last input play). Transitions:
- App goes to background / interruption → `paused` + save. Returning does **not** auto-resume; the pause overlay is shown.
- Back button → `paused` + confirmation.
- `gameOver` clears the run snapshot, updates bests/plays and emits analytics.

### 6.6 Shared scoring (all values in `TuningConfig`, `[A3]`)
- Placement: `+1` per cell of the locked piece.
- Lines cleared simultaneously, `n ≥ 1`: `lineScore(n) = 100 · n · (n+1) / 2` → 100, 300, 600, 1000, 1500…
- Chain step `k` (1-based) multiplies that step's `lineScore` by `k` (SHIFT, and ECHO echo-drops only).
- SHIFT/ECHO drop: swipe-drag toward gravity `+1`/cell, hard drop `+2`/cell travelled.
- Level (SHIFT/ECHO): `level = min(10, 1 + totalLinesCleared ~/ 5)`; fall interval in ms for levels 1..10: `[800, 720, 650, 580, 510, 450, 390, 340, 300, 260]`.
- Lock delay: 300 ms after the piece can no longer advance; a successful move/rotation restarts it, max 4 restarts; hard drop locks immediately.

### 6.7 Rotation of the active piece (SHIFT, ECHO)
Tap rotates clockwise. Try offsets in the **gravity frame** `(lateral, along)` in this order and take the first that fits: `(0,0) (−1,0) (+1,0) (0,−1) (−1,−1) (+1,−1) (−2,0) (+2,0)`. If none fits, the rotation is ignored (light error haptic). `along = −1` means one cell against gravity.

---

## 7. SHIFT — Gravity
**Board 10×10.** Gravity starts `down`. Charges: start **3**, max **3**. Charge progress: every 3 lines cleared (counted across all chains) grants one charge. While at max charges, progress **freezes** (lines are not banked), so nothing is wasted silently.

### 7.1 Spawn and movement
- A new piece spawns on the **entrance edge**, the edge opposite the current gravity (gravity down → row 0).
- Its orientation is rotated into the gravity frame (a piece looks the same relative to gravity at any gravity). Lateral position = `floor((10 − lateralExtent) / 2)`.
- It advances one cell along gravity per fall interval.
- Lateral moves follow the screen axis perpendicular to gravity (gravity down/up → screen left/right; left/right → screen up/down; negative = left/up). On-screen control icons rotate with gravity.
- **Game over** when the spawn footprint overlaps any settled cell (`reason = spawn_blocked`).

### 7.2 Lock and resolution
1. Active piece locks → its cells become settled cells. **No compaction yet** (a locked piece keeps its shape until a clear or a gravity change). `[A4]`
2. Resolution loop:
```
chain = 0
loop:
  full = every line perpendicular to gravity whose 10 cells are all settled
         (gravity down/up → rows; left/right → columns)
  if none: break
  chain += 1
  remove all cells of all full lines (simultaneous)
  score lineScore(count) * chain ; totalLines += count ; update charge progress
  CellGravity compaction in current gravity direction
```
3. Compaction is per lane (column for vertical gravity, row for horizontal): remaining cells keep order and slide to the gravity side. Lanes are independent, so the result is order-independent and deterministic.
4. Next piece spawns. If spawn is blocked → game over.

### 7.3 Gravity rotation (clockwise or counter-clockwise, costs 1 charge)
1. **Validate (dry run on a copy):** apply steps 2–4 below, then check whether the **next piece's spawn footprint** (oriented for the *new* gravity) is free of settled cells. If not → **reject**: no charge spent, no state change, error haptic + inline message "No room for the next piece".
2. Spend 1 charge. Set new gravity.
3. **Active piece does not teleport.** It stays at its current cells and from the next tick moves along the new gravity.
4. Run `CellGravity(newGravity)` on all settled cells, treating the active piece's cells as **immovable obstacles**; then run the resolution loop of §7.2 (rotation can complete lines and chain).
5. Input is locked until the event list has played (gravity shift 220–320 ms + clears).
Rotation is not available when: charges = 0, state is `resolving`/`paused`, or no active piece exists (only during resolution).

The dry run validates only the footprint right after the rotation. The board can still change before the active piece locks; spawn blocking at that time is a normal game over.

### 7.4 Required UI
Persistent gravity arrow (icon, not only colour) beside the board; entrance edge marked with tick marks; charge pips (3) with a 3-step progress meter; next-piece preview drawn in the future gravity orientation of the *current* gravity.

### 7.5 Edge cases (each has a test in §16)
- Rotation on an empty board; rotation with active piece against a wall; rotation causing a multi-line clear; rotation causing a chain; two rotation requests during one resolution (second is ignored); rotation with 0 charges; rejection leaves state byte-identical; rotation when the active piece sits exactly on the spawn region of the next piece (allowed, dry run ignores the active piece); compaction jammed against the active piece, then the piece moves away leaving a gap (expected, no extra compaction until next clear or rotation); charges at max with lines cleared (progress frozen); three lines in one clear grants exactly one charge; 4+ lines grant one charge and keep the remainder only if below max.

---

## 8. ECHO — Time
**Board 8×12** (8 columns × 12 rows). Gravity fixed `down`. Standard piece play (§6.7, §6.6). Collapse policy `LineShift`.

### 8.1 Definitions
- `p` = count of locked placements in the run (1-based, increments at lock).
- `history[p]` = the piece's **absolute cells at the moment of locking** (before any clear affects them).
- A **ghost** = `{cells, remaining}`. At most one ghost exists at a time (`5 > 2` guarantees no overlap).

### 8.2 Per-lock pipeline (order matters)
1. Lock piece; `p += 1`; record `history[p]`.
2. If a ghost is pending and any of its cells is now settled → **cancel** (`cancelled_overlap`).
3. Remove full rows; `LineShift`; score `lineScore(n)`.
4. If a ghost is still pending: `remaining −= 1`.
   - If any ghost cell is occupied (rows may have shifted into it) → **cancel**.
   - Else if `remaining == 0` → **solidify**: write the ghost cells as settled, then drop that set **rigidly** down until it rests; remove full rows again (this is chain step 2, multiplier ×2); `LineShift`.
5. If `p % 5 == 0` and no ghost is pending: take `history[p − 3]`. If every cell of it is free on the stable board → create ghost with `remaining = 2`; otherwise **no ghost** (`cancelled_at_birth`).
6. Spawn next piece. Blocked → game over.

Timeline check: ghost born after lock 5 (`remaining 2`) → lock 6 → `remaining 1` → lock 7 → `remaining 0` → solidifies at lock 7. Next ghost born after lock 10.

### 8.3 Rules
- The ghost does **not** collide with the falling piece; the falling piece may pass through it. Locking a piece onto any ghost cell cancels the ghost (this is intended counterplay).
- Ghost cells stay at their **absolute coordinates**; they do not move with line shifts.
- Solidified cells become ordinary settled cells (no special tag after landing) and participate in clears.
- Existing blocks are never overwritten.
- Cancelled ghost: no penalty, no bonus.

### 8.4 Required UI
Translucent fill **plus** diagonal hatch **plus** dashed outline **plus** a numeric countdown badge (2, then 1) on the ghost; not colour alone. Brief "fizzle" animation on cancel. First-ever ghost shows a one-line hint ("Echoes return as solid blocks").

### 8.5 Edge cases
Ghost born while falling piece overlaps its cells (allowed); ghost whose cells become occupied by a *shifted row* (cancel in step 4); ghost solidifies in mid-air and drops several rows; rigid drop completes a row (chain ×2); solidify blocks spawn → game over; ghost at p=5 uses `history[2]`; run saved and restored with a pending ghost; app killed between lock and ghost creation (snapshot is post-resolution so ghost state is consistent); p rolls past 5 multiples with a ghost pending cannot happen.

---

## 9. MICRO — Quick
**Board 6×8** (6 columns × 8 rows). No falling, no timer.

### 9.1 State
`board`, `closedRows` (count of unusable rows **from the bottom**), `placements`, `tray[3]` (piece or null), `round`, `combo`, `score`. Usable area = rows `0 … 7 − closedRows`.

### 9.2 Tray
- Three pieces per round, any order. Pieces are fixed-orientation (pre-rotated variants from §6.3). No rotation, no refresh button in MVP.
- When all three are placed, a new tray is dealt and `round += 1`, `+25` bonus.
- **Fairness re-roll** `[A5]`: a new tray is re-rolled (consuming the PRNG, max 8 times) until at least one of its pieces has a legal placement, if such a piece exists in the pool. If none can ever fit, the tray is accepted and the run ends.

### 9.3 Placement input
- Drag a tray piece; the piece is drawn **1.5 cells above the finger** so it is visible; a snapped preview outline shows the target. Valid = accent outline; invalid = danger outline **with cross-hatch** (not colour only). Release over an invalid position returns the piece to the tray (120–180 ms).
- Accessibility alternative: tap a tray piece (select), then tap the target cell, which becomes the piece's top-left anchor.
- A placement is legal only if every cell is inside the usable area and empty.

### 9.4 Per-placement pipeline
1. Place; `+1` per cell; `placements += 1`.
2. Detect full **rows** (6 usable cells filled) and full **columns** (all usable rows filled) `[A6]`. Remove the union simultaneously (`None` collapse). `n` = number of lines. Score `lineScore(n)`; if `n ≥ 1`: `combo += 1` and `+50 · (combo − 1)`; else `combo = 0`.
3. If `placements % 10 == 0` → **closure**: `closedRows += 1`; all blocks in the newly closed row are deleted; **no score**. Lines that become complete only because the usable area shrank (columns) are cleared immediately with **0 points and no combo change**.
4. If tray empty → deal new tray.
5. **Game over** if usable rows = 0, or no piece in the tray has a legal placement (`reason = no_move`). This is checked after closure, so a closure can end the run.

### 9.5 Warning and UI
- The next row to close (`usableRows − 1`) is **always** marked (hatch + label "N" = placements remaining, `10 − placements % 10`), from the first placement of each cycle; ≤ 3 remaining adds a pulse (static if reduced motion). Closed rows render as a solid dark strip.
- Placing blocks on the marked row is allowed (they will be deleted without score).

### 9.6 Targets
Typical run 45–90 s `[A7]`. Tuning knobs: `shrinkEvery` (10), piece weights, tray fairness, `+25` round bonus.

### 9.7 Edge cases
Closure deletes blocks in the row; closure completes a column → silent clear, 0 pts, combo unchanged; closure at `usableRows == 1` → game over; closure makes all three tray pieces unplaceable → game over immediately; placement that clears and is also the 10th (clear scored first, then closure); placing the last tray piece ends the run before the new tray is dealt (new tray's fairness re-roll can rescue it only if a legal piece exists; otherwise game over after deal); drag released outside board; drag of an already placed tray slot; backgrounding mid-drag (piece returns to tray, run saved); VoiceOver tap-select flow.

---

## 10. Input and accessibility
- Portrait only. One-hand reach: primary controls in the lower third.
- **SHIFT/ECHO gestures:** horizontal drag on the board = lateral move (one cell per cell-width travelled); tap = rotate piece; swipe toward gravity = hard drop; slow drag toward gravity = soft drop. Always-available buttons: move ◄ ►, rotate, drop, and (SHIFT) gravity ⟲ ⟳ with charge pips.
- **MICRO:** drag from tray; tap-select alternative (§9.3).
- Pause/resume clear; back confirmation; no loss on interruption.
- VoiceOver: labels on all navigation and buttons; board exposes a summarised state ("Score 1,200. Gravity down. 2 charges. Next piece: U shape"); no information by colour alone (ghost, invalid drop, closing row, gravity arrow, charges).
- Dynamic Type for menus; game HUD uses fixed sizes ≥ 12.

---

## 11. Flutter architecture
### 11.1 Folders
```
lib/
  app/        bootstrap, theme, routes, strings, app_shell
  core/       storage/, analytics/, ads/, onboarding/, consent/, tuning_config.dart
  game/
    domain/       board, cells, shapes, pieces, seeded_rng, collapse_policies, game_state, events
    application/  game_controller (ChangeNotifier), save_coordinator
    presentation/ board_painter, controls, animations, hud
  modes/{shift,echo,micro}/   *_rules.dart, *_screen.dart, how_to_play.dart
  home/  settings/  results/
test/  (mirrors lib/)
```
### 11.2 Patterns
Strategy (`GameRules` per mode), state machine (lifecycle), Repository (`SaveRepository`), Adapters (`AnalyticsService`, `AdService`, `OnboardingGateway`, `ConsentService`), constructor DI via a single `AppDependencies` object created in `bootstrap`. `GameController extends ChangeNotifier`; screens listen with `ListenableBuilder`. `game/domain` MUST NOT import Flutter.
### 11.3 Dependency allow-list
`firebase_core`, `firebase_analytics`, `setgreet`, `google_mobile_ads` (includes UMP), `path_provider`, `url_launcher` (privacy policy). Dev: `flutter_test`, `flutter_lints`. Haptics/sound via `flutter/services`. Navigation via `Navigator`. Pin exact versions in `pubspec.yaml`; iOS deployment target = max of all plugins (Setgreet requires **iOS 15.0+**, Flutter 3.27+, Dart 3.6+ as of its pub.dev page on 17 Oct 2026 `[verify before pinning]`).
### 11.4 Everything optional is disableable
Firebase, ads, Setgreet and consent each have a `Noop*` implementation chosen when keys/config are absent or consent is denied. The game MUST run with all four as no-ops.

---

## 12. Persistence (versioned JSON, on device)
Files in the app documents directory via `path_provider`:
| File | Content |
|---|---|
| `settings.json` | `schemaVersion`, `haptics`, `sound`, `analyticsConsent` (`unknown/granted/denied`) |
| `progress.json` | `schemaVersion`, `best{shift,echo,micro}`, `plays{…}`, `countedGames`, `gamesSinceLastAd`, `lastAdAtEpochMs`, `onboardingResolved`, `onboardingAttempts` |
| `run_<mode>.json` | `schemaVersion`, `mode`, `seed`, `rngState`, board cells, active piece, next piece, mode state (SHIFT: gravity, charges, progress, totalLines; ECHO: `p`, last 3 `history`, ghost; MICRO: tray, closedRows, placements, combo, round), `score`, `elapsedMs` |
Rules:
- Writes are atomic: write `file.tmp`, flush, rename; keep one `.bak`.
- Corrupt/unknown-future file → rename to `.corrupt`, load defaults, never crash, never overwrite a future-version file.
- Migrations are pure functions `vN → vN+1` with tests.
- Save after each settled turn (after resolution) and on pause/background. At most one in-flight write; coalesce.
- Reset local data deletes all files and resets in-memory state.

---

## 13. Analytics (Firebase)
### 13.1 Events (names fixed by owner; parameters are the contract)
| Event | Parameters |
|---|---|
| `mode_selected` | `mode` |
| `game_start` | `mode`, `resumed` (0/1) |
| `game_end` | `mode`, `reason` (`spawn_blocked`, `no_move`, `area_exhausted`, `abandoned`), `duration_bucket`, `score_bucket`, `level` |
| `session_duration` | `bucket` (once per app session, at background) |
| `level_reached` | `mode`, `level` (SHIFT/ECHO) |
| `line_cleared` | `mode`, `count` — **one event per resolution**, never per line |
| `gravity_changed` | `direction` (`cw`/`ccw`), `rejected` (0/1) |
| `echo_triggered` | `stage` (`created`, `cancelled`, `solidified`) |
| `how_to_play_viewed` | `mode` |
| `onboarding_result` | `result` (`completed`, `dismissed`, `error`, `skipped`) — derived from Setgreet callbacks |
Buckets: duration `<30s, 30–60s, 1–3m, 3–10m, 10m+`; score buckets are fixed per mode in `TuningConfig`.
### 13.2 Rules
- No names, e-mail, device identifiers, precise locations, raw input or free text in parameters. No `user_id`. No Firebase `setUserId`.
- Event/param names ≤ 40 chars, values ≤ 100 chars, none start with `firebase_`, `google_`, `ga_`.
- Events go through `AnalyticsService`. Offline queueing is the SDK's job.
- Setgreet's own `trackEvent`/`trackScreen` are **not used**, so analytics has one source.
### 13.3 Consent `[BLOCKER: legal review for EU/UK GDPR and Turkey KVKK]`
- Info.plist ships with Firebase Analytics collection **disabled** by default; it is enabled at runtime only when consent = granted.
- Consent UI: Google UMP form where required by region; elsewhere a single in-app prompt after the Setgreet flow ("Share anonymous usage statistics?" Allow / Not now). The Settings toggle changes it at any time.
- The privacy policy and App Privacy answers MUST mention Firebase's automatic identifiers (app instance ID), Setgreet's anonymous ID and AdMob data.

---

## 14. Ads (AdMob)
### 14.1 Allowed formats
| Format | MVP |
|---|---|
| Interstitial after game end | **Yes** |
| Banner, mid-game interstitial, rewarded, app-open | **No** |
### 14.2 Policy (`AdPolicy`, pure Dart, fully tested)
Show an interstitial only if **all** hold:
1. The just-finished game ended by game over (not abandoned) and lasted **≥ 30 s**. Shorter games are not counted and never show an ad.
2. `countedGames > 3` (first three counted games are ad-free).
3. `gamesSinceLastAd ≥ 3` (so ads appear after games 4, 7, 10, …).
4. ≥ 120 s since the last ad.
5. An ad is already loaded (never wait for loading), the device is online, and ads are permitted by consent.
After showing: `gamesSinceLastAd = 0`, `lastAdAtEpochMs = now`. Any failure → silently continue.
### 14.3 Presentation
Results screen is shown first. The interstitial is presented when the player taps **Replay / Home / Back** from Results, then the chosen navigation continues when the ad is dismissed. Never during a run, pause, onboarding or the first launch session's first game. Preload the next interstitial when a run starts (if eligible).
### 14.4 Release verification `[BLOCKER]`
AdMob account and app/ad-unit IDs; test IDs in debug (real IDs via `--dart-define`); `GADApplicationIdentifier` and `SKAdNetworkItems` in Info.plist; privacy manifest; UMP consent for EEA/UK; decide ATT: **MVP posture = no ATT prompt, non-personalised ads, no IDFA use** `[verify against current Google and Apple documentation before submission]`; App Privacy answers; age rating; app not directed to children.

---

## 15. Setgreet boundary
- Official package: `setgreet` on pub.dev (Android/iOS). As observed on 17 Oct 2026: `Setgreet.initialize(appKey)` after `WidgetsFlutterBinding.ensureInitialized()`, `Setgreet.showFlow(flowId)`, flow callbacks via `Setgreet.setFlowCallbacks` / `Setgreet.flowEvents`. **Confirm exact signatures against the pinned version before coding.**
- `[BLOCKER]` Setgreet account, app key and first-launch flow ID. The flow is built and managed in the Setgreet dashboard; the app team builds no onboarding screens or assets.
- `OnboardingGateway.runFirstLaunchFlow() → OnboardingResult {completed, dismissed, error, timeout, skipped}`.
  - Runs only if `onboardingResolved == false`; sets it on `completed`/`dismissed`.
  - Timeout 8 s; on error/timeout/offline → continue to Home and retry on next launch, max 3 attempts, then stop.
  - Keys come from `--dart-define=SETGREET_APP_KEY=…`, `SETGREET_FLOW_ID=…`. Missing → `NoopOnboardingGateway`.
- Not used: `identifyUser`, purchases/paywalls, `trackEvent`, permission flows (so no extra Info.plist usage strings).
- "How to Play" (§5.2) is separate from Setgreet and is not a general tutorial engine.

---

## 16. Test plan
Engine tests use `ScriptedPieceSource` (fixed piece list) and a fake clock. Boards in examples: `#` filled, `.` empty.

### 16.1 Engine / PRNG
| ID | Case |
|---|---|
| E-01 | PRNG matches golden vectors (§6.4) for seeds 1 and 12345 |
| E-02 | Same seed + same inputs → identical final state hash (all modes, 200 random inputs) |
| E-03 | Piece pick never returns the same shape 3× in a row (while re-pick available) |
| E-04 | 4× clockwise rotation returns the original cells for all 10 shapes |
| E-05 | Distinct-orientation counts: I3=2, V3=4, O4=1, P5=4, U5=4, V5=4, T5=4, X5=1, W5=4, Z5=2 |
| E-06 | `canPlace` rejects out-of-bounds and overlap, accepts exact fit |
| E-07 | Kick order: piece at wall rotates using `(−1,0)` first; no kick available → rotation ignored |
| E-08 | Snapshot → JSON → restore → continue gives the same states as an uninterrupted run |
| E-09 | `rotateCW` cycle is down→left→up→right→down |
| E-10 | Lock delay: restarts max 4 times, then locks; hard drop locks immediately |

### 16.2 SHIFT
| ID | Case |
|---|---|
| S-01 | Compaction left: 4×4 `.... / .#.. / #.#. / .### ` → `.... / #... / ##.. / ###.` |
| S-02 | Rotation down→left on 3×3 `#.. / .#. / ##.` (no active piece): compaction gives `#.. / #.. / ##.`, column 0 is full → cleared (1 line, chain 1) → compaction → only `(2,0)` filled; score `100`, totalLines 1 |
| S-03 | Plain lock with an overhang does **not** compact (`[A4]`) |
| S-04 | Line clear compacts remaining cells along gravity; chain step ×2 applied when compaction completes another line |
| S-05 | Simultaneous full lines (2) score `300`; 3 lines score `600` |
| S-06 | Rotation with 0 charges: ignored, state identical |
| S-07 | Rotation rejected when next piece footprint would be blocked: charge unchanged, state identical (byte-compare) |
| S-08 | Rotation accepted when blocked only by the active piece (dry run ignores it) |
| S-09 | Active piece stays on its cells after rotation, then moves along new gravity on the next tick |
| S-10 | Settled cells treat the active piece as an obstacle during compaction; no overlap ever |
| S-11 | Spawn position: gravity down → row 0 centred; right → column 0 centred; footprint oriented to gravity |
| S-12 | Lines perpendicular to gravity only: with gravity left, a full **row** is not cleared, a full **column** is |
| S-13 | Charges: start 3; clear 3 lines → no change at max; spend 1 then clear 3 lines → back to 3; progress frozen at max |
| S-14 | Two rotations requested in one resolution: second ignored |
| S-15 | Game over when spawn footprint overlaps settled cell |
| S-16 | Rotation on empty board: no clears, no score, gravity changes |
| S-17 | Rotation producing a 2-line clear then a chain clear; events ordered correctly |
| S-18 | Pause during resolution: saved snapshot equals post-resolution state |

### 16.3 ECHO
| ID | Case |
|---|---|
| C-01 | After lock 5 the ghost equals `history[2]` cells exactly (absolute coordinates, original orientation) |
| C-02 | Ghost solidifies at lock 7, never earlier; cells appear only if free |
| C-03 | Piece locked onto a ghost cell at lock 6 cancels the ghost |
| C-04 | Ghost cancelled at birth if any cell occupied; no ghost, no retry until lock 10 |
| C-05 | Rows shifting into a ghost cell (via `LineShift`) cancel it at the end of that lock |
| C-06 | Mid-air solidify drops rigidly to rest; shape preserved |
| C-07 | Rigid drop completing a row clears it with ×2 multiplier |
| C-08 | Existing blocks never overwritten under any sequence (property test, 500 random seeds) |
| C-09 | No ghost created while another is pending (cannot occur; assert invariant) |
| C-10 | Falling piece passes through ghost cells without collision |
| C-11 | Snapshot/restore with pending ghost reproduces solidification timing |
| C-12 | Solidified echo cells clear in later lines like any other cell |
| C-13 | Solidify that blocks the spawn area → game over `spawn_blocked` |
| C-14 | `LineShift`: rows above a cleared row move down by count; relative shapes preserved |

### 16.4 MICRO
| ID | Case |
|---|---|
| M-01 | Legal placement only inside usable area and on empty cells |
| M-02 | Completing a row and a column together scores `lineScore(2)=300` |
| M-03 | Combo: clear, clear, no-clear resets; bonus `0, 50, then reset` |
| M-04 | 10th placement closes the bottom usable row; blocks in it deleted; score unchanged by deletion |
| M-05 | Closure completes a column → cleared silently, 0 points, combo unchanged |
| M-06 | A clear and the 10th placement together: clear scored first, then closure |
| M-07 | `usableRows` 1 → 0 ends run `area_exhausted` |
| M-08 | After closure no tray piece fits → immediate game over `no_move` |
| M-09 | Tray deal after 3 placements: `round += 1`, `+25` |
| M-10 | Fairness re-roll: same seed → same tray; at least one legal piece when possible; max 8 re-rolls |
| M-11 | Pieces unplaceable (no legal move) when board full → game over |
| M-12 | Marked row index = `usableRows − 1`; counter = `10 − placements % 10` |
| M-13 | Background during drag: piece returns to tray, snapshot saved |
| M-14 | Tap-select alternative places identically to drag |

### 16.5 Persistence, analytics, ads, onboarding
| ID | Case |
|---|---|
| P-01 | Atomic write survives simulated crash between tmp write and rename |
| P-02 | Corrupt JSON → defaults, file renamed `.corrupt`, no crash |
| P-03 | Future `schemaVersion` → defaults used, file not overwritten |
| P-04 | Migration v1→v2 stub round-trips |
| P-05 | Reset local data removes all files and calls analytics reset |
| P-06 | Best score is per mode and only increases |
| A-01 | `NoopAnalytics` used when consent ≠ granted: zero events dispatched |
| A-02 | No event contains a PII key; all names/values within Firebase limits |
| A-03 | `line_cleared` emitted once per resolution with `count` |
| A-04 | `game_end.reason=abandoned` when leaving via confirmation |
| D-01 | AdPolicy: games 1–3 never show |
| D-02 | AdPolicy: game 4 shows; games 5–6 don't; game 7 shows |
| D-03 | Game < 30 s: no ad, not counted |
| D-04 | < 120 s since last ad: no ad |
| D-05 | Not loaded / offline / consent denied: no ad, navigation unaffected |
| D-06 | Ad failure to present continues navigation within 1 frame |
| D-07 | Abandoned game never shows ad |
| O-01 | Gateway: success → `onboardingResolved=true`, Home |
| O-02 | Gateway: timeout/error/offline → Home, retry next launch, stops after 3 |
| O-03 | Missing keys → Noop gateway, Home |

### 16.6 Widget / golden
Home → each mode intro navigation (×3); mode intro shows best score; Play starts run; pause overlay resume/leave; back confirmation; results shows new best; settings toggles persist; reset confirmation; reduced motion renders static How-to-Play; Dynamic Type 1.5× with no overflow on Home, Mode intro, Settings at widths 320/375/430; golden of Home at 390×844; semantics labels present on all buttons.

### 16.7 Real iPhone smoke test (release gate)
Gestures, safe areas, 60 FPS on oldest supported device, background/foreground mid-run in each mode, airplane mode (launch, play, results), ad load failure, first launch with Setgreet flow and without network, consent flows, low-power mode, VoiceOver pass on menus, memory after 20 runs.

---

## 17. Playtest validation (assumptions)
| ID | Assumption | Evidence needed | If it fails |
|---|---|---|---|
| A1 | Click sound as MVP "sound" is acceptable | Tester feedback | Defer toggle/hide it |
| A2 | 10-piece catalog and weights feel fair | Game-over rate vs piece id (debug log); tester feedback | Retune weights, drop X5/Z5 |
| A3 | Scoring values and level curve | Median run 3–8 min in SHIFT/ECHO | Adjust `TuningConfig` |
| A4 | No compaction on plain lock is understandable | ≥ 8/10 testers predict result without help | Compact on every lock |
| A5 | Fairness re-roll improves MICRO feel without making it trivial | A/B with/without in 10 runs each | Disable re-roll |
| A6 | MICRO uses rows **and** columns | Run length and clarity | Rows only |
| A7 | MICRO typical run 45–90 s | Median `duration_bucket`; if < 45 s raise `shrinkEvery` to 12–14, if > 90 s lower it | Retune |
| A8 | SHIFT rotation guard rejection rate | `gravity_changed.rejected` < 10% of attempts | Show spawn-zone hint |
| A9 | ECHO ghost counterplay | Ghost cancel rate 20–60%; ghost-caused game over < 10% | Change window to 3 placements |
| A10 | MICRO closing the **bottom** row (blocks deleted, no score) is not an exploit | Observe dumping junk into the marked row | Make closing row impassable earlier, or award nothing and block placement there |
| A11 | Lock delay and hard-drop scores feel right | Tester feedback | Retune |
| A12 | Obstacle-aware compaction (SHIFT) is understandable | Testers do not call it a bug | Alternative: pause the piece during rotation |

---

## 18. Release checklist and blockers
**Blockers (outside code):** Apple Developer Program account, bundle ID, signing and macOS/Xcode (iOS builds cannot be produced on Linux); Firebase project and `GoogleService-Info.plist`; Setgreet account, app key, flow ID; AdMob app ID + interstitial ad unit; privacy policy URL; support URL; icon and screenshots; legal review of consent (§13.3).

**Gates:**
1. All tests in §16.1–16.6 green.
2. Real-device smoke test (§16.7) passed.
3. App Privacy answers cover Firebase, Setgreet and AdMob.
4. SDK privacy manifests present; SKAdNetwork list present.
5. Age rating answered; app not child-directed.
6. TestFlight build processed; internal test round done.
7. Submission is a goal, not a promise: Apple processing and review times are outside our control.

---

## 19. Build order for Codex
1. Project scaffold, `TuningConfig`, theme tokens, `Noop*` adapters. Home screen with `CustomPainter` hero and icons (match SVG v0.2).
2. Domain engine: board, shapes, rotation, PRNG (E-01…E-10).
3. MICRO (smallest rules) → ECHO → SHIFT, each with tests from §16.
4. Gameplay UI: board painter, controls, pause, results, How to Play.
5. Persistence + save/restore.
6. Settings, consent, analytics adapter.
7. Ads adapter + `AdPolicy`.
8. Setgreet adapter (after keys and signature verification).
9. iOS setup checklist, README, build commands (`flutter build ipa --release --dart-define=…`).
Report blockers before claiming submission readiness. Do not invent Setgreet API calls beyond what the pinned package exposes.

---

## Appendix A — Changes from v0.1
Reklam moved into MVP (AdMob, end-of-game interstitial); SHIFT board 8×12 → 10×10, charges 1→3 (max 2→3), active piece no longer teleports, rotation guard added; ECHO ghost source now deterministic (`p−3`, every 5th), cancel-on-overlap instead of skip, mid-air solidify drops rigidly; MICRO closes the **bottom** row (was top), drag-and-drop (was tap-tap), refresh removed, closure gives no score and silently clears shrink-completed lines; Firebase event list changed to owner list; state management, storage and DI fixed; Setgreet API verified from pub.dev (to be re-verified on pin); How to Play added as small separate feature; design inconsistencies in the home reference fixed.
