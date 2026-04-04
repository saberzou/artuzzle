# CLAUDE.md — Artuzzle

## Project Overview

Artuzzle is a single-page HTML5 art puzzle game. Players reassemble famous Impressionist paintings by drag-and-drop reordering of horizontal strips. A "hard mode" activates on replay, adding vertical cuts to each strip that can be individually toggled.

The entire application lives in **one file**: `index.html` (~674 lines). There is no build system, no package manager, and no server-side code.

---

## Repository Structure

```
artuzzle/
├── index.html        # Entire application: CSS + HTML + JavaScript
├── README.md         # Minimal placeholder
└── images/           # 15 local Impressionist painting JPEGs
    ├── ballet-class.jpg
    ├── caillebotte-floor.jpg
    ├── caillebotte-rainy.jpg
    ├── cassatt-bath.jpg
    ├── degas-absinthe.jpg
    ├── folies-bergere.jpg
    ├── impression-sunrise.jpg
    ├── monet-poppies.jpg
    ├── monet-rouen.jpg
    ├── moulin-galette.jpg
    ├── pissarro-boulevard.jpg
    ├── renoir-swing.jpeg
    ├── sisley-flood.jpg
    ├── the-cradle.jpg
    └── water-lilies.jpg
```

Images are bundled locally (not fetched remotely) to ensure accessibility behind the GFW.

---

## index.html Structure

| Lines     | Section         | Description                                                  |
|-----------|-----------------|--------------------------------------------------------------|
| 1–189     | `<style>`       | CSS: custom properties, screen layouts, strip animations     |
| 191–245   | `<body>` HTML   | DOM for 5 screens                                            |
| 247–369   | `PAINTINGS` data | Array of 15 painting objects (title, artist, year, img, desc)|
| 371–395   | Navigation      | `goTo()`, `showScreen()` — screen transitions via GSAP       |
| 397–405   | Landing screen  | `animateLanding()` — letter-by-letter title entrance         |
| 407–422   | Level grid      | `buildLevelGrid()` — renders painting cards with lock state  |
| 424–623   | Puzzle engine   | `startPuzzle()`, `buildPuzzle()`, drag/pointer event logic   |
| 625–635   | Win detection   | `checkWin()` — validates strip order + half states           |
| 637–671   | Win overlay     | `showWin()`, `closeWin()` — GSAP-animated victory screen     |

---

## Screens (Navigation Flow)

```
landing → categories → levels → puzzle → win-overlay (then back to levels)
```

Each screen is a `<div class="screen">`. Only one has class `active` at a time. Transitions are GSAP opacity fades. The `categories` screen currently shows placeholder cards (Impressionists, Post-Impressionists, Renaissance); only the Impressionists collection is implemented.

---

## Game Mechanics

### Normal Mode
- The painting is sliced into `NUM_STRIPS = 6` horizontal strips using Canvas API.
- Strips are shuffled (guaranteed: no strip stays in its correct position via `shuffleArray()`).
- Player drags strips vertically to reorder them.
- Win condition: `strip.dataset.correctIndex === i` for every strip in DOM order.

### Hard Mode (replay)
- Triggered when `solved.has(currentPainting.index)` — i.e., the painting has already been solved once.
- Each strip is split into left/right halves using two `<canvas>` elements inside a `.strip-halves` div.
- 3–5 randomly chosen strips start with halves swapped (`.strip-halves.swapped`).
- Player taps/clicks a strip to toggle its halves; state tracked via `halvesDiv.dataset.swapped`.
- Win condition: all strips in correct order **and** no `.strip-halves` has `dataset.swapped === '1'`.

### Drag Implementation
- Uses Pointer Events API (`pointerdown`, `pointermove`, `pointerup`, `pointercancel`) for unified mouse/touch/pen support.
- `setPointerCapture` ensures the drag continues if the pointer leaves the strip.
- `SWAP_COOLDOWN = 180ms` — minimum time between swaps to prevent jitter.
- `SWAP_THRESHOLD = 0.55` — pointer must cross 55% into a neighbor strip to trigger a swap.
- GSAP animates the displaced neighbor strip to slide into place (`y: ±stripH → 0`).

---

## Key Constants

| Constant         | Value  | Purpose                                    |
|------------------|--------|--------------------------------------------|
| `NUM_STRIPS`     | `6`    | Number of horizontal strips per puzzle     |
| `EASE`           | `"power2.out"` | GSAP easing used throughout          |
| `SWAP_COOLDOWN`  | `180`  | ms between swap events (jitter prevention) |
| `SWAP_THRESHOLD` | `0.55` | Fraction of neighbor height to cross       |

---

## State Management

All state is in-memory globals (no localStorage, no backend):

| Variable           | Type          | Description                                      |
|--------------------|---------------|--------------------------------------------------|
| `PAINTINGS`        | `Array`       | Static painting data (15 entries)                |
| `solved`           | `Set<number>` | Indices of completed paintings                   |
| `currentPainting`  | `Object`      | Active painting with spread copy + `.index`      |
| `moves`            | `number`      | Move counter for current puzzle                  |
| `strips`           | `Array<HTMLElement>` | Current strip DOM elements                |

State resets on page reload. There is intentionally no persistence.

---

## External Dependencies (CDN only)

- **GSAP 3** — `https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js` — all animations
- **Google Fonts** — Playfair Display (headings), Instrument Sans (body)

No npm packages, no bundler, no local node_modules.

---

## CSS Conventions

- **CSS custom properties** defined on `:root`: `--black`, `--white`, `--gray`, `--light`, `--accent`, `--font-serif`, `--font-sans`, `--ease`
- **Fluid typography** via `clamp()` for responsive sizing without breakpoints
- **Accessibility**: `focus-visible` outlines on all interactive elements; `tabindex` management for locked level cards
- **Motion reduction**: `@media (prefers-reduced-motion: reduce)` disables animations
- Class names are lowercase hyphenated: `.strip`, `.strip-halves`, `.level-card`, `.win-overlay`, etc.
- State classes: `.active`, `.dragging`, `.locked`, `.solved`, `.swapped`

---

## JavaScript Conventions

- **UPPERCASE** for module-level constants (`NUM_STRIPS`, `EASE`, `SWAP_COOLDOWN`)
- **camelCase** for functions and variables
- No classes — procedural functions with DOM manipulation
- Functional array methods (`map`, `forEach`, `every`, `filter`) preferred
- Canvas rendering via `getContext('2d').drawImage()` for image slicing
- GSAP used for all transitions; no CSS transition/animation for interactive elements

---

## Adding New Paintings

Add an entry to the `PAINTINGS` array in `index.html`:

```javascript
{
  title: "Painting Title",
  artist: "Artist Name",
  year: "YYYY",
  img: "images/filename.jpg",   // place file in images/
  desc: "1–2 sentence art history description.",
  unlocked: true                // false = locked until previous painting solved
}
```

The unlock chain: a painting at index `i` unlocks automatically when `solved.has(i - 1)`, unless `unlocked: true` is set explicitly.

---

## Development Workflow

There is no build step. To run locally:

```bash
# Any static file server works, e.g.:
python3 -m http.server 8080
# then open http://localhost:8080
```

Or open `index.html` directly in a browser (note: `crossOrigin = 'anonymous'` on images may cause issues with `file://` protocol on some browsers; a local server is preferred).

### Testing

There is no automated test suite. Verification is manual:
1. Open in browser
2. Complete a puzzle (normal mode)
3. Re-complete the same puzzle (hard mode — halves should appear)
4. Verify win overlay shows correct painting info
5. Verify "Continue" button returns to level grid in one tap (no double-tap issue)

---

## Deployment

Deploy by copying `index.html` and the `images/` directory to any static web host. No server configuration required.

---

## Known Architectural Decisions

- **Single-file design**: Intentional — simplifies deployment and sharing. Do not split into separate JS/CSS files unless the project complexity clearly warrants it.
- **No localStorage**: State resets on reload by design — each session is a fresh start.
- **Local images**: Moved from remote URLs to local files (commit `c225efc`) to work behind the GFW.
- **Pointer Events over Touch Events**: Chosen for unified mouse/touch/pen handling; `setPointerCapture` prevents lost drags.
- **Canvas over CSS background-image**: Used for image slicing to avoid browser quirks with background-position on sub-pixel strips.
