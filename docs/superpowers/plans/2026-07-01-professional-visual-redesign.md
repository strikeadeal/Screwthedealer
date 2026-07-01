# Professional Visual Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Screw the Dealer PWA read as professionally built by replacing faux-material CSS skeuomorphism with a restrained modern system — dark chrome, two warm crafted paper objects (card + receipt), one flat amber accent — without touching game logic.

**Architecture:** The app is a single `index.html` (HTML + inline `<style>` + one IIFE `<script>`). This plan rewrites the `<style>` block section by section and makes small, surgical DOM-markup changes inside a few `render*` JS functions (card, board, receipt). The engine, state shape, and screen flow are untouched. Supporting files `sw.js` (cache), `manifest.json` (theme + icons), and a new self-hosted font under `fonts/` complete the change.

**Tech Stack:** Hand-written HTML5, CSS custom properties, vanilla ES5-style JS. No build, bundler, framework, or npm. One self-hosted `.woff2` (Fraunces, OFL). Deployed as static files via GitHub Pages from `main`.

## Global Constraints

Every task's requirements implicitly include these. Values copied verbatim from the spec:

- **Single file for the app.** All markup, styles, and logic stay in `index.html` (inline `<style>` + one IIFE). No framework, build step, bundler, or npm dependency.
- **Game logic, rules, drink math, dealer-rotation, undo/history, sound, state shape, and `STORE_KEY` (`screwTheDealer.v1`) are unchanged.** An in-progress saved game must still load.
- **Screens and flow are unchanged:** `setup → game → firstGuess → secondGuess → reveal → rotate → game`, plus `gameOver`.
- **Offline-capable PWA:** any new asset is added to the `sw.js` `ASSETS` list, and the `CACHE` constant is bumped every ship (current: `screw-the-dealer-v12`).
- **iOS Home-Screen target:** preserve safe-area insets (`--sat/--sar/--sab/--sal`), `--visible-h` height math, the sticky game footer, and `viewport-fit=cover`.
- **Accessibility:** all text meets WCAG-AA contrast; keep `:focus-visible` rings (retinted amber); keep existing ARIA labels.
- **Palette:** no green anywhere in the chrome. Win = amber, penalty = red. Accent is a flat color, never a gradient with painted highlights.
- **Type:** display = Fraunces (weight 600 only — the single self-hosted weight; never request 700 for Fraunces or it faux-bolds); everything else = iOS system stack; numbers use `tabular-nums`.
- **Verification is manual/visual** (there is no test suite, linter, or build): open `index.html` in a browser at a phone-width viewport and observe. Automated `grep` checks are used where they add safety.

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `index.html` `<head>` | meta (`theme-color`, fonts), `@font-face` | Modify |
| `index.html` `<style>` | entire visual system | Rewrite section by section (Tasks 1–11) |
| `index.html` `<script>` | `renderFullCard`, `renderMiniCard`, `renderGameOver`, add `suitSym`/`PIPS` helpers, replace `spawnWaxStamp` | Small surgical edits (Tasks 4, 7, 8, 10) |
| `fonts/fraunces-600.woff2` | self-hosted display font (offline-safe) | Create (Task 1) |
| `sw.js` | offline cache list + version | Modify (Task 1) |
| `manifest.json` | install theme colors + icon art | Modify (Task 1) |
| `apple-touch-icon.png` | iOS Home-Screen raster icon | Regenerate if tooling available, else flagged follow-up (Task 1) |

**Dev tip — reaching each screen to verify:** serve locally with `python3 -m http.server 8000` and open `http://localhost:8000` (needed for the service worker; plain file-open works for everything except SW/offline tests). Click-path: Setup → *Start game* → `game`; *Deal a card* → `firstGuess`; pick a value + *Lock in call* → (`secondGuess` if wrong) → `reveal`; *Next round* → `rotate`; *Ready* → back to `game`. To reach `gameOver` fast, temporarily shrink the deck (see Task 10), then revert.

---

## Task 1: Foundation — fonts, tokens, background, PWA plumbing

**Files:**
- Create: `fonts/fraunces-600.woff2`
- Modify: `index.html` (`<head>` meta + `@import` line ~14; `:root` tokens lines ~16–53; `body` lines ~59–74)
- Modify: `sw.js:1-2`
- Modify: `manifest.json` (lines 9–10 colors; icon `src` data-URIs)
- Regenerate (if tooling): `apple-touch-icon.png`

**Interfaces:**
- Produces (CSS custom properties every later task consumes):
  - Colors: `--bg`, `--surface`, `--surface-hi`, `--line`, `--line-strong`, `--text`, `--text-muted`, `--text-faint`, `--accent`, `--accent-hi`, `--accent-ink`, `--red`, `--paper`, `--suit-red`, `--paper-ink`
  - Fonts: `--display` (Fraunces 600), `--ui` (system stack), `--data` (= `--ui`, used with `tabular-nums`)
  - Radii: `--r-sm:10px`, `--r-md:14px`, `--r-lg:20px`, `--r-card:14px`
  - Shadows: `--sh-1`, `--sh-2`, `--sh-card`, `--sh-accent`
  - Unchanged and still relied upon: `--sp-1..--sp-6`, `--tap`, safe-area vars, `--visible-h`, z-index vars

- [ ] **Step 1: Download and self-host the Fraunces 600 subset**

```bash
cd "$(git rev-parse --show-toplevel)"
mkdir -p fonts
# Google returns woff2 URLs, latin subset last. Grab the last woff2.
curl -sH 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)' \
  'https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,600&display=swap' \
  -o /tmp/fraunces.css
URL=$(grep -oE 'https://[^ )]+\.woff2' /tmp/fraunces.css | tail -1)
echo "Latin woff2: $URL"
curl -s "$URL" -o fonts/fraunces-600.woff2
```

- [ ] **Step 2: Verify the font file is a real woff2 of reasonable size**

```bash
ls -la fonts/fraunces-600.woff2
# Expect a file roughly 25-70 KB. Confirm the wOF2 magic header:
head -c 4 fonts/fraunces-600.woff2 | xxd | grep -qi '774f 4632' && echo "OK: valid woff2" || echo "FAIL: not a woff2"
```
Expected: `OK: valid woff2`. If FAIL, the URL grab picked an HTML error page — open `/tmp/fraunces.css`, copy the last `.woff2` URL manually, and re-download. (Fallback: download Fraunces from https://fonts.google.com/specimen/Fraunces, then `pip install fonttools` and `pyftsubset Fraunces.ttf --output-file=fonts/fraunces-600.woff2 --flavor=woff2 --unicodes=U+0000-00FF,U+2013-2014,U+2018-201D,U+2022,U+2026,U+2660,U+2663,U+2665,U+2666,U+FE0E` against the 600 instance.)

- [ ] **Step 3: Replace the Google Fonts `@import` with a self-hosted `@font-face`**

In `index.html`, replace the `@import url(...)` line (~14) inside `<style>` with:

```css
@font-face{
  font-family:'Fraunces';
  font-style:normal;
  font-weight:600;
  font-display:swap;
  src:url('fonts/fraunces-600.woff2') format('woff2');
}
```

- [ ] **Step 4: Replace the `:root` token block**

Replace the entire `:root{ ... }` block (lines ~16–53) with:

```css
:root{
  --bg:#121013;
  --surface:#1B181D;
  --surface-hi:#241F26;
  --line:rgba(245,238,230,.09);
  --line-strong:rgba(245,238,230,.16);

  --text:#F4EEE4;
  --text-muted:#ABA49B;
  --text-faint:#756E67;

  --accent:#E7B24C;
  --accent-hi:#F6C86E;
  --accent-ink:#1C1305;

  --red:#E5615E;

  --paper:#F2E9D6;
  --suit-red:#C23A32;
  --paper-ink:#211B14;

  --display:'Fraunces',Georgia,serif;
  --ui:-apple-system,BlinkMacSystemFont,'SF Pro Text','Segoe UI',system-ui,sans-serif;
  --data:var(--ui);

  --sp-1:6px; --sp-2:10px; --sp-3:16px; --sp-4:24px; --sp-5:36px; --sp-6:52px;
  --r-sm:10px; --r-md:14px; --r-lg:20px; --r-card:14px;
  --tap:52px;

  --sh-1:0 1px 2px rgba(0,0,0,.4);
  --sh-2:0 8px 24px rgba(0,0,0,.38);
  --sh-card:0 24px 60px rgba(0,0,0,.55), 0 6px 16px rgba(0,0,0,.4);
  --sh-accent:0 6px 18px rgba(231,178,76,.18), 0 2px 4px rgba(0,0,0,.4);

  --z-screen:1; --z-banner:5; --z-toast:9; --z-float:40;

  --sat:env(safe-area-inset-top, 0px);
  --sar:env(safe-area-inset-right, 0px);
  --sab:env(safe-area-inset-bottom, 0px);
  --sal:env(safe-area-inset-left, 0px);
  --visible-h:calc(100dvh - var(--sat) - var(--sab));
}
```

- [ ] **Step 5: Replace the `body` background (kill felt weave + vignette)**

Replace the `background-color` + `background-image` + `background-attachment` declarations in the `body` rule (~62–67) with:

```css
  background-color:var(--bg);
  background-image:radial-gradient(120% 80% at 50% -10%, rgba(231,178,76,.06) 0%, rgba(231,178,76,0) 42%);
  background-attachment:fixed;
```
Leave the rest of the `body` rule (font-family, color, padding with safe-area insets, `min-height:100dvh`, `overflow-x:hidden`) unchanged.

- [ ] **Step 6: Update the `<head>` theme-color meta**

In `index.html` `<head>`, change `<meta name="theme-color" content="#1E0F0A">` to:

```html
<meta name="theme-color" content="#121013">
```

- [ ] **Step 7: Update `manifest.json` colors and refresh the SVG icon art**

In `manifest.json` change:
```json
  "background_color": "#121013",
  "theme_color": "#121013",
```
Then replace **all three** icon `src` data-URIs with this new amber-on-near-black spade emblem (same string for the two `"purpose":"any"` icons; use the maskable variant below for the third). Any-purpose icon `src`:

```
data:image/svg+xml,%3Csvg%20xmlns=%22http://www.w3.org/2000/svg%22%20viewBox=%220%200%20512%20512%22%3E%3Crect%20width=%22512%22%20height=%22512%22%20rx=%2296%22%20fill=%22%23121013%22/%3E%3Ccircle%20cx=%22256%22%20cy=%22256%22%20r=%22188%22%20fill=%22none%22%20stroke=%22%23E7B24C%22%20stroke-width=%228%22%20opacity=%220.5%22/%3E%3Ctext%20x=%22256%22%20y=%22322%22%20font-family=%22Georgia,serif%22%20font-size=%22240%22%20text-anchor=%22middle%22%20fill=%22%23E7B24C%22%3E%26%239824;%3C/text%3E%3C/svg%3E
```

Maskable icon `src` (no bleed to the circle edge — emblem kept inside the safe zone):

```
data:image/svg+xml,%3Csvg%20xmlns=%22http://www.w3.org/2000/svg%22%20viewBox=%220%200%20512%20512%22%3E%3Crect%20width=%22512%22%20height=%22512%22%20fill=%22%23121013%22/%3E%3Ctext%20x=%22256%22%20y=%22312%22%20font-family=%22Georgia,serif%22%20font-size=%22200%22%20text-anchor=%22middle%22%20fill=%22%23E7B24C%22%3E%26%239824;%3C/text%3E%3C/svg%3E
```

- [ ] **Step 8: Regenerate the iOS raster icon (best-effort)**

The iOS Home-Screen icon is the raster `apple-touch-icon.png` (iOS ignores SVG). Regenerate it from the new emblem if a renderer is available:

```bash
cd "$(git rev-parse --show-toplevel)"
cat > /tmp/icon.svg <<'SVG'
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 180 180"><rect width="180" height="180" rx="34" fill="#121013"/><text x="90" y="120" font-family="Georgia,serif" font-size="96" text-anchor="middle" fill="#E7B24C">&#9824;</text></svg>
SVG
if command -v rsvg-convert >/dev/null 2>&1; then
  rsvg-convert -w 180 -h 180 /tmp/icon.svg -o apple-touch-icon.png && echo "regenerated via rsvg-convert"
elif command -v magick >/dev/null 2>&1; then
  magick -background none -density 300 /tmp/icon.svg -resize 180x180 apple-touch-icon.png && echo "regenerated via imagemagick"
else
  echo "NO SVG RENDERER: leaving apple-touch-icon.png as-is — see follow-up note."
fi
```
If no renderer is present, **leave the file unchanged** and note in the PR description: "apple-touch-icon.png still shows the old art; regenerate from /tmp/icon.svg with an SVG rasterizer as a follow-up." Do not hand-edit the binary.

- [ ] **Step 9: Add the font to the service worker and bump the cache**

In `sw.js` replace lines 1–2 with:

```javascript
const CACHE = 'screw-the-dealer-v13';
const ASSETS = ['./', './index.html', './manifest.json', './apple-touch-icon.png', './fonts/fraunces-600.woff2'];
```

- [ ] **Step 10: Verify the foundation**

```bash
# No stale palette tokens should remain anywhere:
grep -nE '--felt|--mahogany|--brass|--ivory-shade|--card-red|--text-dim|Cormorant|Courier Prime|fonts.googleapis' index.html || echo "OK: no stale tokens/imports"
```
Then open `index.html` at ~390px width. Expect: warm near-black background with a faint amber glow at top (no felt weave, no green), body text in ivory. Headings may look like the serif fallback until later tasks apply `--display`; that's fine. Load with **Network throttled to Offline** (DevTools) after one online load and confirm the page + font still render (service worker cached the woff2).

- [ ] **Step 11: Commit**

```bash
git add index.html sw.js manifest.json fonts/fraunces-600.woff2 apple-touch-icon.png
git commit -m "Redesign foundation: midnight-amber tokens, self-hosted Fraunces, dark background, PWA colors"
```

---

## Task 2: Typography roles, buttons, actions, sound toggle, focus rings

**Files:**
- Modify: `index.html` `<style>` — TYPOGRAPHY ROLES (~109–153), BUTTONS (~158–236), SOUND TOGGLE (~1008–1027), FOCUS-VISIBLE (~1097–1105)

**Interfaces:**
- Consumes: all tokens from Task 1.
- Produces: `.btn`, `.btn-primary/.btn-secondary/.btn-danger/.btn-ghost/.btn-block/.btn-lg`, `.actions`, `.eyebrow/.title/.subtitle/.screen-head/.brand-rule`, `.sound-toggle` — restyled but with identical class names/DOM, so JS is untouched.

- [ ] **Step 1: Replace the TYPOGRAPHY ROLES block**

Replace lines ~109–153 with:

```css
.eyebrow{
  font-family:var(--ui);
  font-size:.7rem;font-weight:600;
  letter-spacing:.14em;text-transform:uppercase;
  color:var(--accent);
}
.title{
  font-family:var(--display);font-weight:600;
  line-height:1.0;font-size:clamp(2.8rem,15vw,4rem);
  color:var(--text);letter-spacing:-.01em;
}
.title .amp{color:var(--accent);font-style:italic;font-weight:600}
.subtitle{
  font-family:var(--ui);color:var(--text-muted);
  font-size:1rem;line-height:1.55;max-width:34ch;
}
.brand{display:flex;flex-direction:column;gap:var(--sp-2)}
.brand-rule{height:1px;background:var(--line-strong);margin-top:var(--sp-2)}
.screen-head{display:flex;flex-direction:column;gap:var(--sp-1)}
.screen-head h2{
  font-family:var(--display);font-weight:600;
  font-size:clamp(1.9rem,8vw,2.4rem);
  color:var(--text);line-height:1.05;letter-spacing:-.01em;
  text-transform:none;
}
```

- [ ] **Step 2: Replace the BUTTONS block**

Replace lines ~158–236 with:

```css
.btn{
  display:inline-flex;align-items:center;justify-content:center;gap:10px;
  min-height:var(--tap);padding:0 var(--sp-4);
  border-radius:var(--r-md);
  font-family:var(--ui);font-weight:600;font-size:1rem;
  letter-spacing:.005em;
  transition:filter .12s ease, transform .08s ease, background .15s ease;
  -webkit-tap-highlight-color:transparent;user-select:none;
}
.btn-primary{
  background:var(--accent);color:var(--accent-ink);font-weight:700;
  box-shadow:var(--sh-accent);
}
.btn-primary:active{background:var(--accent-hi);transform:translateY(1px)}
.btn-primary:disabled{background:var(--surface-hi);color:var(--text-faint);box-shadow:none;cursor:default}

.btn-secondary{
  background:var(--surface-hi);color:var(--text);
  border:1px solid var(--line);
}
.btn-secondary:active{background:#2C2730}

.btn-danger{
  background:transparent;color:var(--red);
  border:1px solid rgba(229,97,94,.4);
}
.btn-danger:active{background:rgba(229,97,94,.08)}

.btn-ghost{background:transparent;color:var(--text-muted);min-height:48px;font-size:.95rem}
.btn-ghost:active{color:var(--text)}

.btn-block{width:100%}
.btn-lg{min-height:60px;font-size:1.05rem}

.actions{display:flex;flex-direction:column;gap:var(--sp-2);margin-top:auto}

.screen[data-screen="game"] .actions{
  position:sticky;bottom:0;z-index:5;
  margin-left:calc(-1 * var(--sp-3));margin-right:calc(-1 * var(--sp-3));
  margin-bottom:calc(-1 * var(--sp-6));
  padding:var(--sp-3) var(--sp-3) calc(var(--sab) + var(--sp-3));
  background:linear-gradient(180deg, rgba(18,16,19,0) 0%, var(--bg) 34%);
  backdrop-filter:blur(6px);-webkit-backdrop-filter:blur(6px);
}
```

- [ ] **Step 3: Replace the SOUND TOGGLE block (flat, no brass)**

Replace lines ~1008–1027 with:

```css
.sound-toggle{
  position:fixed;
  top:calc(var(--sat) + 8px);right:calc(var(--sar) + 8px);
  z-index:var(--z-float);width:40px;height:40px;border-radius:50%;
  background:var(--surface);border:1px solid var(--line);
  box-shadow:var(--sh-1);color:var(--text-muted);
  display:flex;align-items:center;justify-content:center;
  cursor:pointer;-webkit-tap-highlight-color:transparent;
  transition:color .12s ease, background .12s ease, transform .08s ease;
}
.sound-toggle:active{transform:translateY(1px);color:var(--text)}
```

- [ ] **Step 4: Retint the focus rings**

In the `:focus-visible` rule (~1097–1105) change `outline:2px solid var(--brass-hi);` to:

```css
  outline:2px solid var(--accent-hi);
```

- [ ] **Step 5: Verify**

Open the Setup screen. Expect: "Screw *the* Dealer" title in the Fraunces serif with an amber italic "the"; the *Start game* button a flat amber pill with dark text (no brass gradient/bevel), darkening on tap; *Add player* a subtle outlined surface button. Tab to a button and confirm an amber focus ring. The sound toggle (top-right) is a quiet dark circle, not a brass coin.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Redesign chrome: flat amber buttons, sturdier type roles, quiet sound toggle"
```

---

## Task 3: Setup screen — rules disclosure, player rows, form error

**Files:**
- Modify: `index.html` `<style>` — RULES DISCLOSURE (~241–270), PLAYER SETUP (~275–318)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled `.rules/.rules-body`, `.players/.player-row/.player-seat/.player-input/.player-remove/.add-player/.form-error`. DOM/classes unchanged (JS in `makePlayerRow` untouched).

- [ ] **Step 1: Replace the RULES DISCLOSURE block**

Replace lines ~241–270 with:

```css
.rules{
  border:1px solid var(--line);border-radius:var(--r-md);
  overflow:hidden;background:var(--surface);
}
.rules summary{
  list-style:none;cursor:pointer;
  padding:14px var(--sp-3);
  display:flex;justify-content:space-between;align-items:center;
  font-family:var(--ui);font-weight:600;font-size:.95rem;
  letter-spacing:0;color:var(--text);min-height:48px;
}
.rules summary::-webkit-details-marker{display:none}
.rules summary .chev{color:var(--accent);transition:transform .25s ease;font-style:normal}
.rules[open] summary .chev{transform:rotate(180deg)}
.rules-body{
  padding:0 var(--sp-3) var(--sp-3);
  font-family:var(--ui);color:var(--text-muted);
  font-size:.92rem;line-height:1.65;
}
.rules-body ol{padding-left:1.2em;display:flex;flex-direction:column;gap:6px}
.rules-body b{color:var(--text);font-weight:600}
```

- [ ] **Step 2: Replace the PLAYER SETUP block**

Replace lines ~275–318 with:

```css
.players{display:flex;flex-direction:column;gap:var(--sp-2)}
.player-row{display:flex;align-items:center;gap:var(--sp-2)}
.player-seat{
  width:36px;height:36px;flex:0 0 36px;border-radius:50%;
  display:flex;align-items:center;justify-content:center;
  font-family:var(--display);font-weight:600;font-size:1rem;
  color:var(--accent);
  border:1px solid var(--line-strong);background:var(--surface);
}
.player-input{
  flex:1;min-height:var(--tap);
  background:var(--surface-hi);border:1px solid var(--line);
  border-radius:var(--r-md);padding:0 var(--sp-3);
  color:var(--text);font-family:var(--ui);font-size:1rem;
  transition:border-color .18s ease, background .18s ease;
}
.player-input::placeholder{color:var(--text-faint)}
.player-input:focus{outline:none;border-color:var(--accent);background:#2C2730}
.player-remove{
  width:var(--tap);height:var(--tap);flex:0 0 var(--tap);
  border-radius:var(--r-md);color:var(--text-faint);
  font-size:1.4rem;line-height:1;
  display:flex;align-items:center;justify-content:center;
}
.player-remove:active{color:var(--red)}
.add-player{align-self:flex-start;margin-top:var(--sp-1)}
.form-error{
  font-family:var(--ui);color:var(--red);
  font-size:.9rem;min-height:1.2em;font-weight:500;
}
```

- [ ] **Step 3: Verify**

Open Setup. Expect: seat badges are amber-on-dark circles; inputs are flat dark fields that gain an amber border on focus (no brass); rules panel is a calm dark card with an amber chevron that rotates when opened. Add/remove a player and confirm error text is a clean red.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Redesign setup screen: calm rules disclosure and player inputs"
```

---

## Task 4: Game hub chrome — role card, miss meter, deck meter, board slots, mini-cards

**Files:**
- Modify: `index.html` `<style>` — DEALER/GUESSER ROLE CARD (~323–355), MISS METER (~360–373), DECK METER (~378–420), BOARD (~425–496), DECK STACK (~955–968)
- Modify: `index.html` (HTML) — remove the `.deck-stack` markup in the game screen (~1197–1201)
- Modify: `index.html` `<script>` — add `suitSym` helper; use it in `renderMiniCard` (~1898–1904)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces:
  - JS helper `suitSym(suitIndex)` → returns the suit symbol string with a U+FE0E text-presentation selector appended (forces monochrome text rendering on iOS, so CSS controls color). Signature: `suitSym(s) -> string`. **Consumed by Task 7** (`renderFullCard`).
  - Restyled `.role-card/.role-label/.role-name/.role-sub`, `.miss-meter/.miss-dot`, `.deck-meter/.deck-count/.deck-bar/.deck-bar-fill`, `.board/.value-slot/.mini-card/.slot-count`.

- [ ] **Step 1: Add the `suitSym` helper**

In `index.html` `<script>`, immediately after the `suitOf` function (~1886), add:

```javascript
  // Force text (monochrome) presentation of suit glyphs so iOS doesn't render
  // hearts/diamonds as color emoji; CSS then controls the color.
  function suitSym(s){ return SUITS[s].sym + '︎'; }
```

- [ ] **Step 2: Use `suitSym` in `renderMiniCard`**

In `renderMiniCard` (~1898–1904), change the `innerHTML` line so the suit uses the helper:

```javascript
    el.innerHTML = '<span class="mc-v">'+LABELS[card.value]+'</span><span class="mc-s">'+suitSym(card.suit)+'</span>';
```

- [ ] **Step 3: Remove the fake deck-stack markup from the game screen**

In the game `<section>`, delete the `.deck-stack` div and its three `.ds-tile` children (~1197–1201) so the `.deck-meter` contains only the label and count. The resulting block:

```html
    <div class="deck-meter">
      <span class="label">Cards left</span>
      <span class="deck-count"><span id="deckCount">52</span> <small>/ 52</small></span>
    </div>
    <div class="deck-bar"><div class="deck-bar-fill" id="deckBarFill"></div></div>
```

- [ ] **Step 4: Delete the DECK STACK style block**

Remove the entire `DECK STACK (mini card-back tiles)` CSS block (~955–968, the `.deck-stack` rules). It is now unused.

- [ ] **Step 5: Replace ROLE CARD + MISS METER**

Replace lines ~323–373 with:

```css
.role-card{
  border:1px solid var(--line);border-radius:var(--r-lg);
  background:var(--surface);padding:var(--sp-4);
  display:flex;flex-direction:column;gap:var(--sp-1);
  box-shadow:var(--sh-2);text-align:center;
}
.role-label{
  font-family:var(--ui);font-size:.7rem;letter-spacing:.14em;
  text-transform:uppercase;color:var(--accent);font-weight:600;
}
.role-name{
  font-family:var(--display);font-weight:600;
  font-size:clamp(1.85rem,8.5vw,2.5rem);color:var(--text);
  line-height:1.04;overflow-wrap:break-word;letter-spacing:-.01em;
}
.role-sub{font-family:var(--ui);color:var(--text-muted);font-size:.92rem}
.role-sub b{color:var(--text);font-weight:600}

.miss-meter{display:flex;align-items:center;justify-content:center;gap:8px;margin-top:6px;min-height:16px}
.miss-dot{
  width:10px;height:10px;border-radius:50%;
  border:1px solid var(--line-strong);background:transparent;
  transition:background .2s ease, border-color .2s ease;
}
.miss-dot.on{background:var(--accent);border-color:var(--accent)}
```

- [ ] **Step 6: Replace DECK METER**

Replace lines ~378–420 with:

```css
.deck-meter{
  display:flex;align-items:center;justify-content:space-between;gap:var(--sp-3);
  padding:14px var(--sp-3);
  border:1px solid var(--line);border-radius:var(--r-md);background:var(--surface);
}
.deck-meter .label{
  font-family:var(--ui);font-size:.72rem;letter-spacing:.12em;
  text-transform:uppercase;color:var(--text-muted);font-weight:600;
}
.deck-count{font-family:var(--data);font-variant-numeric:tabular-nums;font-size:1.4rem;font-weight:700;color:var(--text)}
.deck-count small{font-size:.85rem;color:var(--text-muted);font-weight:400}
.deck-bar{
  height:6px;border-radius:999px;background:rgba(0,0,0,.35);
  overflow:hidden;margin-top:calc(-1 * var(--sp-2));
}
.deck-bar-fill{height:100%;border-radius:999px;background:var(--accent);transition:width .3s ease}
```

- [ ] **Step 7: Replace the BOARD block**

Replace lines ~425–496 with:

```css
.board-wrap{display:flex;flex-direction:column;gap:var(--sp-2)}
.board-wrap .label{
  font-family:var(--ui);font-size:.72rem;letter-spacing:.12em;
  text-transform:uppercase;color:var(--text-muted);font-weight:600;
}
.board{display:grid;grid-template-columns:repeat(7,1fr);gap:5px;padding:4px 0}
.value-slot{
  position:relative;aspect-ratio:5/7;border-radius:8px;
  border:1px solid var(--line);background:var(--surface);
  display:flex;align-items:center;justify-content:center;
}
.value-slot .slot-ghost{
  font-family:var(--display);font-weight:600;font-size:.95rem;
  color:var(--text-faint);
}
.value-slot.filled{border-color:transparent;background:transparent}
.value-slot.filled .slot-ghost{display:none}

.mini-card{
  position:absolute;inset:0;border-radius:8px;
  background:var(--paper);color:var(--paper-ink);
  box-shadow:0 2px 6px rgba(0,0,0,.45);
  border:1px solid rgba(33,27,20,.18);
  font-family:var(--display);font-weight:600;
}
.mini-card .mc-v{position:absolute;top:3px;left:5px;font-size:.82rem;line-height:1}
.mini-card .mc-s{position:absolute;bottom:3px;right:5px;font-size:.9rem;line-height:1}
.mini-card.red{color:var(--suit-red)}

.slot-count{
  position:absolute;bottom:-5px;right:-5px;z-index:5;
  min-width:16px;height:16px;padding:0 3px;border-radius:9px;
  background:var(--accent);color:var(--accent-ink);
  font-family:var(--data);font-variant-numeric:tabular-nums;font-weight:700;font-size:.6rem;
  display:flex;align-items:center;justify-content:center;
  box-shadow:0 1px 3px rgba(0,0,0,.55);
}
```

- [ ] **Step 8: Verify**

Play into the game screen and deal a few cards, then return to the hub (Next round → Ready). Expect: dealer name in Fraunces; deck row is a clean label + tabular count with a slim solid-amber progress bar (no fake stacked tiles); the board is a 7-wide grid of quiet slots; played cards are clean ivory mini-cards with amber "×N" count badges. On iOS/Safari, confirm hearts/diamonds render **red monochrome**, not emoji.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Redesign game hub: clean role card, deck meter, board, ivory mini-cards; iOS suit fix"
```

---

## Task 5: The tab (leaderboard)

**Files:**
- Modify: `index.html` `<style>` — THE TAB LIST (~904–950), EDITABLE TAB NAME (~1110–1130)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled `.tab-list/.tab-row/.tab-row--lead/.tab-rank/.tab-name/.tab-count/.lead-mark`, `.tab-name-editable/.tab-name-inline-input`. DOM/classes unchanged (`buildTabRows` untouched).

- [ ] **Step 1: Replace THE TAB LIST block**

Replace lines ~904–950 with:

```css
.tab-list{list-style:none;display:flex;flex-direction:column;gap:2px;padding:2px 0}
.tab-row{
  display:flex;align-items:center;gap:var(--sp-2);
  padding:12px var(--sp-3);border-radius:var(--r-sm);
  background:transparent;
}
.tab-row--lead{background:rgba(231,178,76,.08)}
.tab-rank{
  font-family:var(--data);font-variant-numeric:tabular-nums;font-weight:700;
  color:var(--text-faint);font-size:.9rem;width:1.4em;text-align:center;flex:0 0 auto;
}
.tab-row--lead .tab-rank{color:var(--accent)}
.tab-name{
  flex:1;font-family:var(--ui);font-weight:500;color:var(--text);
  overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
}
.tab-count{
  font-family:var(--data);font-variant-numeric:tabular-nums;font-weight:700;
  font-size:1.15rem;color:var(--text);
}
.tab-row--lead .tab-count{color:var(--accent)}
.lead-mark{font-size:.8em;margin-right:.15em;vertical-align:.05em;color:var(--accent)}
```

- [ ] **Step 2: Replace EDITABLE TAB NAME block**

Replace lines ~1110–1130 with:

```css
.tab-name-editable{cursor:pointer;border-bottom:1px dashed var(--line-strong);padding-bottom:1px}
.tab-name-editable:hover{border-bottom-color:var(--accent)}
.tab-name-inline-input{
  flex:1;background:var(--surface-hi);border:1px solid var(--accent);
  border-radius:var(--r-sm);color:var(--text);
  font-size:.95rem;font-family:var(--ui);font-weight:500;
  padding:2px 8px;min-height:0;outline:none;max-width:100%;
}
```

- [ ] **Step 3: Verify**

On the game hub with some drinks tallied, expect a clean leaderboard: leader row tinted amber with an amber count and star, others quiet. Tap a name to rename inline; the input shows an amber border. Confirm numbers are tabular (aligned).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Redesign the tab as a clean amber-led leaderboard"
```

---

## Task 6: Guess screens — value grid selector + higher/lower hint

**Files:**
- Modify: `index.html` `<style>` — VALUE SELECTOR GRID (~501–539), HINT (~544–578)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled `.selector/.value-grid/.value-cell/.vc-rank/.value-cell.selected/.value-cell.out`, `.hint/.hint-sub/.hint-word/.hint.higher/.hint.lower/.arrow/.prev-guess`. DOM/classes unchanged (`buildValueGrid`, `renderSecondGuess` untouched).

- [ ] **Step 1: Replace the VALUE SELECTOR GRID block**

Replace lines ~501–539 with:

```css
.selector{display:flex;flex-direction:column;gap:var(--sp-3)}
.value-grid{display:grid;grid-template-columns:repeat(5,1fr);gap:8px}
.value-cell{
  min-height:60px;border-radius:var(--r-md);
  background:var(--surface);border:1px solid var(--line);
  color:var(--text);
  display:flex;flex-direction:column;align-items:center;justify-content:center;gap:4px;
  transition:transform .08s ease, background .15s ease, border-color .15s ease;
  -webkit-tap-highlight-color:transparent;
}
.value-cell .vc-rank{font-family:var(--display);font-weight:600;font-size:1.25rem;line-height:1}
.value-cell:active{transform:scale(.94)}
.value-cell.selected{
  background:var(--accent);color:var(--accent-ink);border-color:var(--accent);
  box-shadow:var(--sh-accent);
}
.value-cell.out{opacity:.3;cursor:default;border-style:dashed}
.value-cell.out .vc-rank{
  text-decoration:line-through;text-decoration-color:var(--red);color:var(--text-faint);
}
```

- [ ] **Step 2: Replace the HINT block**

Replace lines ~544–578 with:

```css
.hint{text-align:center;border-radius:var(--r-lg);padding:var(--sp-4) var(--sp-3);background:var(--surface);border:1px solid var(--line)}
.hint .hint-sub{
  font-family:var(--ui);color:var(--text-muted);font-size:.72rem;
  letter-spacing:.14em;text-transform:uppercase;font-weight:600;
}
.hint .hint-word{
  font-family:var(--display);font-weight:600;
  font-size:clamp(2.6rem,13vw,3.6rem);line-height:1;letter-spacing:-.01em;
  margin-top:6px;display:inline-flex;align-items:center;gap:.2em;
}
.hint.higher .hint-word{color:var(--accent)}
.hint.lower  .hint-word{color:var(--red)}
.hint .arrow{font-size:.7em}
.hint .prev-guess{font-family:var(--ui);margin-top:10px;color:var(--text-muted);font-size:.92rem}
.hint .prev-guess b{color:var(--text)}
```

- [ ] **Step 3: Verify**

Reach `firstGuess`: the A–K grid is a clean set of dark cells; tapping one turns it solid amber with dark text. Guess wrong to reach `secondGuess`: the hint reads "Higher" in amber (or "Lower" in red) in big Fraunces, and ruled-out / spent cells are dimmed with a red strike-through and can't be tapped.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Redesign guess screens: amber-select value grid and bold higher/lower hint"
```

---

## Task 7: The hero card — face, real pip layouts, back, flip, deal overlay

**Files:**
- Modify: `index.html` `<style>` — FULL CARD + FLIP (~583–660), DEAL OVERLAY (~973–1003)
- Modify: `index.html` (HTML) — card-back emblem (~1270–1272) and deal-card emblem (~1313–1315) glyphs
- Modify: `index.html` `<script>` — add `PIPS` constant + `pipMarkup` helper; rewrite `renderFullCard` (~1888–1896)

**Interfaces:**
- Consumes: Task 1 tokens; `suitSym(s)` from Task 4.
- Produces: a card face with corner rank+suit, real pip arrangements for 2–10, and centerpiece glyph for A/J/Q/K, on ivory paper. `renderFullCard(container, card)` keeps the same signature.

- [ ] **Step 1: Replace the FULL CARD + FLIP CSS**

Replace lines ~583–660 with:

```css
.card-stage{position:relative;display:flex;justify-content:center;padding:var(--sp-2) 0;perspective:1400px}
.card-flip{
  position:relative;width:210px;height:300px;
  transform-style:preserve-3d;-webkit-transform-style:preserve-3d;
  transition:transform .72s cubic-bezier(.2,.8,.25,1);
}
.card-flip.down{transform:rotateY(180deg)}
.card-face{
  position:absolute;inset:0;border-radius:var(--r-card);
  backface-visibility:hidden;-webkit-backface-visibility:hidden;
  overflow:hidden;box-shadow:var(--sh-card);
}
.card-face.front{
  transform:rotateY(0deg);
  background:var(--paper);color:var(--paper-ink);
  border:1px solid rgba(33,27,20,.18);
}
.card-face.back{
  transform:rotateY(180deg);
  background:var(--surface);border:1px solid var(--line-strong);
}
.card-back-emblem{
  position:absolute;inset:14px;border:1px solid rgba(231,178,76,.5);
  border-radius:10px;box-shadow:inset 0 0 0 3px rgba(231,178,76,.14);
  display:flex;align-items:center;justify-content:center;
}
.card-back-emblem span{font-size:3.2rem;color:var(--accent);opacity:.85}

.card-corner{
  position:absolute;display:flex;flex-direction:column;align-items:center;
  line-height:1;font-family:var(--display);font-weight:600;
}
.card-corner .cc-v{font-size:1.55rem}
.card-corner .cc-s{font-size:1.15rem;margin-top:2px}
.card-corner.tl{top:12px;left:14px}
.card-corner.br{bottom:12px;right:14px;transform:rotate(180deg)}
.card-face.red{color:var(--suit-red)}

/* Pip field for number cards 2-10 */
.card-pips{position:absolute;inset:44px 40px}
.card-pips .pip{
  position:absolute;transform:translate(-50%,-50%);
  font-size:2rem;line-height:1;
}
.card-pips .pip.flip{transform:translate(-50%,-50%) rotate(180deg)}

/* Centerpiece for A / J / Q / K */
.card-center{position:absolute;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:6px}
.card-center .cc-big{font-family:var(--display);font-weight:600;font-size:4.4rem;line-height:.9}
.card-center .cc-suit{font-size:2.6rem;line-height:1}
.card-center.ace .cc-suit{font-size:5rem}
.card-center.ace .cc-big{display:none}
```

- [ ] **Step 2: Add `PIPS` and `pipMarkup` before `renderFullCard`**

In `index.html` `<script>`, just before `renderFullCard` (~1888), add:

```javascript
  // Pip centers as [xPercent, yPercent] within the .card-pips field.
  // Pips below the vertical midline are rotated 180deg (rendered upside-down),
  // matching real playing cards. L/C/R columns, T..B rows.
  var _L=22,_C=50,_R=78,_T=8,_TM=27,_UM=33,_M=50,_LM=67,_BM=73,_B=92;
  var PIPS = {
    2:[[_C,_T],[_C,_B]],
    3:[[_C,_T],[_C,_M],[_C,_B]],
    4:[[_L,_T],[_R,_T],[_L,_B],[_R,_B]],
    5:[[_L,_T],[_R,_T],[_C,_M],[_L,_B],[_R,_B]],
    6:[[_L,_T],[_R,_T],[_L,_M],[_R,_M],[_L,_B],[_R,_B]],
    7:[[_L,_T],[_R,_T],[_C,_TM],[_L,_M],[_R,_M],[_L,_B],[_R,_B]],
    8:[[_L,_T],[_R,_T],[_C,_TM],[_L,_M],[_R,_M],[_C,_BM],[_L,_B],[_R,_B]],
    9:[[_L,_T],[_R,_T],[_L,_UM],[_R,_UM],[_C,_M],[_L,_LM],[_R,_LM],[_L,_B],[_R,_B]],
    10:[[_L,_T],[_R,_T],[_C,_TM],[_L,_UM],[_R,_UM],[_L,_LM],[_R,_LM],[_C,_BM],[_L,_B],[_R,_B]]
  };
  function pipMarkup(card){
    var pts = PIPS[card.value];
    if(!pts) return '';
    var sym = suitSym(card.suit), html = '';
    for(var i=0;i<pts.length;i++){
      var x=pts[i][0], y=pts[i][1];
      var flip = y > 50 ? ' flip' : '';
      html += '<span class="pip'+flip+'" style="left:'+x+'%;top:'+y+'%">'+sym+'</span>';
    }
    return '<div class="card-pips">'+html+'</div>';
  }
```

- [ ] **Step 3: Rewrite `renderFullCard`**

Replace the `renderFullCard` function (~1888–1896) with:

```javascript
  function renderFullCard(container, card){
    var s = suitOf(card);
    var label = LABELS[card.value];
    var sym = suitSym(card.suit);
    container.className = 'card-face front' + (s.red ? ' red' : '');
    var corners =
      '<div class="card-corner tl"><span class="cc-v">'+label+'</span><span class="cc-s">'+sym+'</span></div>' +
      '<div class="card-corner br"><span class="cc-v">'+label+'</span><span class="cc-s">'+sym+'</span></div>';
    var body;
    if(card.value >= 2 && card.value <= 10){
      body = pipMarkup(card);
    }else{
      var isAce = card.value === 1;
      body = '<div class="card-center'+(isAce?' ace':'')+'">' +
             '<span class="cc-big">'+label+'</span>' +
             '<span class="cc-suit">'+sym+'</span></div>';
    }
    container.innerHTML = corners + body;
  }
```

- [ ] **Step 4: Refresh the card-back and deal-card emblem glyphs**

The `<span>♠</span>` inside `.card-back-emblem` (~1271) and `.dc-emblem` (~1314) already use a spade; leave the glyph but confirm they inherit the new amber styling. No markup change needed unless the glyph shows as emoji — if it does on-device, change both to `<span>♠&#xFE0E;</span>`.

- [ ] **Step 5: Replace the DEAL OVERLAY CSS**

Replace lines ~973–1003 with:

```css
.deal-overlay{
  position:fixed;inset:0;z-index:100;
  display:flex;align-items:center;justify-content:center;
  pointer-events:none;visibility:hidden;
}
.deal-overlay.active{visibility:visible}
.deal-card{
  position:relative;width:210px;height:300px;border-radius:var(--r-card);
  background:var(--surface);border:1px solid var(--line-strong);
  box-shadow:var(--sh-card);
  display:flex;align-items:center;justify-content:center;
}
.deal-card .dc-emblem{
  position:absolute;inset:14px;border:1px solid rgba(231,178,76,.5);
  border-radius:10px;display:flex;align-items:center;justify-content:center;
}
.deal-card .dc-emblem span{font-size:3.2rem;color:var(--accent);opacity:.85}
.deal-card.animating{animation:deal-fly .7s cubic-bezier(.15,1.2,.4,1) both}

@keyframes deal-fly{
  from{transform:translate(-130px,180px) scale(.32) rotate(-12deg);opacity:0}
  to{transform:translate(0,0) scale(1) rotate(0deg);opacity:1}
}
```

- [ ] **Step 6: Verify**

Deal and reach `reveal` several times to see different ranks. Expect: an ivory card with corner rank+suit (top-left and rotated bottom-right); number cards 2–10 show the **correct count of pips** in the classic arrangement with the lower half upside-down; A shows one big centered suit; J/Q/K show a big Fraunces letter above the suit. Red suits are `--suit-red`, black are near-black — never emoji. The card back is a dark panel with a thin amber-framed spade. The deal fly-in still plays.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Craft the hero card: ivory face, real pip layouts, amber card back"
```

---

## Task 8: Reveal banner + celebration (subtler glint, replace wax stamp)

**Files:**
- Modify: `index.html` `<style>` — RESULT BANNER (~665–719), CELEBRATION (~1035–1092)
- Modify: `index.html` `<script>` — `spawnWaxStamp` (~1789–1795) and its CSS class

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled `.banner/.banner.win/.banner.penalty/.b-eyebrow/.b-main/.b-sub/.drink-num`; a subtler `.card-glint`; the wax stamp replaced by a clean embossed penalty seal (`.penalty-seal`). `renderReveal` continues to call `spawnWaxStamp()` (renamed internally is fine, but keep the call site working).

- [ ] **Step 1: Replace the RESULT BANNER block**

Replace lines ~665–719 with:

```css
.banner{
  text-align:center;border-radius:var(--r-lg);
  padding:var(--sp-4) var(--sp-3);z-index:var(--z-banner);
  background:var(--surface);border:1px solid var(--line);
  animation:banner-pop .42s cubic-bezier(.2,.8,.3,1) both;
}
.banner .b-eyebrow{font-family:var(--ui);font-size:.7rem;letter-spacing:.14em;text-transform:uppercase;font-weight:600}
.banner .b-main{
  font-family:var(--display);font-weight:600;
  font-size:clamp(1.8rem,8vw,2.4rem);line-height:1.05;margin-top:6px;letter-spacing:-.01em;
}
.banner .b-sub{font-family:var(--ui);font-size:.95rem;margin-top:8px;color:var(--text-muted)}
.banner.win{border-color:rgba(231,178,76,.5)}
.banner.win .b-eyebrow{color:var(--accent)}
.banner.win .b-main{color:var(--text)}
.banner.penalty{border-color:rgba(229,97,94,.5)}
.banner.penalty .b-eyebrow{color:var(--red)}
.banner.penalty .b-main{color:var(--text)}
.drink-num{font-family:var(--data);font-variant-numeric:tabular-nums;font-size:1.1em;color:var(--accent);font-weight:700}
.banner.penalty .drink-num{color:var(--red)}

@keyframes banner-pop{
  from{opacity:0;transform:translateY(8px)}
  to{opacity:1;transform:none}
}
```

- [ ] **Step 2: Replace the CELEBRATION block (subtler glint + embossed seal)**

Replace lines ~1035–1092 with:

```css
.card-glint{
  position:absolute;top:50%;left:50%;
  width:210px;height:300px;transform:translate(-50%,-50%);
  border-radius:var(--r-card);overflow:hidden;pointer-events:none;z-index:6;
}
.card-glint::before{
  content:'';position:absolute;top:-20%;left:0;width:100%;height:140%;
  background:linear-gradient(115deg,
    transparent 40%, rgba(255,255,255,.35) 50%, rgba(231,178,76,.25) 55%, transparent 66%);
  transform:translateX(-120%);
  animation:glint-sweep .7s ease-out forwards;
}
@keyframes glint-sweep{from{transform:translateX(-120%)}to{transform:translateX(120%)}}

/* LOSS: a clean embossed red "SCREWED" seal, not a wax blob. */
.penalty-seal{
  position:fixed;left:50%;top:38%;
  transform:translate(-50%,-50%) rotate(-7deg);
  padding:8px 18px;border:2px solid var(--red);border-radius:8px;
  color:var(--red);background:rgba(229,97,94,.08);
  font-family:var(--display);font-weight:600;font-size:1.5rem;
  letter-spacing:.04em;text-transform:uppercase;
  pointer-events:none;z-index:55;
  animation:seal-press .45s cubic-bezier(.2,.9,.35,1) both;
}
@keyframes seal-press{
  from{transform:translate(-50%,-50%) rotate(-7deg) scale(1.5);opacity:0}
  to{transform:translate(-50%,-50%) rotate(-7deg) scale(1);opacity:1}
}
```

- [ ] **Step 3: Update `spawnWaxStamp` to use the new seal class**

Replace the `spawnWaxStamp` function (~1789–1795) with:

```javascript
  // LOSS: a clean embossed "SCREWED" seal presses onto the card.
  function spawnWaxStamp(){
    var stamp = document.createElement('div');
    stamp.className = 'penalty-seal';
    stamp.textContent = 'SCREWED';
    document.getElementById('app').appendChild(stamp);
    setTimeout(function(){ stamp.parentNode && stamp.parentNode.removeChild(stamp); }, 1500);
  }
```

Also update the cleanup selector in `clearCelebration` (~1774): change `'.card-glint, .wax-stamp'` to `'.card-glint, .penalty-seal'`. And in the REDUCED MOTION block (~1143) change `.wax-stamp{animation:none}` to `.penalty-seal{animation:none}`.

- [ ] **Step 4: Verify**

Win a round (correct first call): banner reads in amber with tabular drink numbers, and a subtle light sheen sweeps once across the card. Lose a round (miss both calls): banner outlined in red, and a crisp embossed "SCREWED" seal presses on — no lumpy wax blob. Toggle OS "Reduce Motion" and confirm both appear instantly without animation.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Refine reveal: clean banners, subtler glint, embossed penalty seal"
```

---

## Task 9: Pass-the-phone (rotate) screen

**Files:**
- Modify: `index.html` `<style>` — ROTATE / PASS (~724–774)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled `.pass/.pass-chip/.pass-label/.pass-name/.pass-note`. DOM/classes unchanged.

- [ ] **Step 1: Replace the ROTATE / PASS block**

Replace lines ~724–774 with:

```css
.pass{text-align:center;display:flex;flex-direction:column;gap:var(--sp-3);align-items:center;justify-content:center;flex:1}
.pass-chip{
  width:56px;height:78px;border-radius:10px;
  border:1px solid var(--line-strong);background:var(--surface);
  box-shadow:var(--sh-2);display:flex;align-items:center;justify-content:center;
}
.pass-chip::after{content:'♠\FE0E';font-size:1.6rem;color:var(--accent);opacity:.85}
.pass .pass-label{
  font-family:var(--ui);color:var(--text-muted);letter-spacing:.14em;
  text-transform:uppercase;font-size:.8rem;font-weight:600;
}
.pass .pass-name{
  font-family:var(--display);font-weight:600;
  font-size:clamp(2rem,10vw,2.9rem);color:var(--accent);
  line-height:1.04;overflow-wrap:break-word;letter-spacing:-.01em;
}
.pass .pass-note{font-family:var(--ui);color:var(--text-muted);font-size:.92rem;max-width:30ch}
```

- [ ] **Step 2: Verify**

After a round, the pass screen shows a calm centered card-back chip, an uppercase amber label, the next person's name large in amber Fraunces, and a muted note. No brass, no felt.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Redesign pass-the-phone screen"
```

---

## Task 10: Game-over receipt refinement

**Files:**
- Modify: `index.html` `<style>` — GAME-OVER RECEIPT (~779–897)

**Interfaces:**
- Consumes: Task 1 tokens.
- Produces: restyled receipt (`.receipt` and its children) on the same ivory paper as the cards. DOM built by `renderGameOver` is unchanged.

- [ ] **Step 1: Replace the receipt paper + header/rows CSS**

In the GAME-OVER RECEIPT block (~779–897), replace the `.receipt` background (the multi-layer grain gradient, ~790–795) with the flat paper, and retune ink colors. Replace the `.receipt` rule's `background:` declaration with:

```css
  background:var(--paper);
```
Then within the same block replace these child rules with:

```css
.rcpt-star{font-family:var(--data);font-variant-numeric:tabular-nums;font-size:.68rem;letter-spacing:.18em;color:var(--suit-red);font-weight:700}
.rcpt-title{font-family:var(--display);font-weight:600;font-size:1.6rem;letter-spacing:0;color:var(--paper-ink)}
.rcpt-meta{font-family:var(--data);font-variant-numeric:tabular-nums;font-size:.68rem;color:rgba(33,27,20,.65);letter-spacing:.02em}
.rcpt-rule{border:none;border-top:1px dashed rgba(33,27,20,.4);margin:var(--sp-1) 0}
.rcpt-row{display:flex;align-items:baseline;gap:5px;font-family:var(--data);font-variant-numeric:tabular-nums;font-size:.9rem;color:var(--paper-ink)}
.rcpt-row .nm{white-space:nowrap;overflow:hidden;text-overflow:ellipsis;max-width:60%}
.rcpt-row .ldr{flex:1;border-bottom:1px dotted rgba(33,27,20,.35);transform:translateY(-3px)}
.rcpt-row .amt{white-space:nowrap;font-weight:700}
.rcpt-row.total{font-weight:700;letter-spacing:.01em}
.rcpt-awards{display:flex;flex-direction:column;gap:4px;margin-top:var(--sp-1);font-family:var(--data);font-variant-numeric:tabular-nums;font-size:.8rem;color:var(--paper-ink)}
.rcpt-award{display:flex;gap:6px;align-items:baseline}
.rcpt-closer{
  text-align:center;font-family:var(--data);font-variant-numeric:tabular-nums;
  font-size:.68rem;letter-spacing:.14em;color:var(--suit-red);font-weight:700;
  margin-top:var(--sp-2);display:inline-block;width:100%;
  border:1px double rgba(194,58,50,.6);padding:3px 8px;transform:rotate(-1.5deg);opacity:.9;
}
```
Keep the `.receipt` perforation mask, `transform:rotate(-1deg)`, padding, and `box-shadow:var(--sh-card)` as they are.

- [ ] **Step 2: Verify (with a shortcut to reach game over fast)**

Temporarily make the deck tiny so the game ends quickly: in `buildDeck` (~1363) change `for(var v=1;v<=13;v++)` to `for(var v=1;v<=1;v++)`. Reload, start a game, and deal until `gameOver`. Expect: a clean ivory receipt (matching the cards), crisp dashed dividers, dark ink rows with aligned tabular numbers, and a red "PAID IN FULL" closer. **Revert the `buildDeck` change** before committing:

```bash
grep -n 'for(var v=1;v<=1;v++)' index.html && echo "REVERT the buildDeck loop to v<=13 before committing!"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Refine game-over receipt onto matching ivory paper"
```

---

## Task 11: Motion easing pass + full-playthrough verification

**Files:**
- Modify: `index.html` `<style>` — `screen-in` keyframe/anim (~97–104)
- Verify only: whole app

**Interfaces:**
- Consumes: everything above.
- Produces: calmer screen-transition easing; final sign-off.

- [ ] **Step 1: Calm the screen-transition easing**

In the `.screen` rule (~97), change the animation to a shorter, less springy curve:

```css
  animation:screen-in .34s cubic-bezier(.2,.7,.25,1) both;
```
And soften the `screen-in` keyframe travel (~101–104):

```css
@keyframes screen-in{
  from{opacity:0;transform:translateY(10px)}
  to{opacity:1;transform:none}
}
```
(The deal-fly, flip, banner-pop, and glint easings were already retuned in their own tasks.)

- [ ] **Step 2: Full playthrough verification**

Serve locally (`python3 -m http.server 8000`) and play a complete game on a ~390px viewport:
- Setup → add up to 8 players → Start.
- Deal, make a correct first call (amber win + sheen), and a double-miss (red penalty + embossed seal).
- Watch the deck bar drain and the tab update; rename a player on the tab.
- Trigger a dealer change (three misses in a row) and confirm the pass screen wording.
- Confirm the sticky *Deal* footer stays pinned and clears the iOS home indicator.

- [ ] **Step 3: Regression + accessibility checks**

- **Saved game:** mid-game, reload the page — the game resumes on the same screen (localStorage `screwTheDealer.v1` intact).
- **Offline:** with the SW registered, go offline and reload — app + Fraunces still render.
- **Reduced motion:** enable OS Reduce Motion — no screen/flip/deal animations, seal/banner appear instantly.
- **Contrast:** spot-check muted text on dark and ink on paper against WCAG-AA (should pass; `--text-muted` on `--bg` ≈ 7:1).
- **Stale tokens:** `grep -nE '--felt|--mahogany|--brass|--ivory|--card-red|--text-dim|\.wax-stamp|\.deck-stack|Cormorant|Courier' index.html || echo "OK: clean"`

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "Calm screen-transition motion; final redesign verification pass"
```

---

## Self-Review

**1. Spec coverage** — every spec section maps to a task:
- Organizing concept (dark chrome / warm paper / amber) → Tasks 1–11 collectively; paper objects = Tasks 7 (card) + 10 (receipt).
- Color system → Task 1 tokens, applied throughout.
- Typography (Fraunces + system, offline self-host, reduced tracking, no all-caps everywhere, tabular numerals) → Task 1 (fonts/tokens) + Task 2 (roles) + per-component tasks; `tabular-nums` in Tasks 4, 5, 8, 10.
- App-chrome components (panels, buttons, value grid, meters, leaderboard) → Tasks 2, 4, 5, 6.
- Hero card + real pips + clean back → Task 7.
- Receipt kept & elevated → Task 10.
- Gimmick cleanup (wax stamp → seal, subtler glint, remove deck-stack/brass badges) → Tasks 4 + 8.
- Per-screen layout polish (setup, game hub, guess, reveal, rotate, game over) → Tasks 3, 4, 5, 6, 7, 8, 9, 10.
- Motion refinement → Task 11 (+ per-task easing).
- Constraints: single-file (all), logic untouched (only `renderFullCard`/`renderMiniCard`/`renderGameOver`/`spawnWaxStamp` DOM markup touched — no engine/state), offline PWA (Task 1 sw.js), iOS safe-area (preserved), AA contrast (Tasks 1 + 11), theme-color/manifest/icons (Task 1).

**2. Placeholder scan** — no "TBD/TODO/handle edge cases". The one interactive value (the font URL) is resolved deterministically via `tail -1` with a verified fallback; the icon-PNG step degrades explicitly rather than leaving a blank.

**3. Type consistency** — `suitSym(s)` defined in Task 4, consumed in Task 4 (`renderMiniCard`) and Task 7 (`renderFullCard`/`pipMarkup`). `PIPS`/`pipMarkup` defined and used only in Task 7. `spawnWaxStamp` retains its name (call site in `renderReveal` unchanged) but emits `.penalty-seal`, and the `clearCelebration` selector + reduced-motion rule are updated in the same task (Task 8) — no dangling `.wax-stamp` references. Card dimensions unified at 210×300 across `.card-flip`, `.card-glint`, and `.deal-card` (Tasks 7–8). CSS var names match the Task 1 definitions.

_No gaps found._

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-07-01-professional-visual-redesign.md`.
