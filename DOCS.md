# Pace — Architecture Notes

A walkthrough of how this project is built and why. Each section is self-contained and can be read in any order.

---

## 1. What this is

`index.html` is a single-file web app. One file contains the page structure (HTML), the visual design (CSS, embedded in `<style>`), and the behaviour (JavaScript, embedded in `<script>`). There is no build step, no `npm install`, no bundler, no framework, and no server. A browser opens the file and it runs.

That choice shapes most of the other decisions in the project.

### Why single-file

| Trade-off | Single-file (this project) | Typical bundled SPA |
|---|---|---|
| Time to first paint | One HTTP request | Multiple JS chunks, parse, hydrate |
| Build complexity | None | Bundler, transpiler, dev server |
| Deployment | Static host, drop one file | CI pipeline, dist artefact |
| Suits | Tools, calculators, prototypes | Apps with routing and shared state |

For a pace calculator the math is small, the UI fits one screen, and the desired user experience is "open the bookmark and it loads instantly." Single-file matches that shape. Once the project grows past a few thousand lines or needs persistence, routing, or background sync, that is the moment to introduce a framework — not before.

### What single-file gives up

- Component reuse across pages. There is only one page.
- Type safety. There is no TypeScript; the small, pure functions partially compensate.
- Build-time optimisations. The file is already small (~25 KB), so this does not matter.

---

## 2. The three layers

The file is laid out top-to-bottom in three stacked layers:

```
<!doctype html>
<html><head>
  <meta ...>             metadata, icons, title
  <style>...</style>     LAYER 1: design (CSS)
</head>
<body>
  <main>...</main>       LAYER 2: structure (HTML)
  <script>...</script>   LAYER 3: behaviour (JS)
</body>
</html>
```

The script tag sits at the end of `<body>` so that every element it queries (`getElementById("min")`, etc.) already exists in the DOM by the time the script runs. This removes any need for a `DOMContentLoaded` listener.

---

## 3. The `<head>`: metadata that earns its keep

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="theme-color" content="#0a0b0e" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-title" content="Pace" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
```

Each line does specific work:

- **`viewport`** tells mobile browsers to render at the device's actual width rather than emulating desktop. `viewport-fit=cover` allows the page to extend beneath the iPhone notch; the layout then uses `env(safe-area-inset-*)` (see §4) to keep content inside the safe area.
- **`theme-color`** colours the browser's address bar and the iOS status bar in standalone mode to match the page background.
- The three `apple-mobile-web-app-*` tags configure iOS so that when the page is added to the home screen, it launches in standalone mode without Safari chrome, with the title "Pace" and a black status bar.

### Inline SVG icons

```html
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml;utf8,<svg ...>" />
<link rel="apple-touch-icon" href="data:image/svg+xml;utf8,<svg ...>" />
```

Two notes:

1. **Data URI.** The icon is encoded directly into the HTML as `data:image/svg+xml;utf8,...` instead of being a separate file. This preserves the single-file constraint and saves an HTTP request. `#` characters inside the SVG must be URL-encoded as `%23`.
2. **SVG.** Modern browsers and iOS 18+ accept SVG for both favicons and home-screen icons, removing the need for multiple PNG sizes.

---

## 4. The CSS layer

### 4.1 Design tokens

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

CSS custom properties on `:root` act as design tokens. The rest of the stylesheet references them via `var(--accent)`. Three benefits:

1. Centralised values — changing the accent colour means editing one declaration.
2. Self-documenting — `var(--muted)` reads more clearly than `#7e8696`.
3. Themeable — a light mode is added by redefining the tokens inside `@media (prefers-color-scheme: light)` without touching the rest of the CSS.

This is the CSS-native equivalent of what design systems like Tailwind and Material UI provide, without the framework dependency.

### 4.2 Mobile-first layout

```css
body {
  padding: env(safe-area-inset-top) 14px env(safe-area-inset-bottom);
}
.wrap { max-width: 560px; margin: 0 auto; padding: 18px 0 36px; }
```

- `env(safe-area-inset-*)` are values the browser exposes for the physical space hidden by the notch or home indicator. The body padding keeps content out of those regions.
- `max-width: 560px` with `margin: 0 auto` caps the readable column. On a phone this is wider than the viewport and has no effect; on tablet or desktop it centres a phone-width column instead of stretching numbers across the screen.

Mobile-first means the default styles target the small screen, and media queries enhance for larger viewports rather than the reverse:

```css
@media (min-width: 480px) {
  h1 { font-size: 22px; }
  .summary .big { font-size: 52px; }
}
```

Phone-portrait users do not download or apply these rules.

### 4.3 Layout primitives

The codebase uses CSS Grid for "columns of equal-ish things" and Flexbox for "a row where some children stretch and some do not."

Grid example — the splits row:
```css
.splits { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; }
```
Three equal-width columns with an 8px gutter. No floats, no margin arithmetic.

Flexbox example — the time input:
```css
.time-input { display: flex; align-items: baseline; gap: 6px; }
.time-input input { flex: 1 1 0; min-width: 0; }
```
Children laid out in a row, aligned on the text baseline. `flex: 1 1 0` is the standard "fill the available space equally" recipe. `min-width: 0` is required because flex items will not otherwise shrink below the intrinsic content width of an `<input type="number">`, which can cause overflow.

The race-prediction row demonstrates Grid with three different sizing strategies:
```css
.race {
  display: grid;
  grid-template-columns: 80px 1fr auto;
  gap: 10px;
}
```
Fixed-width label, flexible primary cell, content-sized trailing cell.

### 4.4 Tabular numerals

```css
font-feature-settings: "tnum" 1;
```

Most modern fonts ship two glyph sets for digits: proportional (visually balanced for body text) and tabular (every digit has the same advance width). For pace times that update on each keystroke (`7:30 → 7:31 → 7:32`), tabular numerals prevent visual jitter. Pure typography polish, no runtime cost.

---

## 5. The HTML layer

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

### 5.1 Semantic elements

`<main>`, `<header>`, `<section>`, and `<button>` are used in place of generic `<div>`s wherever the meaning fits. Screen readers, search crawlers, and reader-mode browsers all understand the implied roles. Keyboard navigation between `<button>` elements works automatically.

### 5.2 `aria-label` on unlabelled controls

```html
<input id="min" ... aria-label="minutes" />
```

The two number inputs are visually unambiguous (a large `mm:ss` display), but a screen reader cannot infer "these are minutes and seconds." `aria-label` provides the assistive-tech equivalent of a `<label>` element without consuming visual space. A real `<label for="...">` is preferable when room exists; for the time input it does not.

### 5.3 Mobile keyboard hints

```html
<input type="number" inputmode="numeric" pattern="[0-9]*" />
```

Three attributes, three roles:

- `type="number"` — the spec-correct type, which activates browser-level number validation.
- `inputmode="numeric"` — instructs iOS and Android to display the numeric keypad rather than the full QWERTY keyboard.
- `pattern="[0-9]*"` — required by older iOS Safari versions in addition to `inputmode` to honour the numeric keypad.

Together these make number entry feel native on mobile.

### 5.4 Why the planner seconds field uses `type="text"`

```html
<input id="paceMiSec" type="text" inputmode="numeric" pattern="[0-9]*" maxlength="2" value="00" />
```

`<input type="number">` normalises its value, so `value="00"` displays as `"0"` and the leading zero is lost. For a clock display the leading zero matters. Switching to `type="text"` with `inputmode="numeric"` preserves the literal string while still surfacing the numeric keypad on mobile. The `maxlength="2"` cap prevents accidental overflow.

---

## 6. The JavaScript layer

The entire script is wrapped in an IIFE (Immediately Invoked Function Expression):

```js
(function () {
  "use strict";
  // ...
})();
```

How it works:

- `function () { ... }` is an anonymous function definition.
- The trailing `()` invokes it immediately.
- The outer parentheses make the parser accept the function as an expression.

The effect is module-like scoping: every variable inside the IIFE is local to it and does not leak onto the global `window` object. An ES module (`<script type="module">`) achieves the same isolation in modern browsers; the IIFE works everywhere with no extra HTTP request and is appropriate for one inline script.

### 6.1 The `$` helper

```js
const $ = (id) => document.getElementById(id);
```

A six-character convenience wrapper. This is not jQuery — only a familiar shorthand for a frequently called API.

### 6.2 Pure helpers, then orchestration

The script is laid out bottom-up: small pure functions first, then DOM-touching renderers, then event wiring.

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

This function takes a number and returns a string. It does not touch the DOM, does not read or write any global state, and always returns the same output for the same input. Functions of this shape are straightforward to reason about and trivial to unit-test. Six of them exist in the file: `clampInt`, `fmtTime`, `fmtTimeLong`, `fmtSpeed`, `readPaceSec`, `writePace`. Keeping pure logic separated from DOM access makes both halves easier to maintain.

### 6.3 Render functions

There are two main render functions, `renderGoal()` and `renderPlan()`, plus `renderRaces(mileSec)`, which is invoked from inside `renderGoal` because race predictions depend on the goal mile time.

The pattern:

```js
function renderGoal() {
  const T = totalSeconds();                         // 1. read state from inputs

  if (T <= 0) { /* show "—" placeholders */ return; } // 2. handle invalid/empty

  $("goalPace").innerHTML = fmtTime(T) + ...;       // 3. compute & write outputs
  const mph = 3600 / T;
  $("mph").textContent = fmtSpeed(mph);
  // ...
  renderRaces(T);                                   // 4. cascade dependent renders
}
```

This is the render-on-input pattern, hand-rolled. React, Vue, and Svelte automate it with virtual DOMs or reactive signals; for eleven output fields, the manual version is sufficient and keeps the code transparent.

### 6.4 Event wiring

```js
minEl.addEventListener("input", renderGoal);
secEl.addEventListener("input", renderGoal);
```

Every keystroke re-runs `renderGoal()`. The full re-render involves a handful of arithmetic operations and `textContent` writes, which complete in well under a millisecond on modern browsers. Should the work ever grow large enough to matter, the standard remedies are throttling with `requestAnimationFrame` or computing diffs incrementally.

---

## 7. The math

### 7.1 Pace and speed

A pace is "seconds per mile." A speed is "miles per hour." They are reciprocals scaled by 3600 (seconds per hour):

```js
const mph = 3600 / T;             // T = seconds per mile
const kph = mph * KM_PER_MI;      // KM_PER_MI = 1.609344
```

For a 7:30 mile (T = 450 s):
- mph = 3600 / 450 = **8.0**
- kph = 8.0 × 1.609344 = **12.87** → displayed as 12.9

### 7.2 Track splits at goal pace

```js
$("s400").textContent = fmtTime(T * (400 / METERS_PER_MILE));
```

If pace `T` (seconds per mile) is held constant, then 400 metres covers `400 / 1609.344` miles and takes `T × (400 / 1609.344)` seconds. For a 7:30 mile that is 450 × 0.2485 = **111.85 s = 1:52**. The same logic applies to 800 m and 1200 m.

### 7.3 The Riegel race-time formula

```js
const finish = mileSec * Math.pow(miles, RIEGEL_EXP);  // RIEGEL_EXP = 1.06
```

From a 1981 paper by Pete Riegel, an American running researcher, who fit running performance data to a power law:

> **T2 = T1 × (D2 / D1)^1.06**

Where T1 is a known time at distance D1 and T2 is the predicted time at distance D2. The exponent **1.06** captures the slow-down a runner experiences as distance grows: at constant energy expenditure the exponent would be 1.0, but real performance follows roughly `^1.06`, so doubling the distance takes about `2^1.06 ≈ 2.085×` as long.

In code, with D1 = 1 mile and T1 = goal mile time:

```js
const miles = r.km / KM_PER_MI;                       // race distance in miles
const finish = mileSec * Math.pow(miles, RIEGEL_EXP); // predicted seconds
const pacePerMi = finish / miles;                     // average pace
```

For a 7:30 mile predicting a 5K (3.107 mi):
- finish = 450 × 3.107^1.06 = 450 × 3.326 = **1497 s = 24:57**
- pace = 1497 / 3.107 ≈ **482 s/mi = 8:02/mi**

Riegel is a rough predictor — actual performance depends on training volume, terrain, and conditions — but for a rule of thumb it is accurate within a few minutes for most recreational runners. The 1.06 exponent slightly over-predicts marathon times for trained marathoners, who typically follow closer to 1.05–1.055.

---

## 8. The linked-input pattern

The planner has four pairs of linked inputs:
- Distance: `distMi` ↔ `distKm`
- Pace: `paceMi*` ↔ `paceKm*`

Editing one updates the other in real time. The implementation:

```js
distMiEl.addEventListener("input", () => {
  const v = parseFloat(distMiEl.value);
  if (Number.isFinite(v) && v >= 0) distKmEl.value = (v * KM_PER_MI).toFixed(2);
  renderPlan();
});
```

The reason no infinite loop occurs: **setting `el.value = ...` programmatically does not fire the `input` event.** Only user input dispatches it. Writing to `distKmEl.value` from inside the `distMiEl` handler is therefore safe — the second handler does not fire as a side effect.

This is a stable property of DOM input events across all browsers. To deliberately fire handlers after a programmatic update, dispatch the event manually: `el.dispatchEvent(new Event("input"))`.

---

## 9. Deployment

The deploy pipeline is entirely GitHub-managed:

```
git push origin main
       │
       ▼
GitHub receives the new commit
       │
       ▼
"pages build and deployment" Action runs:
  1. checks out main
  2. publishes the contents of the configured folder (root)
  3. invalidates the CDN cache for the subdomain
       │
       ▼
The new index.html is served from
https://una95singo.github.io/RunMate/
```

Configuration: Settings → Pages → Source = "Deploy from a branch", branch = `main`, folder = `/`. That single setting is the entire CI/CD configuration.

### 9.1 Browser caching

Browsers cache HTML aggressively. After a deploy, a previously loaded page may still be served from disk for some time. Three remedies:

- **Hard refresh:** `Cmd/Ctrl + Shift + R` bypasses the disk cache for the next load.
- **Cache-buster query parameter:** appending `?v=2` makes the browser treat the URL as new.
- **Service Worker:** a SW can manage update lifecycles explicitly. Worth introducing only when offline support is also needed.

In larger deployments the standard fix is content-addressed asset filenames (e.g. `app.a8b3.js`) so a new build implies a new URL. A single `index.html` cannot do that for itself, so hard-refresh remains the practical option.

### 9.2 Empty commits

```sh
git commit --allow-empty -m "Trigger Pages rebuild"
git push origin main
```

GitHub Pages only rebuilds when commits land on the deploy branch. An empty commit creates a new commit hash without changing files, which is enough to trigger a fresh build — useful when a build appears to have been skipped or the live site is out of sync. The file content is unchanged.

---

## 10. Possible next steps

Future directions, ordered roughly by complexity:

1. **Persist user state to `localStorage`.** Save the last entered mile time and restore it on next load.
   ```js
   localStorage.setItem("paceGoal", T);
   const saved = localStorage.getItem("paceGoal");
   ```

2. **URL-shareable state.** Encode the inputs in the URL hash (`#t=7:30`) so a specific pace can be bookmarked or shared. Read with `location.hash`, write with `history.replaceState`.

3. **Service Worker for offline.** Make the app function without a network. Adds a `service-worker.js` registered on first load; subsequent loads serve from the SW cache.

4. **Test suite.** Extract the pure functions (`fmtTime`, Riegel, `readPaceSec`) into a separate module and add tests with Vitest or Node's built-in test runner.

5. **TypeScript.** Once tests exist, types become inexpensive to add and prevent a class of bugs that JavaScript silently accepts.

6. **Componentise.** When the file passes ~1000 lines or a second page is needed, that is the moment to introduce a framework. Svelte is the smallest jump from vanilla JS.

Each of these adds a layer (persistence, offline, types, tests) without rewriting the existing foundation.

---

## Appendix: file map

```
index.html     the entire app
DOCS.md        this document
.git/          version control
```
