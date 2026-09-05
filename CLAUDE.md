# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-page static website deployed to GitHub Pages at `www.trumpprogressbarpage.com`. It shows the real-time percentage of Donald Trump's second presidential term (Jan 20, 2025 – Jan 20, 2029) as a silhouette fill animation and odometer, plus an embedded endless runner mini-game with a Firebase leaderboard. It also hosts a promo popup for a third-party IQ test app, gated behind a URL query param.

No build system. No dependencies to install. The core application lives in `index.html`; a handful of standalone HTML files are thin app-store redirect/landing pages for Instagram bio-link promos.

## Development

Open `index.html` directly in a browser, or serve locally:

```bash
python3 -m http.server 8080
```

There is no build, lint, or test tooling in this repo — verify changes by opening the page(s) in a browser (and testing mobile/in-app-browser behavior on a real device or via UA spoofing, since several flows key off `navigator.userAgent`).

Deploy by pushing to the `main` branch — GitHub Pages serves it automatically via the `CNAME` file (`www.trumpprogressbarpage.com`).

## Architecture

### `index.html`

Everything for the core site lives here, in four logical sections:

#### 1. Progress bar (inline `<script>`, ~lines 586–845)
- `START`/`END` constants define Jan 20 2025 and Jan 20 2029 (UTC).
- `getRealPct()` computes the live percentage.
- `renderSilhouette(pct)` clips the filled silhouette image from the top: `inset(pct% 0 0 0)` drains it downward visually. `pct=0` → full, `pct=100` → empty.
- `renderOdometer(pct)` slides 10-digit CSS columns (`translateY(-Xem)`) to show the value. Digits are built once by `buildOdometer()`.
- The odometer color interpolates from orange (`#FD9F0F`) to red (`#FF3B30`) as `displayedPct` diverges from `realPct`, driven by `updateOdometerColor()`.
- **Boot animation**: silhouette drains from 0→TARGET over 6 s; odometer rolls 99.99→TARGET over 4.4 s. Skippable.
- **Physics**: `displayedPct` decays toward `realPct` each frame using `retain = 0.55^dt` (exponential decay). Live update runs every 60 s via `setInterval`.

#### 2. Trump Runner mini-game (IIFE, ~lines 847–1399)
Canvas-based Dino-style runner. Launched by the Skip button or `window._startTrumpGame()`.
- `phase` state machine: `'intro'` → `'playing'` → `'over'`
- Double jump (2 jumps per airborne period), increasing speed, red block obstacles with political labels, spinning coin collectibles in 3 tiers.
- Score is stored as raw integer (divide by 1000 for %). HUD shows score as `X.XXX%` and running total progress bar %.
- Milestones (0.1%, 0.25%, 0.5%…) trigger large floating text.
- Combo multiplier for consecutive coin collection (max ×5). Invincibility frames after coin collection.
- Sprite images: `trump_run_1–4.png`, `trump_jump.png`, `trump_jump_rise.png`, `trump_fall.png`, `trump_idle.png`, `trump_land.png`.
- Sounds: `point.wav` (obstacle passed), `starTouched.mp3` (coin), `gameoversound.mp3` (death), `tick.wav` (live minute update).

#### 3. Firebase leaderboard (ES module `<script type="module">`, ~lines 555–584)
Uses Firebase JS SDK v12 loaded from CDN (no npm).
- Firestore collection `scores`, fields: `name` (string, uppercased, max 20 chars), `score` (float, the `X.XXX%` value), `ts` (epoch ms).
- Scores are added with `addDoc` on every game-over submit (one doc per game, not upsert).
- `window._saveScore(name, scoreVal)` and `window._getLeaderboard()` are exposed as globals so the game IIFE can call them without module coupling.
- Leaderboard renders top 10 by score desc with 5 s timeout and `localStorage` cache (`lb_cache`).
- Player name persisted in `localStorage` (`playerName`) and auto-submitted on retry.

#### 4. App promo popup (inline `<script>`, ~lines 1401–1478)
Inert unless the page is loaded with `?promo=iqtest` or `?promo=adhdtest` in the URL, so it never affects normal visitors. One shared popup serves both apps; the `PROMOS` config object maps the query value to that app's eyebrow text and App/Play Store URLs, and `$('iqp-eyebrow').textContent` is set from it at runtime. Add a new app promo by adding a `PROMOS` entry — do not duplicate the popup markup/script.
- Markup (`#iqp-overlay`, `#iqp-card`, etc.) sits in the body near the bottom, styled in the main `<style>` block; overlay uses `backdrop-filter: blur()` over the real (unmodified) homepage content behind it.
- On mobile (iOS/Android UA), for **non**-in-app browsers it attempts an immediate silent redirect to the App/Play Store; only reveals the popup if the page is still visible after ~1.2 s (checked live via `document.hidden`, not a sticky flag — a transient native "leave app?" prompt can blur/hide the page without actually navigating away).
- For **known in-app browsers** (Instagram/FB/TikTok/Snapchat/etc., matched via UA regex) it skips the blind auto-redirect entirely — an unsolicited store navigation can hang or wipe the page in these WebViews — and shows the popup immediately instead.
- Tapping Continue retries the store link with a real user gesture; if that also stalls, the button is replaced with a fixed-position arrow pointing at the browser's own menu button (top-right), telling the user to open the page in an external browser.
- A quit button (and an "×" on the card) hides the popup and cleans `?promo=...` off the URL via `history.replaceState`, leaving the user on the normal homepage.
- Desktop UAs are unaffected — the script no-ops for non-mobile UAs.

### Standalone app-store redirect pages

Separate from `index.html`, used as Instagram bio-link landing pages for third-party apps. All four are byte-identical in pattern (differ only in the `?promo=` value/title): thin redirects (`window.location.replace(...)`) into the shared popup at `index.html?promo=<name>`. Kept as separate files/filenames only so old or cached bio links keep resolving — when editing this flow, edit the popup in `index.html`, not these files.
- `iq-test.html`, `iqapp.html`, `IQTEST.html` → `?promo=iqtest`
- `adhdapp.html` → `?promo=adhdtest`

## Key constraints

- The silhouette uses two layered images: `trump_silhouette_filled.png` (clipped, behind) and `trump_silhouette_empty.png` (unclipped, in front). The empty outline must always be fully visible; only the fill is clipped.
- The odometer format is always `XX.XX%` (5 chars + `%`). `buildOdometer` must be called with a string of that shape before `renderOdometer` is used.
- Firebase API key is public (Firebase security rules enforce write-only from the browser; reads are open for leaderboard).
- Any code path in the app-store-redirect flows that checks "did the browser navigate away" must re-check live state (e.g. `document.hidden`) at the moment it acts, not latch a sticky flag from a past `blur`/`visibilitychange` event — in-app browsers (esp. Instagram) can fire those on a native interstitial without any real navigation, which permanently soft-locks a sticky-flag implementation on a blank page.
