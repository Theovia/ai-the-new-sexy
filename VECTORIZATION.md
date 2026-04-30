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

## Tuned command

```bash
cd assets/
vtracer \
  --input logo.png \
  --output logo.svg \
  --mode spline \
  --colormode color \
  --filter_speckle 16 \
  --color_precision 3 \
  --gradient_step 64 \
  --corner_threshold 60 \
  --segment_length 8 \
  --splice_threshold 60 \
  --path_precision 1 \
  --hierarchical cutout
```

**Why these flags:**

| Flag | Default | Tuned | Reason |
|------|---------|-------|--------|
| `--mode spline` | spline | spline | curves, not polygons — preserves handwriting feel |
| `--filter_speckle 16` | 4 | 16 (max) | drops tiny noise paths from the neon halo glow |
| `--color_precision 3` | 6 | 3 | quantize to fewer color buckets (we don't need 6-bit precision) |
| `--gradient_step 64` | 16 | 64 | wider color steps → fewer halo layers |
| `--segment_length 8` | 4 | 8 | longer minimum segments → smoother curves |
| `--splice_threshold 60` | 45 | 60 | stricter angle to splice splines → cleaner joins |
| `--path_precision 1` | 8 | 1 | one decimal in path data → smaller file, no visible diff |
| `--hierarchical cutout` | stacked | cutout | non-stacked clustering — flatter, cleaner topology |

Default settings produced 671 paths and a **1.34 MB SVG** (worse than the PNG). The tuned settings drop to **30 paths · 162 KB**, which is the sweet spot.

## Result

```
input  · logo.png · 2,342,613 bytes (2.34 MB)
output · logo.svg ·   162,881 bytes (163 KB)
paths  · 30
fills  · 17 unique colors (multi-layer neon halo + core strokes)
size   · 14.4× reduction
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
