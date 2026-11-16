# Art Viewport

An interactive 2.5D illustrated world in the browser. Visitors pan and zoom around a painted scene, explore
parallax depth layers, and click on areas and objects to learn more or watch embedded content.

Scenes are defined entirely by JSON and image files, so new scenes need no code changes, and the same files
can later be served by an API.

| Route | Scene |
|---|---|
| `/conference` | Conference venue built from a generated illustration (default; `/` redirects here) |
| `/island` | Demo island drawn with procedural placeholder art |

## Quick start

```bash
npm install
npm run dev        # http://localhost:5173/conference
```

| Script | What it does |
|---|---|
| `npm run dev` | Development server with hot reload |
| `npm run build` | Type-check, then build to `dist/` |
| `npm run preview` | Serve the production build locally |
| `npm run typecheck` | Type-check only |

Requires Node 20.19+ or 22.12+ (Vite's minimum); developed on Node 24.

## Stack

| Part | Choice |
|---|---|
| Language and build | TypeScript (strict), Vite |
| Rendering | [PixiJS](https://pixijs.com) 8 (WebGL) |
| Camera: pan, zoom, pinch, momentum | [pixi-viewport](https://github.com/pixijs-userland/pixi-viewport) 7 |
| Image tooling (development only) | [sharp](https://sharp.pixelplumbing.com) |
| Hosting | Vercel (see [Deployment](#deployment)) |

## Features

- **Camera:** drag to pan, scroll or pinch to zoom, with momentum. The camera stays inside the image and can't
  zoom out past its edges.
- **Parallax:** each layer moves at its own speed when panning, and the scene tilts slightly with the mouse.
- **Clickable things:** painted areas (invisible hotspots), separate objects with idle animations (`float`,
  `pulse`, `sway`, `spin`), and live iframes. Hover shows a highlight and tooltip; click zooms in and opens a
  detail drawer.
- **Iframes in the scene:** a web page or video pinned in the world, e.g. on a painted stage screen. It starts
  as a lightweight placeholder, loads on first click into a modal, and returns to the scene still playing when
  the modal closes. It is never reloaded, and an **Unload** button returns it to the placeholder.
- **Markers:** map pins with icons float above clickable items, so they read as clickable without hovering
  (touch screens have no hover). They pop in on load, bob and ripple gently, show labels when zoomed in, and get a
  ✓ once visited (remembered in the browser).
- **HUD:** zoom buttons, Reset View, and a nav bar listing every clickable item. The drawer becomes a bottom
  sheet on phones.

## Project structure

```
public/
  data/<scene>/          scene config (see below)
  img/<scene>/           scene images
src/
  main.ts                routing: picks the scene from the URL
  App.ts                 wires everything together and tears it down
  config/                types.ts (config format), loadConfig.ts (fetch + validation)
  camera/                CameraController: zoom limits, edge clamping, animated moves
  parallax/              ParallaxEngine: per-layer depth movement and mouse tilt
  assets/                AssetLoader, plus procedural placeholder art
  objects/               InteractiveObject: hover, click, animation, highlight
  embeds/                EmbedLayer: live iframes in the world and the modal
  markers/               MarkerLayer: map pins, plus the icon list
  effects/               Spotlight: the dimming hover effect
  ui/                    UIOverlay (HUD, drawer, modal) and UI state
tools/                   cut-sprites.mjs, preview-layout.mjs
art-source/              original artwork the conference scene was made from
vercel.json              build settings and route rewrites
```

## Scene configuration

Each scene is a folder in `public/data/<scene>/`. Every file is validated when the scene loads; a mistake
stops startup with its exact location, for example
`points.json[2].animationType: expected float | pulse | sway | spin, got "wobble"`. The full format, with
comments on every field, is in [`src/config/types.ts`](src/config/types.ts).

All positions and sizes are in **world units**: the scene's coordinate system, set in `world.json`.

### `world.json`

```json
{
  "world": { "width": 2048, "height": 1374 },
  "background": "#2b2440",
  "camera": { "maxScale": 1.6, "buttonZoomStep": 1.4, "animateMs": 900, "ease": "easeInOutCubic" },
  "parallax": { "tiltStrength": 60, "tiltSmoothing": 3.5 },
  "allowedEmbedOrigins": ["self", "https://www.youtube-nocookie.com"]
}
```

- `maxScale`: the furthest zoom-in. The furthest zoom-out is always "image fills the screen".
- `parallax.tiltStrength`: how far the mouse tilt moves layers. Set it to `0` for a single-layer scene.
- `allowedEmbedOrigins`: the only sites iframes may load from; any other `src` is rejected.
- `hoverEffect` (optional, default `outline`): hover feedback for painted areas and flush iframes. `outline`
  frames the item; `spotlight` dims the rest of the scene around it, with a soft edge (also while its drawer is
  open); `none` relies on the markers alone. The conference scene uses `spotlight`.
- `markers.labelZoom` (optional, default `1.5`): marker labels appear once zoomed in this far, relative to the
  fully zoomed-out view.

#### Animation settings (`world.json` → `animation`)

Everything about how animations look and feel. The section and **every field in it are required**; the code
holds no default values, so the scene's config (or the API) is the single source of truth. A missing field fails
with e.g. `world.json.animation.spotlight.feather: missing setting`, and an unknown key (such as a typo) is
rejected too. The list of expected keys is in [`src/config/animationSchema.ts`](src/config/animationSchema.ts).
The values below are the ones both bundled scenes use.

```json
"animation": {
  "idle": {
    "float": { "amplitude": 14, "speed": 1.4, "tilt": 0.035 },
    "pulse": { "amount": 0.035, "speed": 2.2 },
    "sway":  { "angle": 0.04, "speed": 1.1 },
    "spin":  { "speed": 0.45, "hoverBoost": 0.9, "bob": 5 }
  },
  "hover":     { "scale": 0.12, "embedScale": 0.04, "speed": 10 },
  "markers":   { "size": 36, "color": "#2a1d48", "hoverScale": 0.15, "bob": 2.5,
                 "rippleEvery": 3.2, "rippleDuration": 1.6, "introStagger": 0.12, "introDuration": 0.45 },
  "spotlight": { "opacity": 0.5, "color": "#0a081c", "padding": 24, "feather": 70, "cornerRadius": 28, "fadeSpeed": 8 },
  "embed":     { "transitionMs": 450 },
  "camera":    { "friction": 0.93, "wheelSpeed": 0.12, "wheelSmooth": 6, "pinchSpeed": 1.5 }
}
```

| Group | Controls |
|---|---|
| `idle` | The four idle animations chosen per object with `animationType`. `speed` is in radians per second (higher is faster; 1 ≈ one cycle every 6.3 s); distances are world units, angles radians, `pulse.amount` a fraction of size |
| `hover` | How much objects grow on hover (`embedScale` for iframes) and how quickly they react |
| `markers` | Pin head diameter and bob (screen pixels), colour of icon, label and outline, hover growth, ripple timing and the intro pop-in (seconds). `rippleEvery: 0` turns ripples off |
| `spotlight` | Darkness (0–1) and colour of the dimmed scene, space around the item, soft-edge width, corner rounding, fade speed |
| `embed` | Duration of the iframe lift into the modal and back |
| `camera` | Momentum after a drag (`friction`, below 1; closer to 1 glides longer), mouse-wheel zoom amount and smoothing, pinch sensitivity |

### `layers.json`

Back to front. `depth` sets how fast a layer moves when panning: `1` moves with the camera, below 1 is further
away, above 1 is closer. Exactly one layer is `interactive`; clickable things go on it.

```json
[
  { "id": "sky",        "depth": 0.3,  "sprites": [{ "asset": "sky",  "x": 0, "y": 0 }] },
  { "id": "venue",      "depth": 1.0,  "interactive": true, "sprites": [{ "asset": "venue", "x": 0, "y": 0 }] },
  { "id": "foreground", "depth": 1.25, "sprites": [{ "asset": "plants", "x": 0, "y": 0 }] }
]
```

Sprites may also set `anchor`, `alpha`, and `drift` (a slow sideways sway, e.g. for clouds).

### `assets.json`

Every image, by key. `size` is the display size in world units; the file itself may be larger (e.g. @2x).
Relative URLs resolve against the location of `assets.json`.

```json
{
  "venue": { "url": "../../img/conference/venue.png", "size": { "width": 2048, "height": 1374 } },
  "booth-banner": { "url": "../../img/conference/objects/banner@2x.png", "size": { "width": 160, "height": 420 } }
}
```

Instead of `url`, an entry can use `"placeholder": "<name>"` to draw built-in placeholder art (used by the
island scene); see `src/assets/placeholders.ts`.

### `points.json`

Everything clickable. Each entry needs `id`, `title`, `x`, `y`, `zoomLevel` and one of the following kinds.

**Area hotspot.** An invisible rectangle over something painted in the background:

```json
{ "id": "agenda", "title": "Agenda", "x": 24, "y": 384, "anchor": { "x": 0, "y": 0 },
  "area": { "width": 300, "height": 392 }, "zoomLevel": 1.4, "description": "Two days, three tracks…" }
```

**Object.** A separate image with an optional idle animation:

```json
{ "id": "tree", "title": "The Whispering Elder", "x": 820, "y": 1060, "anchor": { "x": 0.5, "y": 1 },
  "asset": "landmark.tree", "animationType": "sway", "zoomLevel": 1.7, "description": "…" }
```

**Iframe.** A web page or video in the scene. Clicking opens it in a modal instead of the drawer:

```json
{ "id": "main-stage", "title": "Main Stage", "x": 830, "y": 200, "anchor": { "x": 0, "y": 0 }, "zoomLevel": 1.5,
  "content": {
    "type": "iframe",
    "src": "https://www.youtube-nocookie.com/embed/VIDEO_ID?autoplay=1&rel=0&playsinline=1",
    "width": 390, "height": 226,
    "appearance": "flush",
    "modalLayout": "resize",
    "sandbox": "allow-scripts allow-same-origin allow-presentation allow-popups allow-popups-to-escape-sandbox",
    "allow": "autoplay; encrypted-media; picture-in-picture; fullscreen",
    "placeholder": { "image": "https://i.ytimg.com/vi/VIDEO_ID/hqdefault.jpg", "label": "Keynote · click to watch" }
  } }
```

| Field | Meaning |
|---|---|
| `x`, `y` + `anchor` | The anchor point's position. `{x: 0, y: 0}` means x/y is the top-left corner; `{x: 0.5, y: 1}` (the default) means bottom-centre |
| `zoomLevel` | Maximum camera zoom when focusing. It is lowered automatically so the whole item fits beside the drawer |
| `appearance` | `card` (default): rounded, with a shadow. `flush`: square, for a screen painted into the artwork |
| `modalLayout` | `scale` (default): enlarge the page as a whole. `resize`: re-lay it out at the modal's size; video players need this to play in HD |
| `placeholder` | Shown until first opened and after **Unload**. Without it, a dark card with the title |
| `marker` | Optional map pin: `{ "icon": "calendar", "offset": [0, 60] }`. The tail sits at the item's top-centre, shifted by `offset` (world units). Icons: `play`, `video`, `presentation`, `mic`, `calendar`, `camera`, `coffee`, `utensils`, `sofa`, `users`, `handshake`, `store`, `id-card`, `ticket`, `gift`, `music`, `star`, `info`, `map-pin` ([Lucide](https://lucide.dev); add more in `src/markers/icons.ts`) |

For YouTube, use the `/embed/` URL (a normal watch URL won't load in an iframe) and add the site to
`allowedEmbedOrigins`.

## Adding a scene

1. Create `public/data/<name>/` with the files above, and put images in `public/img/<name>/`.
2. Add the name to `SCENES` in [`src/main.ts`](src/main.ts).
3. Open `/<name>`.

To load config from an API instead of static files, set `VITE_CONFIG_BASE_URL` to a URL that serves the same
file names. It overrides the route-based folder.

## Artwork export guide

### World size and resolution

Choose the world size first; every position uses it. Export all artwork at **2× the world size** (the conference
scene is 2048 × 1374 world units, so 4096 × 2748 px files) and keep world units in the JSON.

- Keep every file at **4096 px per side or less**. Many phones can't load larger images onto the GPU.
- At 2× export, zooming up to about 2× stays sharp on normal screens.

### Depth layers

Export every depth layer at the **full canvas size, aligned and untrimmed**, so each sits at `x: 0, y: 0`.

| Layer | Suggested depth | Contents | Transparency |
|---|---|---|---|
| Sky / far background | 0.3 | Sky, skyline | Opaque |
| Far scenery | 0.6 | Distant buildings, windows | Transparent above |
| Mid scenery | 0.8 | Back walls, far furniture | Transparent above |
| Main floor | 1.0 | Floor and everything clickable on it | Mostly opaque |
| Foreground | 1.15–1.3 | Plants, railings, banners at the edges | Transparent centre |

- **Paint behind things.** Each layer must be complete where front layers cover it. Parallax reveals what was
  hidden, and any gap shows as a hole.
- **Keep about 5% clear at the edges.** Back layers are enlarged slightly to cover the tilt movement, which
  crops their outer edge.
- **Keep the foreground sparse:** corners and edges only, never over hotspots or screens. Iframes always draw
  above the canvas, so the foreground can't cover them anyway.

### Objects

Anything that animates or moves is its own file: a transparent PNG or WebP, trimmed, at the same 2× scale.
Remove it from the main floor layer and paint the floor behind it. Note its anchor (bottom-centre for things
standing on the floor, centre for floating or spinning things) and the anchor's position in world units.

Clickable things that don't move stay painted in the main floor layer and become `area` hotspots.

### Screens for iframes

A flat, front-facing rectangle with no perspective tilt, painted blank, roughly 16:9 for video. Measure its
inner edge as x, y, width and height in world units.

### Formats and naming

Use WebP (quality about 90, with transparency) for large layers, PNG or WebP for sprites, sRGB colour.

```
public/img/<scene>/
  layer-1-sky@2x.webp
  layer-2-far@2x.webp
  layer-3-main@2x.webp
  layer-4-foreground@2x.webp
  objects/<name>@2x.png
```

## Tools

```bash
# Cut a green-screen sheet of objects into one trimmed, transparent PNG per object
node tools/cut-sprites.mjs art-source/sheet.png public/img/<scene>/objects object

# Draw a layout draft (screen, hotspot areas, paths) over an image to check placements by eye
node tools/preview-layout.mjs art-source/venue.png layout.json preview.png
```

`preview-layout.mjs` takes coordinates in the image's own pixels:

```json
{ "screen": [415, 100, 195, 113],
  "areas": { "agenda": [12, 192, 150, 196] },
  "nodes": { "agenda": [205, 390], "coffee": [195, 474] },
  "edges": [["agenda", "coffee", []]] }
```

## Deployment

The app is a static site: deploy `dist/` anywhere. Scene routes such as `/island` exist only in the browser,
so the host must serve `index.html` for them.

- **Vercel:** handled by [`vercel.json`](vercel.json). It rewrites every path except `/data/`, `/img/` and
  `/assets/` to `index.html`, and also sets the build command and output folder.
- **S3 + CloudFront:** set CloudFront's 403 and 404 error responses to return `/index.html` with status 200.

Keep data and image folders out of the fallback, so a missing config or image file returns a real 404 (and a
clear error) instead of `index.html`.

## Known limitations

- The conference scene is one flat image, so it has no parallax yet. Exporting it as sky, venue and foreground
  layers adds it.
- The conference artwork is 1024 px wide, so zoom is capped at 1.6×. Positions are stored at double that
  size, so a 2048 px image can replace `public/img/conference/venue.png` without changing the config.
- Iframes are HTML elements above the canvas, so they always appear above every layer.
- After the modal closes, an iframe video keeps playing in the scene, with sound, until it's unloaded.
