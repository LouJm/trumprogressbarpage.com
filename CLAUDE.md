# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-page static website deployed to GitHub Pages at `www.trumpprogressbarpage.com`. It shows the real-time percentage of Donald Trump's second presidential term (Jan 20, 2025 – Jan 20, 2029) as a silhouette fill animation and odometer, plus an embedded endless runner mini-game with a Firebase leaderboard.

No build system. No dependencies to install. The entire application lives in `index.html`.

## Development

Open `index.html` directly in a browser, or serve locally:

```bash
python3 -m http.server 8080
```

Deploy by pushing to the `main` branch — GitHub Pages serves it automatically via the `CNAME` file (`www.trumpprogressbarpage.com`).

## Architecture

Everything is in `index.html`. It has three logical sections:

### 1. Progress bar (inline `<script>` block, ~lines 462–720)
- `START`/`END` constants define Jan 20 2025 and Jan 20 2029 (UTC).
- `getRealPct()` computes the live percentage.
- `renderSilhouette(pct)` clips the filled silhouette image from the top: `inset(pct% 0 0 0)` drains it downward visually. `pct=0` → full, `pct=100` → empty.
- `renderOdometer(pct)` slides 10-digit CSS columns (`translateY(-Xem)`) to show the value. Digits are built once by `buildOdometer()`.
- The odometer color interpolates from orange (`#FD9F0F`) to red (`#FF3B30`) as `displayedPct` diverges from `realPct`, driven by `updateOdometerColor()`.
- **Boot animation**: silhouette drains from 0→TARGET over 6 s; odometer rolls 99.99→TARGET over 4.4 s. Skippable.
- **Physics**: `displayedPct` decays toward `realPct` each frame using `retain = 0.55^dt` (exponential decay). Live update runs every 60 s via `setInterval`.

### 2. Trump Runner mini-game (IIFE, ~lines 722–1256)
Canvas-based Dino-style runner. Launched by the Skip button or `window._startTrumpGame()`.
- `phase` state machine: `'intro'` → `'playing'` → `'over'`
- Double jump (2 jumps per airborne period), increasing speed, red block obstacles with political labels, spinning coin collectibles in 3 tiers.
- Score is stored as raw integer (divide by 1000 for %). HUD shows score as `X.XXX%` and running total progress bar %.
- Milestones (0.1%, 0.25%, 0.5%…) trigger large floating text.
- Combo multiplier for consecutive coin collection (max ×5). Invincibility frames after coin collection.
- Sprite images: `trump_run_1–4.png`, `trump_jump.png`, `trump_jump_rise.png`, `trump_fall.png`, `trump_idle.png`, `trump_land.png`.
- Sounds: `point.wav` (obstacle passed), `starTouched.mp3` (coin), `gameoversound.mp3` (death), `tick.wav` (live minute update).

### 3. Firebase leaderboard (ES module `<script type="module">`, ~lines 431–460)
Uses Firebase JS SDK v12 loaded from CDN (no npm).
- Firestore collection `scores`, fields: `name` (string, uppercased, max 20 chars), `score` (float, the `X.XXX%` value), `ts` (epoch ms).
- Scores are added with `addDoc` on every game-over submit (one doc per game, not upsert).
- `window._saveScore(name, scoreVal)` and `window._getLeaderboard()` are exposed as globals so the game IIFE can call them without module coupling.
- Leaderboard renders top 10 by score desc with 5 s timeout and `localStorage` cache (`lb_cache`).
- Player name persisted in `localStorage` (`playerName`) and auto-submitted on retry.

## Key constraints

- The silhouette uses two layered images: `trump_silhouette_filled.png` (clipped, behind) and `trump_silhouette_empty.png` (unclipped, in front). The empty outline must always be fully visible; only the fill is clipped.
- The odometer format is always `XX.XX%` (5 chars + `%`). `buildOdometer` must be called with a string of that shape before `renderOdometer` is used.
- Firebase API key is public (Firebase security rules enforce write-only from the browser; reads are open for leaderboard).
