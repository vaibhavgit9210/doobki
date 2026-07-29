# Skyfall — third person

View 04 of the [doobki](../) render tests. The same fall as
[`pixel-fall/`](../pixel-fall/), with the camera moved behind the diver.

**Live:** not deployed yet — open `index.html` locally.

He falls away from you with the sea a long way below, punches through a cloud deck,
goes feet-first into the water, and the camera follows him through the surface and
watches him sink away into the dark. Click anywhere (or press space) to replay.

One file, no build step, no dependencies, no network calls. It works offline and
under a strict CSP.

## Why it isn't just pixel-fall with a different `cam.y`

`pixel-fall/` is a flat side-on view, so the sea can be one horizontal band and the
diver can be a sprite at a screen position. Put the camera *behind* him and neither
holds: the water becomes a plane stretching to a horizon, the splash rings become
ellipses, and the diver's size has to fall off with distance. So this view is built
on an actual camera instead:

- **The sea is a per-pixel raycast.** For every pixel below the horizon, a ray is
  intersected with the plane `y = 0` to get a world position, and the water is shaded
  from a wave height field sampled there. Cost is one wave evaluation per pixel over
  111k pixels — see **Cost** below for what that took. Ray direction depends only on the row, so the hit distance and
  world `z` are computed once per row and only world `x` varies along it.
- **Everything shares one projection.** `project(wx, wy, wz)` — the diver, cloud
  billboards, droplets, bubbles, marine snow. Splash rings and the foam patch aren't
  projected at all: they're tested in *world* space inside the sea shading pass
  (`hypot(wx - ring.x, wz - ring.z)`), so their perspective is free and exact.
- **Same pixel discipline as view 01.** A 384×288 `ImageData` buffer blitted up by an
  integer factor with smoothing off, 4×4 Bayer dither on every gradient, sprites as
  character grids, nearest-neighbour rotate *and scale* so shrinking him as he sinks
  stays blocky.
- **He is drawn at the size he appears.** The poses are authored at ~69×48, and during
  the fall the sprite scale is pinned to exactly 1, so one sprite pixel is one frame
  pixel. Nothing about him is a scaled-up smaller drawing.

## Things that had to be solved

### Framing can't be a lerp

The obvious camera — trail him and ease toward the ideal position — fails. At terminal
velocity a follow with gain *k* settles with a steady-state error of `v/k`: at 560
units/s and k = 6 that's 93 units of lag, so he leaves the bottom of the frame and
never comes back.

Instead the camera height is *solved* every frame. Given a trail distance `back`, a
pitch, and the screen row he should sit on, the height that puts him there is

```
up = back * tan(pitch + atan((FRAME_Y - H/2) / F))
```

so framing is exact at any speed and the pitch is free to swing. He then recedes
underwater by opening `back` up (18 → 98), not by letting the camera lag.

### Why his scale is locked during the fall

`back` is fixed while he falls, but the pitch swings from 0.34 to 0.58 rad as the sea
comes up, and that alone moves his distance from the camera enough to change his
projected size by 16%. Him swelling because the camera tilted is an artifact, not
motion — so the scale is pinned to 1 for the whole fall and `SPRITE_K` is chosen to
make the perspective scale land on 1.0 at the entry framing, so there is no pop when
it takes over. Underwater he shrinks smoothly as `back` opens up, floored at 0.26.

### Wave LOD is what makes or breaks the water

One wave field has to read from 1500 units up and from 20 units up. Each of the seven
sine trains is faded out by distance with `1 / (1 + (cell · 4/λ)²)`, where `cell` is
how many world units one pixel covers at that distance — i.e. hold a train until one
wavelength gets down to about 4 pixels, then fade it. Two mistakes on the way:

- **Too aggressive** and the near field goes glassy exactly when you're closest to it.
  The first pass killed the chop by 90% at one unit per pixel, which is 20+ pixels per
  wavelength — nowhere near aliasing.
- **Normalising by the surviving weight** (to keep contrast constant with distance)
  pumps whichever single train survives at distance up to full contrast, and the far
  sea turns into corrugated iron. A fixed divisor is right: the far field *should*
  flatten out.

A very long train (λ ≈ 440) that never fades gives the distant water broad patches of
light and dark, so it doesn't read as tiling. At the other end a λ ≈ 3.7 train only
resolves in the last few metres, which is what keeps the water from going glassy as he
closes on it.

Foam is the one thing *not* dithered on the Bayer grid. At this pixel size a 4×4
threshold reads as a halftone lattice, and being screen-space it would sit still while
the water slid underneath it — so whitecaps, glitter and the impact patch threshold
against a hash of their **world** position instead, and travel with the wave that made
them.

### Cost

The wave field was about 45% of the frame, because six or seven `Math.sin` calls per
pixel over 111k pixels adds up. Along any one row `wz` is constant and `wx` is linear
in x, so each train's phase is `A + B·x` — which means sin/cos can be advanced by a
fixed rotation per pixel instead of being recomputed:

```
s' = s·cos B + c·sin B
c' = c·cos B − s·sin B
```

Two `Math.sin` calls per train per *row* instead of one per pixel. That took 384×288
from ~15 ms/frame to ~6–12 ms depending on machine load, i.e. 2.56× the pixels of the
old 240×180 for roughly the cost the naive version had at 240×180. The recurrence was
checked against the direct formulation over 357k pixels: max difference 2×10⁻¹⁴.

### Screenshots

`requestAnimationFrame` **does** fire under headless Chrome's `--virtual-time-budget`,
but with multi-second `dt` jumps, so a running loop lands nowhere near the requested
time — every early screenshot here was several sim-seconds past its label. So `#t=`
seeks and then *freezes*, repainting the same frame forever. It has to keep repainting
rather than draw once: with no frame pending, the screenshot can be taken before the
canvas layer is ever rasterised, and you get a blank page.

## The shot list

| Time | What |
|---|---|
| 0 → 1.9 s | spread eagle, lazy roll, cloud deck below |
| 1.9 → 2.45 s | pulling in, arms to his sides |
| 2.45 s → impact | feet-first, camera pitching down as the sea fills the frame |
| ≈ 3.5 s | entry: foam patch, three rings, 240 droplets, 145 bubbles |
| +0.1 s | camera punches the surface — brief white curtain, shake |
| after | pitch tips back up so the lit surface stays overhead while he sinks |

Gravity is 300 units/s² capped at 560, from 1500 up. Time eases to 0.24× after impact,
the fade starts 7.5 s later and the loop resets at 9.9 s — same beats as view 01.

One rough edge: for the two or three frames where the camera is a unit or so above the
surface, the visible patch of water is smaller than any wavelength in the field, so it
flattens out. It is immediately followed by the white curtain of going under, so it
does not read in motion.

## Dev hooks

`#t=<seconds>` seeks that many **real** seconds, renders that frame and freezes,
printing `rt / t / pose / h / cam / pit / phase` into the hint line.

```
index.html#t=1.15    # the establishing shot, clouds below
index.html#t=3.45    # last moment before entry
index.html#t=3.56    # rings on the surface, him dimmed underneath
index.html#t=7       # sinking, shafts hanging in the water
```

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --use-angle=swiftshader \
  --window-size=1200,940 --virtual-time-budget=2500 \
  --screenshot=shot.png \
  "file:///path/to/third-person/index.html#t=1.15"
```

At `--window-size=1200,940` the canvas comes out at 3× (1152×864) and is centred, so
`sips -c 864 1152 shot.png --out crop.png` trims the window down to exactly the canvas.

## Deploy

Not wired up yet. When it goes out, see the [repo README](../README.md#deploy) — Pages
serves `gh-pages`, so push both branches. This page already carries the site-analytics
beacon.
