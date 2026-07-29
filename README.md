# doobki

Render tests — look-development for a game — and now the game itself.

**Live:** https://vaibhavkumar.is-a.dev/doobki/

A man falls out of the sky and sinks, rendered three times — two renderers side-on
and once with the camera behind him — plus a 3D art-style study of the look a real
game would ship in. Views 01–04 are deliberately just visuals: the point was to
find out how far each look could be pushed before committing. View 05 is the
commitment: the third-person renderer with steering, rings to thread, birds and
balloons to dodge, and a score.

Every view is a single HTML file. No build step, no libraries, no CDN, no network
calls; they work offline and under a strict CSP.

| | View | Style | Notes |
|---|---|---|---|
| 01 | [`pixel-fall/`](pixel-fall/) | pixel art | 200×300 hand-plotted pixel buffer, Bayer dither, slow-mo sink — [README](pixel-fall/README.md) |
| 02 | [`freefall/`](freefall/) | smooth vector | the same scene resolution-independent, as a control |
| 03 | [`game-art-styles/low-poly/`](game-art-styles/low-poly/) | low-poly 3D | WebGL2 island, raw GL, faceted normals — [README](game-art-styles/low-poly/README.md) |
| 04 | [`third-person/`](third-person/) | pixel art, from behind | camera over his shoulder, so the sea is a raycast plane with a horizon — [README](third-person/README.md) |
| 05 | [`game/`](game/) | **the game** | steer the fall, thread rings, dodge birds & balloons, combo scoring — [README](game/README.md) |

`index.html` at the root is the hub that links all four, with preview stills in
`previews/`.

## Why two takes on the same scene

`freefall/` is the control. Putting it next to `pixel-fall/` separates what the
composition is doing (the fall, the framing, the slow-motion beat, the darkening
depth) from what the pixel-art constraint is doing (the dithered banding, the chunky
sprite rotation, the 13-frame poses). Anything that reads well in both is structural;
anything that only works in one is style.

`game-art-styles/` is a separate series — one folder per 3D art style, with
`low-poly` as 01.

## Dev hooks

Each view can be frozen at a moment for screenshots, which matters because
**`requestAnimationFrame` is useless under headless Chrome's `--virtual-time-budget`**:
it either never advances or fires with multi-second `dt` jumps, so without a hook a
screenshot lands on frame 1 or several seconds past whatever you asked for.

| View | Hook |
|---|---|
| `pixel-fall/` | `#t=<seconds>` — real seconds; also prints `rt / t / pose / ang / y / phase` |
| `freefall/` | `#seek=<seconds>` — renders one still frame and freezes |
| `low-poly/` | `#t=0.7&flat=0&wire=1&play=0&spin=0&seed=42` — `t` is 0–1, 0.5 = noon |
| `third-person/` | `#t=<seconds>` — real seconds, then freezes; prints `rt / t / pose / h / cam / pit / phase` |
| `game/` | `#t=<seconds>` — input-free run, then freezes; `#seed=<n>` pins the course layout |

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --use-angle=swiftshader \
  --window-size=1100,900 --virtual-time-budget=3000 \
  --screenshot=shot.png \
  "file:///path/to/doobki/pixel-fall/index.html#t=3.15"
```

## Deploy

GitHub Pages serves the `gh-pages` branch, so push both:

```bash
git push origin main && git push origin main:gh-pages
```

Every page carries the shared site-analytics beacon (anonymous load + 60 s presence
heartbeats, no cookies, no raw IPs). It early-returns on `file:` and `localhost`, so
local editing never pollutes the stats — paste the same block into any new view
before deploying it.
