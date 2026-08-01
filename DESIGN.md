# Design notes

How the tank is built, and why it's built that way. Read `PERFORMANCE.md`
alongside this — several design choices here exist purely because of findings
recorded there.

---

## The single-file constraint

Everything is one `index.html`: markup, CSS, and ~1900 lines of JavaScript. No
build step, no dependencies, no module system.

**Why.** The original ask was "make a webpage", and the first follow-up was
"how do I give this to someone else". A single file you can email, drag onto
Netlify, or double-click offline is worth more here than clean module
boundaries. 88 KB, 27 KB gzipped.

**What it costs.** No service worker, so it is not an installable PWA — a
service worker script must be a separate same-origin file, which would break
the property that makes the file worth having. There's an inline web manifest
via a blob URL, which is enough for iOS "Add to Home Screen" but not for a
Chrome install prompt. If offline install ever matters more than
single-file-ness, that's the trade to revisit.

---

## Fish rendering

### Spine-driven geometry

A fish is not a sprite and not a fixed path. Each is generated per frame from a
**flexing spine**:

```js
spineY = (sp, u, phase, k) => sp.flex * u * u * Math.sin(phase - u * 2.6) * k;
```

`u` runs 0 (nose) → 1 (tail). A travelling sine wave moves head-to-tail with
amplitude growing as `u²`, so the head barely moves and the tail whips. The body
outline is sampled at `SAMP` points above and below the spine and closed into a
ring, then drawn as a quadratic B-spline (`ringPathOn`) which rounds the corners
and gives an organic silhouette.

The tail is **not** animated separately. It's rooted at `u = 0.94` and rotated by
`atan2` of the spine's tangent there, so it inherits the body's motion and lags
naturally. This is why the swimming reads as one connected animal rather than a
body with a flapping triangle attached.

### Unit space

All fish geometry is authored in **unit space** (body length ~1–3 units), then
`ctx.scale(s, s)` maps it to pixels. This isn't cosmetic — it's what makes the
gradient cache possible. A gradient built for a colour in unit space is valid
for every fish of that colour at any size, so `bodyGradient()` can memoise by
`colour|height|contrast` instead of rebuilding per fish per frame.

### Species

Five species plus a predator, each a data object: a 9-sample body profile
interpolated with smoothstep, plus flex stiffness, fin placement, tail type and
a `social` weight. Adding a species is adding an entry to `SPECIES` — no new
drawing code. Weighted by `w` into `SPECIES_BAG` so small common fish outnumber
the showy ones.

### Level of detail

`drawFish` computes a `detail` tier (0/1/2) from the fish's projected pixel size
scaled by `LOD_SCALE`, and skips work accordingly: filaments and markings and
the outline stroke and the gill line and the catchlight all go first. A 12 px
fish costs ~4 draw calls instead of ~12.

`LOD_SCALE` also drops with crowd density (above 55 and 90 fish), because in a
crowded tank the fish are small, overlapping, and individually unexamined.

---

## Simulation

### Schooling

Boids — separation, alignment, cohesion — over a uniform spatial hash (`CELL`
110 px, numeric keys to avoid string allocation). Alignment and cohesion apply
only between **same-species** neighbours; separation applies to everyone.

Per-species `social` weight is what makes it read as an aquarium rather than a
particle system: tetras 1.0, darts 0.9, angels 0.35, puffers 0.12, bettas 0.0.

Measured effect at 150 fish, same seed, 20 s of simulation:

| | schooling off | schooling on |
|---|---|---|
| fish with ≥3 same-species neighbours | 19.2% | **45.0%** |
| mean nearest same-species distance | 69.5 px | 59.2 px |

One non-obvious fix: fish that have found a shoal wander at 25 % strength
(`f.inShoal`). Without that, the random heading kick every few seconds fights
the flocking and the school never tightens — that change alone took clustering
from ~25 % to 45 %.

### States

`calm / hungry / startled / resting` drive a speed multiplier and loosen the
flocking weights. Startled fish drop cohesion entirely, which is what makes a
shoal visibly *burst* when the predator closes in rather than politely drifting.

### Determinism

`mulberry32`, with **a separate stream per fish**, derived as
`makeRng(SEED ^ 0x5bf03635 + i * 2654435761)`.

Per-fish streams matter: with a single shared stream, changing the fish count
would re-roll every fish. Per-index streams mean fish #7 is identical whether
there are 10 fish or 150. Plants and pebbles get their own derived seeds.

The seed lives in the URL fragment (`#s1xizsv2`), so a link reproduces a tank
exactly. Runtime randomness (bubble spawns, turn timers) deliberately uses plain
`Math.random` — reproducing the *tank* matters, reproducing the *motion* doesn't
and would be impossible anyway with variable frame timing.

---

## Rendering pipeline

Per frame, in order:

1. `bgCanvas` blit — water gradient + vignette, baked, rebuilt only on resize
2. **light layer** — rays + caustics into one low-res offscreen, composited once
3. plants (animated strokes)
4. `sandCanvas` blit — **only its band**, not full-screen
5. contact shadows
6. food
7. far fish → low-res layer → composited back up (this upscale *is* the depth blur)
8. near fish
9. bubbles, waterline

Caustics are a procedurally generated **tileable** texture — every term is
2π-periodic in both axes, so the tile seams cleanly and can be a repeating
pattern. Two layers scroll at different scales and directions to read as moving
water.

The light layer collapsing 9 blended full-screen passes into 1 is the single
biggest structural performance decision; see `PERFORMANCE.md`.

---

## Adaptive quality

Three levels (Low / Medium / High) plus Auto. What each level gates was set by
**measurement, not intuition** — an earlier version had caustics gated at the
top tier despite costing ~0 ms, which meant nobody ever saw them.

The governor sheds work in this order:

1. **effects** (blend modes → depth → shadows → rays/caustics)
2. **pixels** (DPR, floored at 1.0 — never below native)
3. **frame rate** (lock to half refresh, which is judder-free because every
   frame is displayed for exactly two refreshes)

Both the quality probe and the half-refresh probe use exponential backoff
(15 s → 45 s → 135 s → …). Without backoff on *both*, a machine sitting exactly
between two tiers oscillates forever — verified in simulation at 54 changes per
400 s before the fix, 8 after.

The DPR floor of 1.0 is load-bearing. Rendering below CSS resolution upscales by
a non-integer factor, and the resulting sub-pixel crawl on moving edges reads as
"lag" even at a flawless frame rate. Shed effects before sharpness.

---

## Deliberate omissions

- **No WebGL.** Canvas 2D keeps it single-file and readable. If per-fish cost
  ever becomes the wall again (it currently isn't, after LOD + bigger fish),
  instanced quads with vertex-shader spine deformation is the escape hatch.
- **No sprite caching of fish.** Tried in thinking, rejected: the body flexes
  continuously, so a sprite atlas would need many phase frames per
  species × colour × size bucket. LOD was cheaper to build and to reason about.
- **No fish sprites/images at all.** Everything is procedural, which is why
  themes can recolour the entire tank instantly.
