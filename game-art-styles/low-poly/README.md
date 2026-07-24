# 3D Game Art Styles · 01 — Low-Poly

Single-file WebGL2 demo of the low-poly 3D art style: a procedurally generated island,
day/night cycle, and toggles that show *why* the style looks the way it does.

Open `index.html` in Brave. No build step, no CDN, no textures — everything is inline,
so it works offline and under a strict CSP.

## Controls

| | |
|---|---|
| drag / scroll | orbit, zoom |
| `space` | pause the day cycle (or use the Time slider) |
| `F` | flat (faceted) ⇄ smooth shading — the core of the style |
| `W` | wireframe over the terrain, shows the tessellation |
| `O` | auto-orbit |
| `R` | new island (new seed) |
| Density slider | terrain resolution, 16–96 quads across |

Dev hook: `index.html#t=0.7&flat=0&wire=1&play=0&spin=0&seed=42` sets state on load
(`t` is 0–1 where 0 = midnight, 0.5 = noon). Handy for screenshots.

## How the low-poly look is produced

- **Faceted normals from derivatives.** The fragment shader takes
  `normalize(cross(dFdx(worldPos), dFdy(worldPos)))`, so every triangle is shaded as one
  hard plane regardless of what the vertex normals say — including animated water, for free.
  Smooth mode instead uses per-vertex analytic normals, which is what `F` compares.
- **Colour in the vertices, no textures.** Terrain faces are coloured by height, slope and a
  coherent noise band (so cliffs read as cliffs instead of grey speckle), with a small
  per-face jitter for the hand-painted feel.
- **Non-indexed geometry.** Vertices are deliberately not shared, which is what keeps colours
  and facets hard-edged.
- **Limited palette + stylised light.** One directional sun, hemisphere ambient, a rim term
  and distance fog, all lerped between three hand-picked palettes (day / dusk / night).
- **Silhouette-first props.** Pines are three stacked 6-sided cones; rocks are jittered
  octahedrons; clouds are clustered octahedrons. ~7–8k triangles for the whole scene.

## Notes

- Needs WebGL 2 (for `dFdx`/`dFdy` and GLSL 300 es); shows a message if unavailable.
- All meshes are wound **clockwise-front** (`gl.frontFace(gl.CW)`), and `tri()` negates the
  computed cross product so default face normals point outward. Get either wrong and
  camera-facing surfaces vanish or go black.
- Live at https://vaibhavkumar.is-a.dev/doobki/game-art-styles/low-poly/ as view 03 of the
  [doobki](../../) render tests. The site-analytics beacon is already in place; paste the same
  block into any new style in this series before deploying it.
