# RunMate

> A minimalist pace calculator for runners — mile pace, metric ↔ imperial conversion, and race-time predictions. Single HTML file, no dependencies.

**Live:** [una95singo.github.io/RunMate](https://una95singo.github.io/RunMate/)

---

## Features

- **Goal mile pace** — enter a target time as `mm:ss` and see treadmill speed (mph & kph), 400 m / 800 m / 1200 m track splits, and easy (+90 s/mi) and race (−30 s/mi) variants.
- **Run planner** — distance and pace inputs that link bidirectionally between miles and kilometres. Computes finish time and per-km / per-mile cumulative splits for any distance, with presets for 5K, 10K, half, and full marathon.
- **Race predictions** — uses the [Riegel formula](https://en.wikipedia.org/wiki/Pete_Riegel) (`T₂ = T₁ × (D₂/D₁)^1.06`) to project equivalent finish times for 5K, 10K, half marathon, and marathon from your goal mile pace.
- **Mobile-first dark UI** — designed for one-handed use on a phone. Tabular numerals so values don't jitter as they update. Numeric keypad on iOS/Android.
- **Bookmark-ready** — `apple-mobile-web-app-*` meta tags and an inline SVG icon make "Add to Home Screen" feel like a native app launch.

---

## Quick start

The project is a single HTML file with no build step.

### Run locally

Open the file directly in a browser:

```bash
git clone https://github.com/Una95Singo/RunMate.git
cd RunMate
open index.html      # macOS
xdg-open index.html  # Linux
start index.html     # Windows
```

Or serve it over HTTP if you prefer (recommended for testing iOS-specific behaviour like `inputmode`):

```bash
# Python 3
python -m http.server 8000

# Node (with npx)
npx serve .
```

Then visit `http://localhost:8000`.

### Edit

Open `index.html` in any editor. The file is structured in three contiguous sections:

| Section | Lines | Purpose |
|---|---|---|
| `<style>` | top | Design tokens, layout, components |
| `<body>` markup | middle | Semantic structure |
| `<script>` | bottom | Pure helpers, render functions, event wiring |

Save and reload the browser. There is no compile or restart step.

---

## Project structure

```
.
├── index.html    # The entire app (HTML + CSS + JS)
├── DOCS.md       # Architecture walkthrough — read this for the "why"
└── README.md     # You are here
```

That is the project. The simplicity is intentional; see [`DOCS.md`](./DOCS.md) for the rationale.

---

## Tech notes

- **No framework, no bundler, no dependencies.** The file ships as-authored.
- **Vanilla JavaScript**, wrapped in an IIFE for module-like scoping.
- **CSS custom properties** for design tokens (`--accent`, `--panel`, `--radius`, …).
- **CSS Grid** for layout primitives, **Flexbox** for row alignment.
- **Mobile-first** styles by default; `@media (min-width: 480px)` enhancements for larger screens.
- **No analytics, no tracking, no third-party scripts.**

Browser support: modern evergreen browsers (Chrome, Firefox, Safari, Edge) and iOS Safari 14+. No polyfills required.

---

## Deployment

The site is deployed via **GitHub Pages** from the root of the `main` branch.

Every push to `main` triggers the `pages build and deployment` workflow automatically. After a successful run (typically 30–90 s), the new version is live at the URL above.

To force a rebuild without changing files (e.g. when a build appears stuck):

```bash
git commit --allow-empty -m "Trigger Pages rebuild"
git push origin main
```

After deployment, browsers may serve the previous HTML from cache — hard-refresh with `Cmd/Ctrl + Shift + R`, or append a cache-buster query (`?v=2`).

---

## Architecture

For a thorough walkthrough of the design choices, layout strategy, math (including the Riegel derivation), and the linked-input pattern, see [**DOCS.md**](./DOCS.md). It covers:

1. Why single-file vanilla JS
2. The `<head>` metadata, including iOS home-screen support
3. The CSS layer — design tokens, mobile-first layout, Grid/Flex primitives
4. The HTML layer — semantic elements, accessibility, mobile keyboard hints
5. The JavaScript layer — IIFE, pure helpers, render functions, event wiring
6. The math — pace ↔ speed, track splits, Riegel race prediction
7. The linked-input pattern (and why programmatic `value =` does not loop)
8. GitHub Pages deployment and cache behaviour
9. Possible next steps for the project

---

## Roadmap

Possible directions (see [DOCS §10](./DOCS.md#10-possible-next-steps) for detail):

- [ ] Persist last-used inputs to `localStorage`
- [ ] Shareable URL state (`#t=7:30&d=5K`)
- [ ] Service Worker for offline use
- [ ] Interval / tempo workout builder
- [ ] Heart-rate / effort-zone calculator
- [ ] Test suite for the pure helpers

---

## Contributing

Issues and pull requests are welcome. For changes to the math or the public UI, please include a short note explaining the reasoning so it can be reflected in `DOCS.md`.

---

## License

MIT — see [`LICENSE`](./LICENSE) if present, otherwise treat as MIT.
