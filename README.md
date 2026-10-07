# VOLTA° — immersive WebGL studio site

A sample "scroll is the camera" site, in **two selectable monolith styles**:

| Style | What scroll does | Best for |
|---|---|---|
| **Corridor** (default) | Flies the camera down a tunnel, straight **through** six solid forms — in one side, out the other. | Launches, brand statements, first-ten-seconds impact. |
| **Orbit** | Holds one sculpted form at the origin and swings the camera around and past it while the form spikes and settles. | Products, portfolios, pages with real reading to do. |
| **Daylight** | Lights the whole scene with the visitor's **real local sun** — correct position, colour and hour — and swings the camera across a horizon. | Restaurants, hotels, venues: anywhere time of day is part of the decision. |

Either way the DOM rides the same perspective rig, so the page itself always
moves in 3D. Switch live with the control in the bottom-left corner, or link
straight to one with `?style=corridor` / `?style=orbit` — the choice is
remembered in `localStorage` and reflected in the URL.

Built as the demo/pitch piece for the web studio. Everything is swappable.

## Run it

```bash
python3 -m http.server 4712 --directory web-studio
```

Then open <http://localhost:4712>. No build step, no npm install, no CDN — three.js
r169 and the postprocessing addons are vendored in `vendor/`, so it also runs from
a file share, a USB stick, or any static host (GitHub Pages, Netlify, S3).

## Files

| File | What's in it |
|---|---|
| `index.html` | All content. Sections are plain HTML — edit copy here. |
| `styles.css` | Design system (colours, type, glass panels, the perspective stage). |
| `main.js` | The corridor scene, the DOM depth rig, and all interaction. |
| `vendor/` | three.js r169 + EffectComposer/UnrealBloom/OutputPass. |

## How the effect works

**The WebGL corridor** (`main.js` §3). Six displaced icosahedron "monoliths" sit
along −Z, with `side: DoubleSide` so their interiors are real places. Gate rings
recycle infinitely down the tunnel; a particle field fills the space. Scroll maps
linearly onto `camera.position.z` across `DEPTH` (400 world units). As the camera
nears a monolith its skin dissolves (`opacity`), its wireframe flares, bloom and
fog spike — so punching through reads as an event.

**Station placement** (`placeStations`). Monoliths are parked in the *gaps between
content sections*, measured from the live layout — so a fly-through always lands on
a transition, never on top of something you were reading. `FIRST`/`MIN_GAP`/`LAST`
keep the opening screen and the contact form clear, whatever the viewport.

**The DOM depth rig** (`driveDOM`). Anything with `data-depth` gets a per-frame
`translate3d(0,0,Zpx) rotateX()` based on its distance from screen centre: far down
the corridor below the fold, flat in a readable dead-zone, then rushing past the
lens above. `data-depth` is a multiplier — `1.35` on the marquee makes it travel
faster than the `0.7` cards, which is where the parallax comes from.

Two rules learned the hard way, both encoded:
- Content visible at load is marked `__arrived` and never pushed away.
- Outgoing panels scale up as they pass the lens, so they fade *fast* and cull —
  a few translucent slabs stacked at 2× will otherwise black out the viewport.

## Embed mode (`?embed=1`)

Add `embed=1` and the site becomes a self-playing preview you can drop in an
`<iframe>` on another page:

    demo/immersive/?embed=1&style=corridor

It hides all chrome (nav, style switch, depth rail, cursor, loader), skips the
loading bar, resolves every reveal — the frame starts mid-page, so nothing may
be waiting to animate in — and then **scrolls the document itself** on a slow
15-second ping-pong. Driving the real scroll (rather than a separate "demo"
animation path) means the WebGL camera and the DOM depth rig stay exactly in
sync with what a real visitor sees.

Two things embed mode must not break, both handled:
- `scroll-behavior` is forced to `auto`. The stylesheet sets `smooth` globally,
  which would make every frame's `scrollTo()` start a competing animation.
- Overflow stays scrollable — only the scrollbar is hidden. `overflow: hidden`
  would stop the self-scroll dead.

Embeds also opt into the phone render budget, since they display small — that's
what makes it safe to run two previews on one page.

## The Daylight style: solar context with zero permissions

`Intl.DateTimeFormat().resolvedOptions().timeZone` hands us the visitor's IANA
timezone with **no prompt, no API key, and nothing leaving the page**. From that:

- their exact local clock (`localHourNow`), and
- a lat/lon from `TZ_COORDS` — about 60 zones people actually use — falling back
  to longitude derived from the UTC offset, good to roughly half an hour.

`sunAngles()` then does standard solar-position maths (declination, equation of
time, hour angle) for elevation and azimuth. `SKY_KEYS` maps elevation onto a
palette; the dome shader just paints a gradient plus a sun disc, with colour
chosen on the CPU so it can be tuned by eye.

Three things that are load-bearing:

- **`TZ_ALIASES`.** Plenty of systems still report legacy names — this machine
  says `Asia/Calcutta`, not `Asia/Kolkata`. Without the alias map India gets lit
  like Kansas. Any new zone added to `TZ_COORDS` should get its legacy alias too.
- **The `--veil` custom property.** Sky brightness drives a bottom-weighted
  scrim so type keeps its contrast at noon as well as midnight. Legibility must
  never depend on the hour. It's flat-zero for the other two styles.
- **The readout panel.** Without something on screen naming the time and phase,
  a visitor just sees "a nice gradient" and the whole idea goes unnoticed. The
  scrubber matters too: at 3am the page would otherwise look broken rather than
  nocturnal. In embed mode the panel hides and the clock cycles a full day per
  loop instead.

## Rebranding it for a client

1. **Name** — search `VOLTA` in `index.html` (nav, loader, footer, `<title>`).
2. **Palette** — the `:root` block in `styles.css`. `--accent` is the gold.
3. **3D mood** — `LOOKS` in `main.js` sets each monolith's radius, hue and spin.
   `ORBIT_KEYS` is the camera path for the Orbit style. `makeEnv()` is the
   lighting: it's a canvas gradient, so change the colour stops and the whole
   scene's reflections change.
   To ship only one style, delete the `.styleswitch` block from `index.html` and
   set the `STYLE` fallback in `main.js` to the one you want.
4. **Intensity** — `DEPTH` (how far you travel), `bloom.strength`, `scene.fog.density`.
5. **Project thumbnails** are generated procedurally (`main.js` §10) from the
   `--a`/`--b` custom properties on each `.proj`. Swap in real `<img>` when you have
   client photography; the canvas exists so the demo ships with zero licensing.

## Things that are demo-only

- The contact form does nothing — it prints a confirmation and clears. Wire it to
  Formspree/Mailchimp/a CRM before launch.
- Project case studies, stats (47/12/98) and the phone number are invented.
- `window.__volta.settle()` is a debug hook that force-renders a settled frame
  (useful when a background tab throttles `requestAnimationFrame`). Harmless, but
  strip it for a client build if you'd rather.

## Accessibility & performance

- `prefers-reduced-motion` disables the perspective stage and all transitions.
- Phones get a reduced budget automatically: lower device-pixel-ratio cap, coarser
  geometry, fewer rings and particles (`MOBILE` in `main.js`).
- Content is real HTML in document order — the 3D is decoration, not structure.
- No horizontal overflow at 375px; verified in-browser.

## Where this is deployed

A copy lives at `everbuilt/demo/immersive/`, served from the Everbuilt Studio
site as the demo behind `work/immersive.html`. That copy is identical to this
one **except** for a "Concept demo — built by Everbuilt Studio" badge, which is
deploy-only so this source stays clean and reusable for client work.

To push changes made here into the deployed copy:

```bash
cd ../everbuilt && ./sync_demo.sh
```

That rsyncs this folder over and re-applies the badge patch automatically. Don't
rsync by hand — you'll silently drop the badge.
