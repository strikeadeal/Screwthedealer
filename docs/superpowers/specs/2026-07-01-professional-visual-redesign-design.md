# Screw the Dealer — Professional Visual Redesign

**Date:** 2026-07-01
**Status:** Approved design, pending implementation plan
**Scope:** Visual system + layout polish. Game logic, rules, and flow unchanged.

---

## Problem

Users report the game "does not feel professionally built." The cause is not a
lack of effort — it is the opposite. The current design imitates many physical
materials entirely in CSS gradients (felt weave, brass with painted-on specular
highlights, guilloché card backs, a wax "SCREWED" stamp, aged-paper grain, a torn
receipt). Faux-materials rendered in CSS cannot fake real lighting or texture, so
on a phone they land as banding, flatness, and muddiness. Combined with a
generic green-felt/brass/mahogany/red "casino app" palette, tracked-out uppercase
everywhere, a delicate high-contrast serif (Cormorant) that renders thin at UI
sizes, and low-contrast washed-out text, the result reads as amateur-but-ambitious.

A separate, concrete defect: fonts load via a Google Fonts `@import`, which
silently fails when the installed PWA is offline — unacceptable for an app people
add to their iPhone Home Screen and open in a bar with poor signal.

## Goal

Make the app read as professionally built through **restraint**: a tight, modern,
consistent visual system; real hierarchy from type and spacing rather than
decoration; one confident accent; flat surfaces with subtle real depth; and
craft concentrated on the few objects that deserve it.

## Non-goals

- No change to game rules, drink math, dealer-rotation logic, state shape,
  `localStorage` key, undo/history, or sound effects.
- No change to the set of screens or the flow between them.
- No framework, build step, bundler, or npm dependency. The app stays a single
  `index.html` (markup + inline `<style>` + one IIFE), per repo conventions.
- No rewrite of copy (a light touch only where a label is unclear).

---

## Organizing concept

**"The table goes dark; the cards stay warm."**

Every element is exactly one of two things, and that decides how it looks:

- **App chrome** (buttons, panels, meters, lists, backgrounds) → calm, flat,
  modern, near-black. This is the source of the "professional" feel: restraint,
  hierarchy, breathing room.
- **Physical game pieces** (the playing cards and the end-of-night receipt) → the
  *only* richly rendered, warm, tactile objects, on real ivory "paper."

A single thread of **amber** ties the two worlds together. This rule resolves
every design decision — is it app-chrome (clean/flat/modern/dark) or a game-piece
(crafted/warm/paper)? — and structurally eliminates the "everything is a
faux-material" problem: only two things get to be tactile, so we make those two
genuinely excellent.

---

## Color system

Replaces the entire mahogany/felt palette. **No green anywhere in the chrome.**
The accent is a **flat** color, not a gradient with painted highlights — this
alone removes most of the "fake brass" cheapness.

| Role | Token | Value |
|---|---|---|
| Background (warm near-black) | `--bg` | `#121013`, plus one very soft amber top-glow |
| Panel surface | `--surface` | `#1B181D` |
| Input / secondary surface | `--surface-hi` | `#241F26` |
| Hairline border | `--line` | `rgba(245,238,230,.09)` |
| Stronger hairline | `--line-strong` | `rgba(245,238,230,.16)` |
| Primary text (warm ivory) | `--text` | `#F4EEE4` |
| Muted text (WCAG-AA on `--bg`/`--surface`) | `--text-muted` | `#ABA49B` |
| Faint text (least important) | `--text-faint` | `#756E67` |
| **Accent — flat amber** | `--accent` | `#E7B24C` |
| Accent pressed/hi | `--accent-hi` | `#F6C86E` |
| Ink on accent | `--accent-ink` | `#1C1305` |
| Penalty red (on dark UI) | `--red` | `#E5615E` |
| Card / receipt paper | `--paper` | `#F2E9D6` |
| Suit red (on paper) | `--suit-red` | `#C23A32` |
| Ink on paper (black suits/text) | `--paper-ink` | `#211B14` |

Semantic use: **win = amber, penalty = red**. Focus-visible rings retinted amber.
`theme-color` and the iOS status-bar meta update to the near-black background.

Contrast requirement: all text meets WCAG-AA against its background, fixing the
current desaturated-sage `--text-dim` on dark green.

---

## Typography

From three fonts to a lean, sturdy two-voice system.

- **Display / headings / big names:** **Fraunces**, bold weight (~600–650) — a
  warm, modern, high-personality serif built for premium-but-characterful display.
  Sturdy at large sizes, unlike Cormorant.
- **All UI, body, and numbers:** the **iOS system font (SF Pro)** via an
  `-apple-system` stack — flawless on the target device, instant, and requires no
  network. Numbers use tabular figures so tab/deck counts align.
- **Letter-spacing dialed down** from `.16em`–`.26em` everywhere to near-zero,
  with light tracking only on small eyebrow labels. **All-caps used sparingly**,
  not on every button, header, and label.

Dropped: Cormorant and Courier Prime. The receipt/tab numbers use SF Pro tabular
figures rather than a third (mono) font.

### Font delivery — offline-safe (decided)

Self-host a **subset `Fraunces` `.woff2`** (only the glyphs actually used;
~20–30 KB) instead of the Google Fonts `@import`. Add the file to the `sw.js`
`ASSETS` list and bump the `CACHE` version so the installed PWA renders identically
with no network. This replaces a real offline defect, not just an aesthetic one.

---

## Components — app chrome (calm, flat, modern)

One consistent treatment replaces today's many bespoke bordered/inset panels.

- **Panels** (role card, deck meter, board, tab, rules): flat `--surface`, a
  single hairline `--line`, one soft drop shadow for elevation, consistent
  rounded corners. No inset fake-bevels, no brass edges.
- **Buttons:**
  - *Primary* → flat amber fill, `--accent-ink` text, subtle shadow, clear
    pressed state. (Removes the fake-brass gradient with painted specular
    highlights — the single biggest "cheap" tell.)
  - *Secondary* → surface fill + hairline. *Ghost* → text only.
  - One height, one radius, minimal letter-spacing, normal case.
- **Value grid (A–K)** — the core interaction: clean tappable cells, unmistakable
  **amber "selected"** state, quiet dashed/dimmed "ruled-out / none-left" state.
  Gets the most polish since players spend the most taps here.
- **Meters & indicators:** remove the fake stacked "deck tiles" and brass disc
  badges. Deck progress becomes one slim clean bar with a tabular count. The
  miss-streak becomes three simple legible pips.
- **Leaderboard ("the tab"):** a clean ranked list, leader marked in amber,
  tabular numbers — a real leaderboard, not bordered rows.

## The two crafted objects (warm, tactile — where craft goes)

### 1. The playing card (the hero)

What players stare at every round, so it earns real detail:

- Crisp ivory paper (flat warm `--paper`, *no* heavy grain), proper rounded
  corners, a genuine soft drop shadow, thin refined border.
- **Real pip layouts** for number cards (classic 2–10 arrangements) instead of one
  giant center glyph; proper corner rank+suit; distinct centerpiece for A / J / Q /
  K. This is the single highest-ROI upgrade — correct card typography instantly
  reads "professionally built."
- A clean, tasteful card **back** (simple amber-on-dark emblem with a thin double
  frame) replacing the busy guilloché.
- Flip animation kept; easing refined.

### 2. The receipt (the closer)

Genuinely good and on-theme — kept and elevated: same ivory paper as the cards,
cleaner (less grain), better type hierarchy, perforated edge tuned. The card and
the receipt are the only two paper objects, so they feel like a matched set.

## Gimmicks cleaned up

- **Wax "SCREWED" stamp** (a lumpy radial-gradient blob): replaced with a refined
  penalty reveal — a clean stamped/embossed red treatment over the card, not a
  fake wax lump.
- **Brass glint sweep** on wins: kept, but subtler (a light sheen).

---

## Per-screen layout polish

Same screens, same flow, same rules — hierarchy and spacing improved.

- **Setup:** cleaner brand lockup, roomier player rows with bigger taps and clear
  focus, calmer rules disclosure.
- **Game (the hub, currently densest):** clear top-to-bottom hierarchy —
  dealer/guesser focal at top, slim deck bar, clean rank board, clean tab, sticky
  primary action. Far fewer competing borders.
- **First / second guess:** the value grid takes center stage; the higher/lower
  hint becomes one bold, confident directional cue instead of a tracked-caps box.
- **Reveal:** hero card center stage + a clean amber/red result banner.
- **Pass-the-phone (rotate):** calm, big name, unmistakably a hand-over moment.
- **Game over:** the elevated receipt.

---

## Motion

Keep every existing animation (screen transitions, card flip, deal fly-in, banner
pop, glint) but **refine the easing** — today's springs overshoot in a way that
reads cheap. Calmer, more intentional curves; slightly shorter durations.
`prefers-reduced-motion` support stays fully intact.

---

## Constraints (honored)

- **Single file.** Everything stays in `index.html` (markup + inline `<style>` +
  one IIFE). No framework, build, or npm.
- **Game logic untouched.** Zero changes to the engine, state shape, `STORE_KEY`,
  drink amounts, or flow. An in-progress saved game keeps working.
- **Offline-capable PWA.** The new subset `.woff2` is added to the `sw.js`
  `ASSETS` list and the `CACHE` version is bumped so returning users get the update.
- **iOS Home-Screen target.** Safe-area insets, `100dvh` height math, sticky
  footer, and `viewport-fit=cover` preserved. `theme-color` / status-bar meta
  updated to the new near-black.
- **Accessibility improved, not regressed:** text meets WCAG-AA; focus-visible
  rings kept (retinted amber); ARIA labels preserved.

## What explicitly does NOT change

Rules, drink math, dealer-rotation logic, undo/history, sound effects, most copy,
the set of screens, and the flow between them.

---

## Verification

Open `index.html` in a browser (or the live GitHub Pages URL after merge) and play
a round at a phone-width viewport — the existing manual-test approach. No new
tooling. Additionally confirm: fonts render with the network disabled (offline
PWA), text contrast passes AA, and an in-progress `localStorage` game still loads.
