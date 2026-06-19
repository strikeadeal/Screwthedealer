# Screw the Dealer

A self-contained, installable **Progressive Web App (PWA)** of the card-guessing
drinking game *Screw the Dealer*. It is designed for pass-and-play on a single
phone — typically an iPhone added to the Home Screen from Safari — for 2 to 8
players.

> This README is written primarily as **context for AI agents** working on the
> repository. It explains what the project is, how it is deployed, how the code
> is structured, and the conventions to follow when changing it.

---

## TL;DR for agents

- **Pure static front-end.** No build step, no framework, no dependencies, no
  package manager. Everything is hand-written HTML/CSS/vanilla JS.
- **The entire app lives in `index.html`** — markup, styles (`<style>`), and game
  logic (one IIFE in `<script>`). `manifest.json` and `sw.js` support PWA
  install/offline.
- **Deployment is GitHub Pages served directly from the `claude/screw-dealer-pwa-yer1cz`
  branch.** Pushing to that branch publishes the live site. There is no CI and no
  separate deploy step.
- **To change game behaviour or UI, edit `index.html`.** Keep the game-logic
  values and the user-facing copy in sync (see "Gotchas").
- **Test by opening `index.html` in a browser.** No server is required for the
  game itself; a static server is only needed to exercise the service worker.

---

## Repository contents

| File | Purpose |
|------|---------|
| `index.html` | The whole application: HTML structure, inline CSS, and the game engine + UI layer in a single inline `<script>` IIFE. |
| `manifest.json` | PWA manifest — app name, colors, `display: standalone`, `orientation: portrait`, and inline SVG (data-URI) icons. No external image files. |
| `sw.js` | Service worker. Cache-first strategy for offline play; falls back to the cached `index.html` shell for navigations. Bump the `CACHE` constant when shipping changes so clients pick them up. |
| `README.md` | This file. |

There are intentionally **no** other source files, lockfiles, or tooling.

---

## How it works (architecture)

The code in `index.html` is a single `(function(){ 'use strict'; ... })()` IIFE,
split into two layers:

### 1. Game engine (pure-ish state + transitions)

- A single `state` object is the source of truth. Shape (see `freshState`):
  ```
  {
    screen,        // which screen is showing (string)
    players,       // array of name strings (2–8)
    dealerIndex,   // index of current dealer
    deck,          // array of {value:1..13, suit:0..3}, shuffled (Fisher–Yates)
    discard,       // cards already played this deck
    reshuffles,    // how many times the deck has been exhausted
    current,       // the card currently in play, or null
    firstGuess,    // the guesser's first call (rank 1..13)
    hint,          // 'higher' | 'lower' after a wrong first call
    result,        // outcome of the round {type, amount, attempt, total, drinkerIndex, ...}
    tally,         // per-player running drink total ("on the tab")
    history        // snapshots for undo (capped at 8)
  }
  ```
- Cards are `{value, suit}` where `value` is 1–13 (A..K, via the `LABELS` array)
  and `suit` is an index into `SUITS` (`♠♥♦♣`).
- Key transitions: `startGame` → `deal` → `submitFirstGuess` → (`submitSecondGuess`)
  → `nextRound` → `rotateReady`. The deck reshuffles when exhausted; after
  `MAX_RESHUFFLES` (3) the game ends (`gameOver`).
- State is **persisted to `localStorage`** under `STORE_KEY` (`screwTheDealer.v1`)
  on every `save()`, and restored on `boot()` so a refresh resumes the game.

### 2. UI / DOM layer (screen state machine)

The UI is a set of `<section class="screen" data-screen="...">` panels; exactly one
is `.active` at a time. `render()` shows the screen named by `state.screen` and
calls the matching `renderXxx()` function. Screens:

`setup` → `game` (between rounds) → `firstGuess` → `secondGuess` →
`reveal` → `rotate` (pass the phone) → back to `game`; plus `gameOver`.

Other UX details: per-player drink **tally** ("on the tab"), an **undo/history**
buffer, a card-flip reveal animation (respects `prefers-reduced-motion`), and
haptic feedback via `navigator.vibrate`. Layout targets iPhone with safe-area
insets and a max width of 480px.

---

## Game rules (current)

When a card is drawn the guesser (player to the dealer's left) calls its value:

1. **First call correct →** the **dealer drinks 4**.
2. **First call wrong →** the dealer says *higher* or *lower*.
3. **Second call correct →** the **dealer drinks 2**.
4. **Second call wrong (after two guesses) →** the **guesser drinks 1**.

The card then goes to the discard/board and the deal passes left.

> These numbers live in **both** the engine (`submitFirstGuess`,
> `submitSecondGuess`, including the `tally` increments) **and** the on-screen
> copy (the "How to play" list and the screen subtitles). Always update both.

---

## Local development & testing

No install needed.

- **Quickest:** open `index.html` directly in a browser. The game runs fully from
  the local file. State persists via `localStorage`.
- **To exercise the PWA / service worker** (which requires an `http(s)` origin),
  serve the folder statically, e.g.:
  ```
  python3 -m http.server 8000
  # then visit http://localhost:8000
  ```
- There is **no test suite, linter, or build**. Validate changes by playing
  through a round on a narrow (mobile-width) viewport.

---

## Deployment

- **GitHub Pages serves the live site directly from the
  `claude/screw-dealer-pwa-yer1cz` branch.** A push to that branch publishes;
  Pages typically rebuilds within a minute or two.
- When changing app files, **bump the `CACHE` version string in `sw.js`** so the
  service worker invalidates old caches and clients receive the update instead of
  serving stale assets.
- ⚠️ The `main` branch and this Pages branch have **diverged**; this branch is the
  one that is actually deployed and is generally more complete. Make user-facing
  changes here (or ensure they reach here) so they go live.

---

## Conventions & gotchas for agents

- **Single-file app:** prefer editing `index.html` in place; do not introduce a
  framework, bundler, or npm dependency without explicit instruction.
- **Vanilla ES5-ish style:** the existing script uses `var`, function
  declarations, and IIFE scoping for maximum mobile-Safari compatibility. Match
  it rather than introducing modules or modern syntax piecemeal.
- **Keep logic and copy in sync:** drink amounts, rule text, and subtitles appear
  in multiple places — grep for the numbers/phrases and update all of them.
- **Persisted state shape:** if you change the `state` object's shape, consider
  the `STORE_KEY` / load-guard logic so an old saved game in `localStorage`
  doesn't break `boot()` (bump the key or guard the new fields).
- **Service worker caching:** changes won't appear for returning users until the
  `CACHE` constant in `sw.js` is bumped.
- **No external assets:** icons are inline SVG data-URIs in `manifest.json`; fonts
  are loaded from Google Fonts via `@import`. Keep the app installable and
  offline-capable.
