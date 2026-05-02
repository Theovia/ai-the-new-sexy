# Vectorization log

How the hand-lettered neon logo went from a 2.3 MB raster PNG to an inline SVG that draws itself stroke-by-stroke on entrance.

## Why

The PNG version was already styled with a four-layer FX stack (drop-shadow ×2 + radial bloom + cursor-reactive `feDisplacementMap`) — visually convincing. But three signature effects were impossible without vector paths:

1. **DrawSVG-style entrance** — animate `stroke-dashoffset` so the lettering "writes itself" on load.
2. **Per-stroke control** — different filter / glow per group (only "Ai" gets blue, only "TheNewSexy" gets pink).
3. **Resolution independence** — Retina, 4K, print, poster zoom — all stay sharp.

Plus a side-effect: file weight dropped from **2.34 MB → 162 KB** (≈14× smaller).

## Tool

[`vtracer`](https://github.com/visioncortex/vtracer) — Rust CLI, color-aware, spline-mode (curves) for handwriting.

```bash
cargo install vtracer
```

(Not on Homebrew; install via cargo.)

## The journey: v0 (failed) → v7 (winner)

The first vectorization attempt was *too* aggressive and dropped the bright lettering strokes into the halo's color regions. The lettering became invisible — the page rendered as a magenta blob.

Five subsequent traces were rendered side-by-side at the same viewport to find the actual sweet spot. v7 — moderate filter_speckle, mid color_precision, gradient_step 32 — preserved the lettering while still cutting size 6.4× vs the PNG.

| version | params summary | size | paths | result |
|---------|----------------|------|-------|--------|
| v0 (initial) | filter_speckle 16 / color_precision 3 / gradient_step 64 / cutout | 163 KB | 30 | ❌ lettering lost, blob render |
| v2 (defaults) | speckle 4 / precision 6 / grad 16 | 1.4 MB | 1071 | ✓ legible, but heavier than necessary |
| v3 mid | speckle 8 / precision 5 / grad 24 | 689 KB | 254 | ✓ legible, soft halo |
| v4 max fidelity | speckle 2 / precision 8 / grad 8 | 2.5 MB | 4115 | ✓✓ near-PNG, but **heavier** than PNG (anti-vector) |
| v5 | speckle 6 / precision 6 / grad 20 | 1.0 MB | 492 | ✓ same as v3 with more paths |
| v6 | speckle 10 / precision 5 / grad 28 | 626 KB | 167 | ✓ |
| **v7 (winner)** | speckle 12 / precision 4 / grad 32 / stacked | **366 KB** | **93** | ✓✓ legible, 6.4× lighter than PNG |

Side-by-side renders are in `_compare/variants.png` and `_compare/variants2.png`.

## Tuned command (v7 — currently shipped)

```bash
cd assets/
vtracer \
  --input logo.png \
  --output logo.svg \
  --mode spline \
  --colormode color \
  --filter_speckle 12 \
  --color_precision 4 \
  --gradient_step 32 \
  --corner_threshold 60 \
  --segment_length 6 \
  --splice_threshold 45 \
  --path_precision 1 \
  --hierarchical stacked
```

**Why these flags (v7):**

| Flag | Default | v7 | Reason |
|------|---------|----|----|
| `--mode spline` | spline | spline | curves, not polygons — preserves handwriting feel |
| `--filter_speckle 12` | 4 | 12 | drops most halo noise but keeps lettering strokes |
| `--color_precision 4` | 6 | 4 | quantize to 4 bits — enough fidelity for neon palette |
| `--gradient_step 32` | 16 | 32 | wider color steps → fewer halo layers, but lettering colors survive |
| `--segment_length 6` | 4 | 6 | smoother curves on the calligraphy |
| `--splice_threshold 45` | 45 | 45 | default — clean spline joins |
| `--path_precision 1` | 8 | 1 | one decimal → smaller file, no visible diff |
| `--hierarchical stacked` | stacked | stacked | preserves the layered structure (halo → mid → strokes) |

The lesson: `gradient_step 64` (max simplification) collapses too many color regions into the dominant one. `gradient_step 32` is the threshold where the lettering's distinct colors survive as their own paths.

## Result (v7)

```
input  · logo.png · 2,342,613 bytes (2.34 MB)
output · logo.svg ·   366,190 bytes (366 KB)
paths  · 93
size   · 6.4× reduction
visual · lettering legible, halo stepped (vs PNG smooth alpha)
```

## Inlining + animation

Vector benefits only fully unlock when the SVG is **inlined** in the HTML (so JS can reach individual `<path>` elements). The original `<img src="assets/logo.png">` was replaced with:

```html
<svg id="logo" class="logo-svg" viewBox="0 0 2000 1520"
     preserveAspectRatio="xMidYMid meet"
     style="filter:url(#liquid) drop-shadow(...) drop-shadow(...);"
     aria-label="Ai · The New Sexy" role="img">
  <path d="..." fill="#FF1F71"/>
  <path d="..." fill="#0000FF"/>
  ... (28 more paths)
</svg>
```

The `assets/logo.svg` file is kept for portability / external use, but the rendered logo on the page is the inline copy.

### Two visual states · one element

The same `<svg>` represents the logo in two states:

1. **`is-drawing`** — fills go transparent, strokes appear (white, 1.4 px, rounded). Each path's `stroke-dasharray` and `stroke-dashoffset` are set to its measured length, hiding the stroke entirely.
2. **default** — fills render as authored. No strokes.

CSS:

```css
.logo-svg path{ vector-effect: non-scaling-stroke }
.logo-svg.is-drawing path{
  fill: transparent !important;
  stroke: rgba(255,255,255,.92);
  stroke-width: 1.4;
  stroke-linecap: round;
  stroke-linejoin: round;
}
```

`vector-effect: non-scaling-stroke` keeps the 1.4 px line width visually constant even when the SVG scales to fill its container.

### The entrance animation

Two functions in `index.html`:

```js
let _logoPaths = [], _logoLens = [];

function prepLogoForDraw(){
  const svg = document.querySelector(".logo-svg");
  if(!svg) return;
  _logoPaths = Array.from(svg.querySelectorAll("path"));
  _logoLens = _logoPaths.map(p => {
    try { return p.getTotalLength(); } catch(e){ return 0; }
  });
  svg.classList.add("is-drawing");
  _logoPaths.forEach((p,i)=>{
    p.style.strokeDasharray  = _logoLens[i];
    p.style.strokeDashoffset = _logoLens[i];   // start fully hidden
  });
}

function animateLogoDraw(){
  const svg = document.querySelector(".logo-svg");
  if(!svg || !_logoPaths.length) return gsap.timeline();
  const tl = gsap.timeline();
  tl.to(_logoPaths, {
    strokeDashoffset: 0,
    duration: 2.2,
    stagger: { each: .012, from: "start" },
    ease: "silk"
  });
  tl.add(()=>{
    svg.classList.remove("is-drawing");           // swap back to fills
    _logoPaths.forEach(p => {
      p.style.strokeDasharray  = "";
      p.style.strokeDashoffset = "";
    });
  });
  tl.fromTo(svg, {scale:1.02}, {scale:1, duration:.6, ease:"silk", transformOrigin:"50% 50%"}, "<");
  return tl;
}
```

`prepLogoForDraw()` runs **before** the stage fades in, so the logo container becomes visible already in "drawing" state with strokes hidden. `animateLogoDraw()` is added to the hero entrance timeline so the writing happens as the headline reveals.

The hand-off back to fills is intentional: at the end of the stroke animation, the class flips off and the original colored fills appear all at once. Combined with the soft 1.02 → 1 scale pop, it reads as the lettering "settling in" after being drawn.

## Tradeoffs

**What we gained:**
- DrawSVG entrance (handwriting-style)
- Resolution independence (Retina / print)
- 14× smaller transfer
- Per-path control reachable from JS (any path can flicker, color-shift, mask, etc.)
- A11y: `<title>` / `<desc>` / `aria-label` are now meaningful

**What we lost (slightly):**
- **Hand-painted texture** of the original PNG. The trace is clean; the original had soft gradient edges around the strokes that vtracer simplified into discrete fill layers.
- **Some halo softness** — the multi-layer neon glow in the PNG had subtle alpha falloff that the trace converts into stepped color regions. The CSS `drop-shadow` filters compensate, but a careful eye notices.
- **30 paths means 30 measurements** at boot. Negligible (~1 ms total) but technically heavier than `<img>`.

## Reverting

The original raster is still in `assets/logo.png`. To revert:

```html
<!-- replace the inline <svg id="logo" ...>...</svg> with: -->
<img id="logo" src="assets/logo.png" alt="Ai · The New Sexy"
     style="filter:url(#liquid) drop-shadow(0 0 28px rgba(255,31,113,.55)) drop-shadow(0 0 48px rgba(30,95,255,.25))">
```

…and delete the `prepLogoForDraw()` / `animateLogoDraw()` functions plus their calls in `heroIn()`. Everything else (cursor, manifesto, marquee, drops) is independent of the logo format.
