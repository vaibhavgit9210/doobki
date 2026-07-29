# doobki — the game

The game the [render tests](../) were auditioning for, built on the
[`third-person/`](../third-person/) renderer: raycast sea plane, one shared
projection, 384×288 dithered buffer, the same sprite rig.

**Live:** https://vaibhavkumar.is-a.dev/doobki/game/

You fall from 6000 units up. Steer left/right to thread the gold rings and dodge
the birds and balloons on the way down, then take the splash. One file, no build
step, no dependencies, no network calls.

## Rules

- **Ring threaded** — +100 × combo. The combo climbs by one per ring (capped ×8)
  and breaks on a miss *or* a hit.
- **Bird / balloon** — knocks you spinning, kills the combo.
- **Clean dive** — finish with zero hits for +500.
- Best score sticks in `localStorage`.

Controls: ← → or A/D; on touch, hold a side of the screen. R restarts mid-run.

## Two things the render test didn't have to solve

**The camera has to look much steeper during play.** At terminal velocity the dive
path runs ~70° below horizontal, and at the render test's cinematic pitch nothing
on that path is inside the frustum at all — you would never see a ring before
hitting it. So during play the pitch opens to ~0.95 rad (sea fills the frame and
the rings ahead stack up like a tube), then eases back to the render test's
horizon framing over the last 700 units, so the entry splash is the exact shot the
look-dev was built around.

**Steering only ever touches `vx`.** Vertical speed and the forward drift stay on
the nominal path, which means `z(y)` is closed-form — every gate laid on that
curve is *guaranteed* to line up in depth when you cross its height, and left/right
is the whole skill. If a hit ever knocked `vy` or `vz`, every gate below it would
silently stop being threadable.

Sound is synthesized WebAudio (wind keyed to airspeed, a chime that climbs with
the combo, a thud, the splash), created on the first input so autoplay policies
never see it.

## Dev hooks

`#t=<seconds>` steps the sim (no input) that many real seconds in, renders one
frame and freezes — rAF is useless under headless Chrome's `--virtual-time-budget`.
`#seed=<n>` pins the course layout (default 20260725).

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --use-angle=swiftshader \
  --window-size=1200,940 --virtual-time-budget=2500 \
  --screenshot=shot.png \
  "file:///path/to/game/index.html#t=8"
```

At `--window-size=1200,940` the canvas is 3× (1152×864) with its top-left at
(24, 38), so `sips -c 864 1152 --cropOffset 38 24 shot.png` trims to the canvas.

## Deploy

Pages serves `gh-pages`, so push both branches — see the [repo README](../README.md#deploy).
The page carries the shared site-analytics beacon.
