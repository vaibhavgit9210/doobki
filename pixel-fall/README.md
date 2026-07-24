# Skyfall — pixel art

View 01 of the [doobki](../) render tests. A man falls out of a blue sky, hits the
water, and sinks.

**Live:** https://vaibhavkumar.is-a.dev/doobki/pixel-fall/

Roughly 13 seconds, then it loops. He falls in a skydiver's arch — back low, heels
and arms up, face to the sky — corrects himself into a feet-first dive, punches a
crown of foam out of the surface, and then time slows to a quarter speed while he
drifts down into the dark. Click anywhere (or press space) to replay.

One file, no build step, no dependencies, no network calls. It works offline and
under a strict CSP.

## How it works

Everything is drawn by hand into a 200×300 `ImageData` buffer, then blitted to a
visible canvas scaled up by an integer factor with smoothing off. So every pixel is
a real pixel — nothing is a scaled-down photo of a smooth gradient.

- **Gradients are dithered, not smooth.** Sky and water colours come from keyframe
  stop lists (`SKY`, `WATER`), and each channel is then quantised with a 4×4 Bayer
  matrix. That is what produces the banded, chunky look instead of a soft CSS-style
  gradient. Water is keyed on *depth below the surface*, not screen position.
- **Sprites are character grids.** Each pose is an array of strings (`h` hair,
  `s` skin, `t`/`u` shirt, `p` trousers, `b` boots) so poses are editable as text.
- **Rotation is nearest-neighbour, done manually.** `drawSprite` walks the
  destination bounding box and inverse-rotates each pixel back into the sprite grid.
  Rotating pixel art normally looks like mush; doing it this way keeps it blocky,
  which is the point. It also means arbitrary angles are free.
- **Two clocks.** `t` is simulation time (slows to 0.24× on impact) and `rt` is real
  time. Animation physics run on `t`; the camera, the closing fade and the loop reset
  run on `rt`. Mixing these up was the main bug during the build — a camera on
  slow-mo time can never catch a sinking body, and a fade on slow-mo time stretched
  the loop out to nearly 40 seconds.

### The fall

Gravity is 300 px/s² capped at a 560 px/s terminal velocity, from a start height of
1480 px. He hits the water at about 3.4 s.

The arch is drawn side-on as a 13×7 sprite, with the face **above** the hair — that
one detail is what makes him read as looking up at the sky rather than down at the
water. Two frames alternate so the limbs flutter, plus a slow ±0.1 rad roll.

The correction into the dive is a three-stage handoff:

| Time | Pose | Angle |
|---|---|---|
| 0 → 1.85 s | `arch` / `archB` | lazy ±0.1 rad roll |
| 1.85 → 2.30 s | `arch` rotating | 0 → −1.57 rad (head swings up, heels drop) |
| 2.30 → 2.58 s | `strand` | −0.25 → 0 (arm trailing, legs straightening) |
| 2.58 s → impact | `dive` | ~0, small wobble |

Because the arch is a horizontal sprite, rotating it −90° lands it vertical and
head-up, which is why the swap to the upright `strand` sprite is not visible.

The camera trails him at 8/s, and as the sea gets within 430 px it smoothsteps its
framing down so the water rises into shot *before* he lands, instead of arriving
with no warning.

### The splash

On impact: three crown columns (tapered, dithered, rising and collapsing on a sine),
90 droplets thrown up on ballistic arcs, an expanding foam disc, four ripple rings,
and 70 bubbles seeded below the surface. Ripples are drawn correctly for a side view —
each ring is two crests travelling in opposite directions along the wavy surface line.

### The sink

Time eases to 0.24× over 0.45 s. Drag bleeds the 560 px/s entry speed down to a 45 px/s
settle, so the plunge is fast and the drift afterwards is slow. He goes limp, arms
floating above his head, rotating gently. Depth is what carries the scene:

- water darkens through six stops from `#2f93ae` to `#010810` by 520 px down
- god rays fade out entirely by 340 px
- a depth vignette closes in by 460 px
- his own colours are mixed toward the deep, so he ends as a silhouette

He reaches about 420 px down. The fade to black starts 7.5 s after impact and the
loop resets at 9.9 s.

## Dev hooks

`#t=<seconds>` fast-forwards the simulation that many **real** seconds before the
first paint, and prints `rt / t / pose / ang / y / phase` into the hint line.

```
index.html#t=3.5     # the moment of entry
index.html#t=2.2     # mid-correction, arch rotating upright
index.html#t=12      # deep, silhouetted, fading
```

This exists because **`requestAnimationFrame` does not advance under headless
Chrome's `--virtual-time-budget`** — screenshots without it only ever capture frame 1.

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --use-angle=swiftshader \
  --window-size=900,800 --virtual-time-budget=2500 \
  --screenshot=shot.png \
  "file:///path/to/index.html#t=3.5"
```

Add `--dump-dom` and grep the hint line to read state instead of squinting at pixels.

## Deploy

See the [repo README](../README.md#deploy) — Pages serves `gh-pages`, so push both
branches. This page already carries the site-analytics beacon.
