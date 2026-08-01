# 🐠 Pastel Fish Tank

A calm, procedurally-drawn aquarium that runs in a browser. One self-contained
HTML file, no dependencies, no build step, works offline.

**[→ Open the tank](https://capyturtlesoftware.github.io/pastel-fish-tank/)**

## What's in it

- **Procedural fish.** No sprites or images. Each fish is drawn from a flexing
  spine — a travelling sine wave runs nose-to-tail, and the tail inherits the
  spine's tangent so it lags and whips naturally. Five species with distinct
  body profiles, four tail types, dorsal/anal/pectoral fins, countershading,
  gills, markings.
- **Schooling.** Boids separation / alignment / cohesion over a spatial hash,
  weighted per species — tetras shoal tightly, bettas never do.
- **Reactions.** A fast cursor scatters them; a still one draws the curious
  ones in. Switch on the predator and watch the shoals break formation.
- **Light.** Procedural tileable caustics, god rays, depth of field, contact
  shadows on the sand, an animated waterline.
- **Six themes**, or let it follow your clock from dawn to midnight.
- **Design your own fish** — shape, colours, markings, name. It joins the tank
  and persists.
- **Shareable tanks.** Every tank is generated from a seed in the URL, so
  `#s1xizsv2` reproduces exactly the same fish for anyone who opens it.

## Performance

The tank measures the display's real refresh rate and adapts: it sheds effects,
then pixels, then falls back to half refresh (which is judder-free) before it
gives up. There's a **🩺 Diagnose lag** button that runs an automated A/B sweep
across nine configurations and reports where the frame time actually goes —
including unquantised CPU work, because frame intervals alone are rounded to
whole refresh periods and hide anything cheaper than one frame.

## Running it

Open `index.html`. That's it — there's no server, no install, nothing to build.

## Licence

MIT.
