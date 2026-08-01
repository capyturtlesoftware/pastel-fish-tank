# Working on this project

Single-file browser aquarium. Read `DESIGN.md` for architecture and
`PERFORMANCE.md` before touching anything performance-related — several
non-obvious findings there were learned the hard way over six rounds.

## Layout

```
index.html        the entire app (markup + CSS + ~1900 lines JS, one IIFE)
README.md         user-facing
DESIGN.md         architecture and design decisions
PERFORMANCE.md    measurement methodology + findings  ← read before optimising
```

## Hard constraints

- **Keep it a single file.** No build step, no dependencies, no separate JS/CSS.
  This is the property that makes it shareable and it has been chosen
  deliberately over module structure. It's why there's no service worker.
- **Never render below native resolution.** DPR floor is 1.0. Sub-native
  upscaling produces sub-pixel crawl that reads as lag even at a perfect frame
  rate.
- **No `ctx.filter` in a per-frame path.** Measured ~110 ms/frame for a 1.4 px
  full-screen blur in Firefox.
- **No allocations in the frame loop.** No `rgba()`/`shade()` calls, no
  `for...of`, no array literals, no closures in `update()`, `render()`,
  `drawFish()` or any `draw*` function. Precompute colours in `prepPaint()` /
  `bakeThemePaint()`; use indexed `for`. Periodic hitching is GC.
- **Don't assume 60 Hz.** `REFRESH` is measured continuously. Anything
  frame-rate related must scale off `TARGET_MS`.

## Verifying changes

There is no test suite. What has actually caught bugs here:

1. **`node --check`** on the extracted script — catches syntax errors fast:
   ```
   node --check <(sed -n '/^<script>/,/^<\/script>/p' index.html | sed '1d;$d')
   ```
2. **Render it and look at it.** Most real bugs found in this project were
   visual (malformed fins, a stray stroked path, a white blowout at the
   waterline). Screenshot both a light theme and Moonlit — they exercise
   different code paths (`soft-light` vs `lighter`, glow sprites).
3. **Extract pure logic and simulate it in Node.** The quality governor's
   oscillation bug and its convergence across five machine profiles were both
   found this way. Anything with hysteresis or feedback deserves this.
4. **The in-page profiler** (🩺 Diagnose lag) for anything performance-related.
   Do not trust timings taken from a headless/offscreen preview — a hidden page
   doesn't rasterise, and the numbers are meaningless.

## Gotchas that have bitten before

- `ctx.restore()` does **not** restore the current path. Re-issue the path
  before stroking after a clipped block.
- Changing quality calls `resize()` → re-bakes layers → synchronous hitch. Only
  do it when the DPR actually changes.
- Any probe/feedback loop needs exponential backoff on **every** branch, or a
  machine sitting between two tiers oscillates forever.
- Custom (user-designed) fish live outside the count-managed pool. Anything
  iterating fish should use `allFish`.
- The modal preview builds a plain "ghost" fish object — it must go through
  `prepPaint()` or it renders with stale colours.

## Deploying

```sh
git add -A && git commit -m "..." && git push
```

GitHub Pages rebuilds from `main` at root in ~30 s.
Live: https://capyturtlesoftware.github.io/pastel-fish-tank/
