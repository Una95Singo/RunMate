# Pace — Technical Walkthrough

A line-by-line, "why does this work" tour of the codebase. Read top-to-bottom or jump around — each section is self-contained.

---

## 1. What this is, and what it isn't

`index.html` is a **single-file web app**. One file contains:

- The page structure (HTML)
- The visual design (CSS, embedded in `<style>`)
- The behaviour (JavaScript, embedded in `<script>`)

There is **no build step**, no `npm install`, no bundler, no framework, no server. A browser opens the file and it just works.

That choice matters. Every other architectural decision in the project flows from it.

### Why single-file?

| Trade-off | Single-file (us) | Typical React/Vite app |
|---|---|---|
| Time to first paint | Instant — one HTTP request | Several JS chunks, parse, hydrate |
| Build complexity | Zero | Bundler, transpiler, dev server |
| Deployment | Drop one file on any static host | CI builds, dist folder, cache invalidation |
| Suits | Tools, calculators, prototypes, personal sites | Apps with routing, state machines, many components |

For a pace calculator, single-file is the right call: the math is small, the UI is one screen, and the user wants to bookmark it and have it load instantly. We don't need a framework's machinery.

### What we give up

- **Component reuse across pages.** There's only one page.
- **Type safety.** No TypeScript. We compensate with small, readable functions.
- **Build-time optimisations.** The file is already small (~25 KB), so it doesn't matter.

When this app grows past a few thousand lines, or needs persistence, routing, or background sync, that's the moment to introduce a framework. Not before.

---

## 2. The three layers, in order

The file is organised top-to-bottom in three stacked layers:

```
<!doctype html>          ← document type
<html><head>             ← metadata, icons, title
  <meta ...>
  <style>...</style>     ← LAYER 1: design (CSS)
</head>
<body>
  <main>...</main>       ← LAYER 2: structure (HTML)
  <script>...</script>   ← LAYER 3: behaviour (JS)
</body>
</html>
```

Why scripts at the end of `<body>`? Because by the time the browser reaches the `<script>` tag, every element it queries (`getElementById("min")`, etc.) already exists in the DOM. No need for `DOMContentLoaded` listeners. Old trick, still good.

---

## 3. The `<head>`: metadata that earns its keep

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="theme-color" content="#0a0b0e" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-title" content="Pace" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
```

Each line is doing real work:

- **`viewport`** tells mobile browsers "render at the device's actual width, don't pretend to be a desktop." Without it, the page would render at 980px and shrink to fit, making text microscopic. `viewport-fit=cover` lets the page extend behind the iPhone notch (we then add safe-area padding in CSS — see §4).
- **`theme-color`** colours the browser's address bar (and the iOS status bar in standalone mode) to match the page background. Tiny touch, big polish improvement.
- **`apple-mobile-web-app-capable`** + the two siblings tell iOS: "when added to the home screen, launch in standalone mode (no Safari chrome) with a black status bar and the title 'Pace'." This is what makes a bookmarked web page feel like an app.

### The inline SVG icon

```html
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml;utf8,<svg ...></svg>" />
<link rel="apple-touch-icon" href="data:image/svg+xml;utf8,<svg ...></svg>" />
```

Two interesting things here:

1. **Data URI.** Instead of linking to a separate `.png` file, we encode the icon directly in the HTML using `data:image/svg+xml;utf8,...`. This keeps the project single-file and saves an HTTP request. The icon is a stopwatch drawn with two SVG `<path>` elements — small enough to inline.
2. **SVG icon.** Modern browsers (and iOS 18+) accept SVG for both favicons and home-screen icons. SVG scales perfectly to any size, so we don't ship 14 different PNGs.

Note we URL-encode `#` as `%23` inside the data URI — `#` would otherwise be interpreted as a fragment marker.

---

## 4. The CSS layer: tokens, layout, responsive

### 4.1 Design tokens (CSS custom properties)

```css
:root {
  --bg: #0a0b0e;
  --panel: #15181f;
  --panel-2: #1c2129;
  --line: #262c36;
  --text: #f4f6fa;
  --muted: #7e8696;
  --accent: #ff5a1f;
  --accent-soft: rgba(255, 90, 31, 0.12);
  --accent-line: rgba(255, 90, 31, 0.35);
  --radius: 14px;
  --radius-sm: 10px;
}
```

These are **design tokens**: named values that the rest of the CSS references via `var(--accent)`. Three reasons this is powerful:

1. **One place to change.** Want to try a different accent colour? Edit one line. Every button, highlight, and header bar updates everywhere.
2. **Self-documenting.** `var(--muted)` reads better than `#7e8696`. Anyone scanning the CSS understands intent.
3. **Themeable.** If you later add a light mode, you redefine the tokens inside `@media (prefers-color-scheme: light)` and nothing else changes.

This is the CSS-native equivalent of what design systems like Tailwind or Material UI provide — without the framework.

### 4.2 Mobile-first layout

```css
body {
  padding: env(safe-area-inset-top) 14px env(safe-area-inset-bottom);
}
.wrap { max-width: 560px; margin: 0 auto; padding: 18px 0 36px; }
```

Two ideas working together:

- **`env(safe-area-inset-*)`** are values the browser exposes for "how much physical space is hidden by the notch / home indicator." We pad the body so content never sits underneath those areas.
- **`max-width: 560px; margin: 0 auto;`** caps the readable width. On a phone, this is wider than the screen and doesn't activate. On a tablet or desktop, it centres a phone-width column instead of stretching numbers across an entire monitor — which would be hostile to read.

This is what "mobile-first" means in practice: write the styles for the small screen as the default, then add media queries to *enhance* for larger screens, not the other way around.

```css
@media (min-width: 480px) {
  h1 { font-size: 22px; }
  .summary .big { font-size: 52px; }
}
```

Above 480px we bump a couple of font sizes. Phone-portrait users never download or apply these rules; the browser ignores them.

### 4.3 Layout primitives: Flexbox vs Grid

The codebase uses **CSS Grid** for "two/three columns of equal-ish things" and **Flexbox** for "a row where some things stretch and some don't."

Grid example — the splits row:
```css
.splits { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; }
```
"Three columns, each taking 1 fractional unit (i.e., equal). 8px gap between them." Done. No floats, no margins, no math.

Flexbox example — the time input:
```css
.time-input { display: flex; align-items: baseline; gap: 6px; }
.time-input input { flex: 1 1 0; min-width: 0; }
```
"Lay out children in a row, align them on their text baseline. The inputs grow to fill remaining space." `flex: 1 1 0` is shorthand for "grow yes, shrink yes, basis 0" — the standard "fill available space equally" recipe.

The `min-width: 0` is a subtle but critical fix: by default, a flex item won't shrink below its intrinsic content width, which on `<input type="number">` can be wider than its container, causing overflow. Setting `min-width: 0` allows the item to shrink properly.

### 4.4 The race-prediction row — Grid for asymmetric columns

```css
.race {
  display: grid;
  grid-template-columns: 80px 1fr auto;
  gap: 10px;
}
```

"First column is fixed at 80px (the distance label). Second column gets all remaining space (the finish time, allowed to expand). Third column is `auto` (sized to its content — the per-mile/per-km pace text)." Three different sizing strategies in one line.

### 4.5 Tabular numerals (for clocks)

```css
font-feature-settings: "tnum" 1;
```

Modern fonts ship with two glyph sets for digits: **proportional** (looks nicer in body text — `1` is narrower than `8`) and **tabular** (every digit is the same width, so columns of numbers line up). For pace times like `7:30 → 7:31 → 7:32`, tabular nums prevent visual jitter as the values change. Pure typography polish, costs nothing.

---

## 5. The HTML layer: semantic structure

```html
<main class="wrap">
  <header>...</header>
  <section class="card" aria-label="Goal mile pace">
    <div class="ch">Goal mile time</div>
    <div class="time-input">
      <input id="min" type="number" inputmode="numeric" pattern="[0-9]*" min="0" max="59" value="7" aria-label="minutes" />
      ...
    </div>
  </section>
  ...
</main>
```

A few things worth noticing:

### 5.1 Semantic elements over `<div>` soup

We use `<main>`, `<header>`, `<section>`, and `<button>` — not generic `<div>`s — wherever the meaning fits. Screen readers, search crawlers, and reader-mode browsers all understand these tags. Anyone hitting Tab on the keyboard navigates between `<button>`s automatically.

### 5.2 `aria-label` for unlabelled controls

```html
<input id="min" ... aria-label="minutes" />
```

The two number inputs are visually obvious (a big `mm:ss` clock), but a screen reader doesn't know "these are minutes and seconds" without help. `aria-label="minutes"` is the assistive-tech equivalent of a `<label>`, but invisible. Real `<label for="...">` elements are even better, but they take screen real estate; `aria-label` is the right trade-off here.

### 5.3 Mobile keyboard hints

```html
<input type="number" inputmode="numeric" pattern="[0-9]*" />
```

Three attributes, each doing a different job:
- `type="number"` — the spec-correct type. Activates browser-level number validation.
- `inputmode="numeric"` — tells iOS/Android "show the numeric keypad, not the full QWERTY." This is what gives mobile users the giant number keys instead of a tiny 1-2-3 row.
- `pattern="[0-9]*"` — older iOS Safari needs this in addition to `inputmode` to honour the numeric keypad.

Together they make tapping numbers on phone feel native instead of clumsy.

### 5.4 The seconds field uses `type="text"` — and that's deliberate

```html
<input id="paceMiSec" type="text" inputmode="numeric" pattern="[0-9]*" maxlength="2" value="00" />
```

Wait — text? For a number? Yes, in the planner pace inputs only. Reason: `<input type="number">` normalises its value (so `value="00"` displays as `"0"`), losing the leading zero. For a clock display, "0" looks wrong; we want "00." Switching to `type="text"` with `inputmode="numeric"` gets us the numeric keypad on mobile *and* preserves the literal string. Best of both worlds.

This kind of small, deliberate divergence from "what the spec suggests" is how you get a polished UI.

---

## 6. The JavaScript layer: simple, event-driven

The whole script is wrapped in an **IIFE** (Immediately Invoked Function Expression):

```js
(function () {
  "use strict";
  // ...all code...
})();
```

What this does and why:

- The `function () { ... }` is an anonymous function definition.
- The trailing `()` calls it immediately.
- The wrapper `(...)` exists only to make the parser accept the function-as-expression syntax.

Effect: every variable inside the IIFE is **scoped to the IIFE**, not leaked into the global window object. If two scripts on the page each use a `T` variable, they don't clash.

In modern code you'd often use an ES module (`<script type="module">`) for the same isolation. The IIFE works in every browser including ancient ones, with no extra HTTP request. For one file, it's perfect.

### 6.1 The `$` helper

```js
const $ = (id) => document.getElementById(id);
```

A six-character function. We could call `document.getElementById("min")` everywhere, but `$("min")` is clearer and matches a long jQuery convention. (Note: this is *not* jQuery — just a tiny lookalike.)

### 6.2 Pure helpers, then orchestration

The script is laid out **bottom-up**: tiny pure functions at the top (no DOM, just data → data), then bigger functions that read and write the DOM, then event wiring.

Example pure helper:

```js
function fmtTime(sec) {
  if (!Number.isFinite(sec) || sec < 0) sec = 0;
  const total = Math.round(sec);
  const m = Math.floor(total / 60);
  const s = total % 60;
  return m + ":" + String(s).padStart(2, "0");
}
```

This function:
- Takes a number, returns a string.
- Doesn't touch the DOM.
- Doesn't read or write any global state.
- Always returns the same output for the same input.

Functions like this are **easy to reason about** and **easy to test** (just call `fmtTime(450)` and assert it equals `"7:30"`). The codebase has six of them: `clampInt`, `fmtTime`, `fmtTimeLong`, `fmtSpeed`, `readPaceSec`, `writePace`. Keep your pure logic separate from your DOM-touching code and your future self will thank you.

### 6.3 The render functions

There are two main render functions: `renderGoal()` and `renderPlan()`. Plus `renderRaces(mileSec)`, called from inside `renderGoal` because race predictions depend on the goal mile time.

Pattern:

```js
function renderGoal() {
  const T = totalSeconds();           // 1. read state from the inputs

  if (T <= 0) { ...show "—" ...; return; }   // 2. handle invalid/empty

  $("goalPace").innerHTML = fmtTime(T) + ...; // 3. compute & write outputs
  const mph = 3600 / T;
  $("mph").textContent = fmtSpeed(mph);
  // ...
  renderRaces(T);                     // 4. cascade to dependent renders
}
```

This is the **render-on-input** pattern, hand-rolled. React, Vue, Svelte all do something similar but with fancier machinery (virtual DOMs, reactive signals, fine-grained reactivity). For 11 output fields, doing it by hand is fine.

### 6.4 Event wiring

```js
minEl.addEventListener("input", renderGoal);
secEl.addEventListener("input", renderGoal);
```

Every input event re-runs `renderGoal()`. The whole panel re-computes on every keystroke. Sounds expensive — but it's six floating-point ops and a few `textContent` writes. Modern browsers do this in well under a millisecond.

If we ever needed to optimise: throttle the render with `requestAnimationFrame`, or do incremental updates. We don't need to.

---

## 7. The math, demystified

### 7.1 Pace ↔ speed

A pace is "seconds per mile." A speed is "miles per hour." These are reciprocals scaled by 3600 (seconds per hour):

```js
const mph = 3600 / T;     // T = seconds per mile
const kph = mph * KM_PER_MI;  // KM_PER_MI = 1.609344
```

If your mile pace is 7:30 = 450 seconds:
- mph = 3600 / 450 = **8.0**
- kph = 8.0 × 1.609344 = **12.87** → displayed as 12.9

### 7.2 Track splits at goal pace

```js
$("s400").textContent  = fmtTime(T * (400  / METERS_PER_MILE));
```

If you maintain pace `T` (seconds per mile), then in 400 metres you cover 400/1609.344 of a mile and spend `T × (400/1609.344)` seconds. For a 7:30 mile that's 450 × 0.2485 = **111.85s = 1:52**.

Same logic for 800m and 1200m. This is just unit conversion — no fancy physics.

### 7.3 The Riegel race-time formula

```js
const finish = mileSec * Math.pow(miles, RIEGEL_EXP);  // RIEGEL_EXP = 1.06
```

This is the most interesting bit of math in the codebase. It comes from a 1981 paper by Pete Riegel, an American running researcher, who fit running performance data to a power law:

> **T2 = T1 × (D2 / D1)^1.06**

Where T1 is your time at distance D1, and T2 is the predicted time at distance D2. The exponent **1.06** captures **fatigue**: if you doubled the distance, a perfectly even-energy runner would take exactly 2× as long, but real humans slow down a bit, so it's 2^1.06 ≈ 2.085× as long.

In code, with D1 = 1 mile and T1 = goal mile time:

```js
const miles = r.km / KM_PER_MI;                       // race distance in miles
const finish = mileSec * Math.pow(miles, RIEGEL_EXP); // predicted seconds
const pacePerMi = finish / miles;                     // average pace
```

For a 7:30 mile and a 5K (3.107 mi):
- finish = 450 × 3.107^1.06 = 450 × 3.326 = **1497s = 24:57**
- pace = 1497 / 3.107 = **482s/mi = 8:02/mi**

Riegel is a rough predictor (real performance depends on training volume, terrain, weather), but for a rule of thumb it's accurate within a few minutes for most recreational runners. The 1.06 exponent slightly over-predicts marathon times for trained marathoners, who can get closer to 1.05–1.055. We're keeping it simple.

---

## 8. The linked-input pattern (the cool trick)

The planner has four pairs of "linked" inputs:
- Distance: `distMi` ↔ `distKm`
- Pace: `paceMi*` ↔ `paceKm*`

Editing one updates the other in real time. How do we avoid an infinite loop where each update triggers the other?

```js
distMiEl.addEventListener("input", () => {
  const v = parseFloat(distMiEl.value);
  if (Number.isFinite(v) && v >= 0) distKmEl.value = (v * KM_PER_MI).toFixed(2);
  renderPlan();
});
```

The key insight: **setting `el.value = ...` programmatically does NOT fire the `input` event.** Only user keystrokes do. So writing to `distKmEl.value` from inside the `distMiEl` handler is safe — `distKmEl`'s own handler won't run, no loop.

This is a fundamental property of DOM input events you can rely on across all browsers. (If you ever need to *force* a programmatic update to also fire handlers, use `el.dispatchEvent(new Event("input"))`. We don't.)

---

## 9. Deployment: GitHub Pages, end to end

The deploy pipeline has zero pieces we built — it's entirely GitHub:

```
git push origin main
       │
       ▼
GitHub receives the new commit
       │
       ▼
"pages build and deployment" GitHub Action auto-runs
  1. checks out the main branch
  2. publishes the contents of the chosen folder (we use root)
  3. invalidates the CDN cache for our subdomain
       │
       ▼
The new index.html is served from
https://una95singo.github.io/RunMate/
```

Settings → Pages → Source = "Deploy from a branch", branch = `main`, folder = `/`. That single config is the whole CI/CD setup.

### 9.1 Why hard-refresh matters

Browsers cache HTML aggressively. After a deploy, your phone may still hold the old `index.html` for hours. Three ways to bust the cache:

- **Hard refresh:** `Cmd/Ctrl + Shift + R`. Bypasses the disk cache for this load.
- **Cache-buster query param:** `?v=2` — the browser treats `?v=2` as a different URL.
- **Service Worker** (advanced): write a SW that updates itself. Overkill for now.

In production apps, the standard fix is to give every JS/CSS file a hashed name (`app.a8b3.js`) so a new deploy means a new filename. Single-file HTML can't do that for itself (the URL is always `index.html`), so we live with hard-refresh.

### 9.2 Why an empty commit can "fix" things

```sh
git commit --allow-empty -m "Trigger Pages rebuild"
git push origin main
```

GitHub Pages only rebuilds when commits land on the deploy branch. An empty commit creates a new commit hash without changing files — enough to trigger a fresh build, useful when Pages gets stuck or you suspect a build was skipped. This is purely a "kick the build pipeline" trick; the file content is identical.

---

## 10. Where to go next (suggested explorations)

If you want to keep upskilling, here are the most fruitful next steps in roughly increasing difficulty:

1. **Persist user state to `localStorage`.** Save the last entered mile time so it's there next session.
   ```js
   localStorage.setItem("paceGoal", T);
   const saved = localStorage.getItem("paceGoal");
   ```

2. **URL-shareable state.** Encode the inputs in the URL hash (`#t=7:30`) so users can share or bookmark a specific pace. Read with `location.hash`, write with `history.replaceState`.

3. **Service Worker for offline.** Make the app work with no internet. Adds a `service-worker.js`, registered on first load; subsequent loads serve from the SW cache.

4. **A real test suite.** Extract the pure functions (`fmtTime`, Riegel, `readPaceSec`) into a separate `pace.js` module, then write tests with Vitest or Node's built-in test runner.

5. **Convert to TypeScript.** Once tests exist, types become cheap to add and catch a class of bugs you can't see in JS.

6. **Component-ise.** When the file passes ~1000 lines or you want a second page, *that* is the moment to introduce a framework. Svelte is the gentlest jump from vanilla JS.

For each of these, we'd be **adding** a layer (persistence, offline, types, tests), not rewriting what's here. The current vanilla foundation stays useful underneath.

---

## Appendix: file map

```
index.html     ← the entire app
DOCS.md        ← this file
.git/          ← version control
```

That's the project. The simplicity is a feature.
