# Performance: findings and methodology

This file exists because getting this tank smooth took **six rounds**, and four
of them fixed things that were not the problem. The findings below were mostly
counterintuitive, and several contradict what seemed obvious at the time. If you
are about to optimise something here, read this first.

Measurements are from Firefox 151 on macOS, a 1512×898 window on a 120 Hz
ProMotion display, unless noted.

---

## The one rule

**Measure on the target machine. Do not reason about performance from the
outside.**

Four consecutive rounds of optimisation targeted things that measured `+0.0ms`
on the user's hardware. What finally worked was building a profiler *into the
page* so it could measure itself where it actually runs. That's the
**🩺 Diagnose lag** button.

Two environmental traps that make external measurement useless here:

- A **hidden/offscreen page doesn't rasterise.** The dev preview reported the
  same render as both 41.5 ms and 2.28 ms depending on how it was timed. Neither
  was real. `document.hidden === true` means canvas work may never reach the GPU.
- **Per-stage timing misattributes GPU work.** Timing individual draw calls with
  `performance.now()` measures submission, not execution; the cost lands in
  whichever stage happens to flush. Isolated microbenchmarks showed `soft-light`
  as *ten times faster* than `source-over`, which is nonsense.

The reliable technique is **A/B on whole frames**: toggle one feature, re-measure
the entire frame, many samples, compare medians.

---

## Finding 1 — Frame intervals are quantised. They hide everything.

The single most misleading data point in the whole exercise:

```
CONFIG                    FRAME    HITCH/80
baseline (as configured)  33.3ms   40
without caustics          33.3ms   36
without light rays        33.3ms   45
without depth             33.3ms   40
...
no effects at all         22.5ms   2
```

Every individual effect measured **+0.0 ms**, yet removing all of them saved
10.8 ms. Both cannot be true.

**Why:** 33.3 ms is exactly 4 × 8.3 ms. A frame that misses vsync is displayed
at the *next* refresh, so measured intervals can only land on whole multiples of
the refresh period. Any cost smaller than one refresh rounds away completely.

**Fix:** the profiler now also measures **unquantised CPU work** —
`performance.now()` around `update()` + `render()` — and derives all per-feature
deltas from that. Same data, real costs visible:

```
COST OF EACH (median CPU work vs baseline)
  caustics    +2.5ms
  depth       +1.6ms
  plants      +1.0ms
```

**Lesson:** if you A/B on frame intervals alone, you cannot resolve anything
cheaper than one refresh period. On a 120 Hz display that's an 8.3 ms blind spot
— wide enough to hide every effect in this project.

---

## Finding 2 — A healthy frame is vsync-locked, not fast

The adaptive governor originally tried to detect spare headroom with
`median < 12 ms`. That threshold is **unreachable**: when everything is fine,
frames arrive at exactly the refresh period (16.7 ms at 60 Hz, 8.3 ms at 120 Hz)
because they're waiting for vsync. Being fast doesn't make the interval shorter.

The governor could therefore only ever *downgrade*, never recover.

**Fix:** if we're comfortably hitting vsync, **optimistically probe** one level
up and see if it holds. Back off exponentially when a probe fails.

---

## Finding 3 — Don't assume 60 Hz. Measure, and keep measuring.

The refresh rate was hardcoded to 60 Hz. On the 120 Hz display this meant a
*dropped* frame (16.7 ms, half rate) scored as perfect, so the governor was
blind to visible judder while quietly degrading quality for other reasons.

The first fix — measure once at boot — **also failed**, because at boot the page
was already running at 150 ms/frame and even the fast tail looked like 60 Hz.

**Fix:** estimate continuously from a rolling 180-frame window (5th percentile,
snapped to a known rate), and only ever revise *upward* within a session. The
fastest frames a page ever actually achieves are what reveal the panel's rate.

---

## Finding 4 — `ctx.filter` is catastrophic

A 1.4 px blur used for depth of field:

```js
ctx.filter = 'blur(1.4px)';
ctx.drawImage(farCanvas, 0, 0, W, H);   // ~110ms/frame at 5.43MP in Firefox
```

**110 ms of a 151 ms frame** for a barely-visible blur. It's frequently a
software path.

**Fix:** deleted at every level. The far-fish layer renders at 0.34 scale and the
bilinear upscale *is* the defocus — visually near-identical, free.

**Lesson:** never put `ctx.filter` in a per-frame path on a full-screen canvas.

---

## Finding 5 — Blend modes on full-screen fills are expensive

The original light rendering did ~9 blended full-screen passes per frame: five
ray polygons plus four caustic pattern fills, each with `soft-light` or
`lighter`. A blended full-screen **pattern fill** is close to the worst
primitive available in 2D canvas.

**Fix:** rays and caustics render into one half-resolution offscreen with plain
`source-over`, then composite to the main canvas with a **single** blended
`drawImage`. Nine blended full-screen ops → one. Blend modes are further gated
to the High tier only.

---

## Finding 6 — GC stutter came from string building and iterators

Symptom: perfectly smooth, then a hitch, roughly every five seconds. **A
periodic hitch is a garbage collection signature**, not fill rate.

Two sources, both invisible at a glance:

**Colour strings.** Every `rgba()` / `shade()` call allocated ~4 times over —
`hex.replace()`, `parseHex()` returning a fresh array, and a template literal.
The draw path called those ~6× per fish per frame:

> **~86,000 allocations/second at 60 fish.**

None of it was dynamic. All per-fish and per-theme colours are now precomputed
once into `f.pFin70`, `f.pBody`, `TP.gill` etc.

**`for...of` iterators.** Every `for...of` allocates an iterator object. The
worst was the boids neighbour scan — 9 cells × every fish × 60 fps:

> **~43,000 iterator objects/second from one loop.**

All hot loops are now indexed `for`. The spatial grid keeps a flat bucket array
so `rebuildGrid` never iterates a `Map` either.

**Also:** the governor itself allocated and sorted a 45-element array *every
frame*. The stutter-fixing code was a stutter source. It now sorts in place into
a preallocated `Float32Array` and evaluates every 15 frames.

---

## Finding 7 — Sub-native DPR looks like lag but isn't

At one point the pixel budget drove the canvas to **dpr 0.77** — rendering at
1164×691 and upscaling to 1512×898, ~38 % of native on a Retina display.

Frame timing was *perfect*. It still looked laggy, because a non-integer upscale
makes moving edges sub-pixel crawl.

**Fix:** DPR floor of 1.0. Never render below CSS resolution. Shed effects
before sharpness.

**Lesson:** "looks laggy" and "is dropping frames" are different problems.
Ask which one you have before optimising.

---

## Finding 8 — Compositor cost is invisible to canvas profiling

The control panel used `backdrop-filter: blur(14px) saturate(1.3)` over a
268 px × full-height strip sitting on top of a canvas that repaints every frame.
The compositor must re-run that blur every frame and can never cache it.

None of the canvas instrumentation could see this — it isn't canvas work.

**Fix:** the glass effect is opt-in at the High tier only; below that, flat
translucency. The panel is also promoted to its own compositor layer
(`will-change: transform; contain: layout paint style`) so canvas repaints don't
force it to re-rasterise.

**Lesson:** glassmorphism over animated content is a per-frame cost. If you have
a canvas animation and a fancy overlay, suspect the overlay.

---

## Finding 9 — Bigger fish are cheaper than more fish

Per-fish cost is dominated by **draw calls, not pixels** — ~8–12 separate path
fills each, independent of size.

So for constant visual coverage: double the size → 4× the area each → use ¼ as
many → **same total fill, a quarter of the path overhead.**

This is why the defaults are 30 fish at 1.9× rather than 115 at 1.0×. It
reverses above roughly 2×, where fill area starts to dominate; hence the 2.2×
ceiling on the size slider.

---

## Finding 10 — A canvas bug worth remembering

```js
ctx.save();
ctx.clip();
/* draw markings, each with beginPath() */
ctx.restore();
ctx.stroke();   // ← strokes the LAST MARKING, not the body
```

**`ctx.restore()` restores drawing state but NOT the current path.** After a
clipped block, the path still points at whatever was last built. This shipped a
visible dark ellipse across every striped fish. Re-issue the path before
stroking.

---

## Symptom → cause cheat sheet

| Symptom | Most likely cause |
|---|---|
| Hitch on a regular period (~seconds) | GC — hunt per-frame allocations |
| Uniformly slow, every frame | Fill rate — pixels, blend modes, full-screen passes |
| Smooth timing but "looks laggy" | Sub-native DPR upscale, or shimmer/aliasing |
| Slow only at high element counts | Per-element draw calls — LOD or batch |
| Every A/B delta reads `+0.0ms` | Vsync quantisation — measure CPU work instead |
| Fine in isolation, slow in the page | Compositor: backdrop-filter, layer promotion |

## Using the built-in profiler

**🩺 Diagnose lag** holds nine configurations for 80 real frames each (~20 s) and
reports FRAME, WORK, P95 and hitch count for every one, then attributes cost and
gives a verdict. Both the fish-vs-effects split and the GC/compositor
discrimination come out of it. Copy the report — it's designed to be pasted into
a conversation.
