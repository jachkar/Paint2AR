# Painting One — AR

Point a phone at a painting and it comes alive: the ocean ripples, the hair and branches
sway, two birds flap on their perch, and a heart beats — all registered 1:1 over the
physical artwork, with real parallax between the layers as you move.

Built with [three.js](https://threejs.org/) and [AR.js](https://ar-js-org.github.io/AR.js-Docs/)
image tracking (NFT). No app, no marker card — the painting itself is the tracking target.
Runs entirely in the mobile browser.

> **Status:** verified end-to-end against synthetic camera frames (tracking, registration,
> layer rendering, animation). Live on-device tracking quality — acquisition distance,
> lighting tolerance, jitter — has not been formally benchmarked.

---

## How it works

The artwork is cut into transparent PNG layers that all share one canvas and one set of UVs,
stacked at different depths and animated on the GPU.

| Layer | Depth | Animation |
|---|---|---|
| `background` | 0.00 | still |
| `ocean` | 0.09 | 12-frame sprite sheet, plus a procedural sun glint and glow |
| `hair` | 0.24 | vertex-shader sway, damped to nothing around the ocean aperture |
| `branches` | 0.45 | vertex-shader sway, anchored at the trunk |
| `bird1` | 0.47 | rocks on its perch in bursts, and rides the branch sway |
| `bird2` | 0.48 | same, offset in time so the two never beat together |
| `heart` | 0.50 | heartbeat scale pulse, rides the branch sway |

Two details that carry most of the visual weight:

- **Sway is procedural, not baked.** A sway is a transform of the whole shape, so it lives in
  a vertex shader — smoother and far lighter than frames. Sprite sheets are reserved for
  motion a transform cannot express, like moving water.
- **Depths are exaggerated.** The layer separation is roughly 8% of the painting's height,
  far more than physical relief. Real depth at true scale reads as nothing through a phone.

Tracking is AR.js NFT: the descriptors in `output/` were generated from the artwork at
300 DPI, so AR.js reports a pose in millimetres and the overlay is scaled to match. The
physical painting does **not** have to be the descriptor's nominal size — NFT is
scale-invariant, so a larger print simply reads as being further away and the overlay stays
registered.

---

## Running it

Any static file server works. The camera requires a **secure context**, so it must be
`https://` or `localhost` — opening `scene.html` as a `file://` URL will not work.

```bash
node .claude/serve.js     # bundled, serves this folder on :8765
```

then open <http://localhost:8765/scene.html>. Any other static server does just as well —
`npx serve .` or `python3 -m http.server 8765` — just use whichever port it reports.

### On a phone

A phone on your LAN hits the dev server over plain HTTP, which the browser will refuse the
camera for. You need real HTTPS. The quickest route is a tunnel:

```bash
cloudflared tunnel --url http://localhost:8765
```

That prints a public `https://…trycloudflare.com` URL. Note it is **public while it runs**,
and ephemeral — you get a different URL each restart. For a stable, private alternative, use
[`mkcert`](https://github.com/FiloSottile/mkcert) to issue a local certificate and serve
HTTPS on your LAN directly.

Expect a few seconds on first load while ~2 MB of tracking descriptors parse in a worker.
The loading overlay is deliberately translucent so you can see the camera working behind it.

### How far to stand

The calibrated vertical FOV is about 43°. For a 1 m-tall painting that puts the whole canvas
in frame at roughly **1.3 m**, with a usable range of about 0.8–3 m. Real phone lenses are
wider than the bundled calibration, so in practice stand slightly closer. Past ~3 m the
target covers under ~100 px of the ~320 px detector canvas and tracking degrades.

---

## Files

```
scene.html                    the AR build — this is the app
scene-parallax-original.html  the original flat-screen build, mouse/touch parallax, no camera
nft-probe.html                dev tool: solves the NFT axis/origin convention empirically
_measure.html                 dev tool: reads alpha bounding boxes and centroids from layer PNGs

*.png                         artwork layers, all full-canvas and sharing one set of UVs
output/painting1.{fset,fset3,iset}   NFT tracking descriptors
ar-data/camera_para.dat       ARToolKit camera calibration (required)
vendor/ar-threex.js           AR.js 3.4.7 three.js build, vendored so the piece runs offline
```

`scene-parallax-original.html` is kept deliberately: it is the same artwork and the same
shaders without the camera, which makes it the fastest way to check an animation change on a
desktop.

The two dev tools are worth keeping — `nft-probe.html` is what produced the measured
placement constants in `scene.html`, and `_measure.html` is how you find the pivot for any
new layer.

---

## Tuning

Everything is a constant near the top of `scene.html`. The ones you are most likely to touch:

| Constant | Default | What it does |
|---|---|---|
| `DEPTH_SCALE` | `0.35` | Layer separation. Up for more pop-out, down for subtlety. |
| `SMOOTH` | `0.35` | Pose lerp. 0 = frozen, 1 = raw tracker output. |
| `LOST_GRACE_MS` | `1500` | How long the art holds its last pose after tracking drops. |
| `BIRD_BURST` / `BIRD_REST` | `0.90` / `1.30` | Seconds of flapping, then stillness. |
| `BIRDS[].amp` / `.freq` / `.delay` | — | Per-bird swing in radians, beats per second, and stagger. |
| `HAIR_SPAN.amp` / `BRANCH_SPAN.amp` | `0.18` / `0.10` | Sway amplitude at the free tips. |
| `NEAR_MM` / `FAR_MM` | `10` / `20000` | Camera frustum, in millimetres. |

Three can be overridden from the query string for on-device tuning without a rebuild:

```
?depth=0.6     override DEPTH_SCALE
?yaw=180       spin about the image normal
?roll=180      turn the artwork upside down in its own plane
```

### Adding a layer

1. Export it as a full-canvas transparent PNG at the same dimensions as the others, so its
   UVs line up with every other layer.
2. Add it to `files` and give it a depth in `depths`.
3. If it needs a pivot, run `_measure.html` to get its alpha bounding box and centroid.

Give any two layers **different** depths. Every layer is a full-canvas quad, and a
transparent fragment still writes depth — two layers at the same z will stamp over each
other's empty area and one will vanish.

### Replacing the artwork

Regenerate the descriptors with the
[NFT Marker Creator](https://carnaux.github.io/NFT-Marker-Creator/), drop the
`.fset`/`.fset3`/`.iset` trio into `output/`, and point `descriptorsUrl` at the new basename.
Use 300 DPI or better; low-DPI targets force the viewer to hold very still. Then update
`NFT_REF` and `ASPECT` to the new image's pixel dimensions.

---

## AR.js NFT notes

Things that cost real debugging time and are not in the documentation. All verified against
AR.js 3.4.7's `three.js` build.

**The NFT worker can silently never start.** AR.js builds its entire NFT pipeline inside an
`arjs-video-loaded` listener that `ArMarkerControls` only registers once
`context.arController` exists. But `ArToolkitSource` dispatches that event *before* calling
your `onReady`. Create the controls any later and the event is already gone — no worker, no
descriptors fetched, no error. `scene.html` re-dispatches the event itself, with a retry.

**Pass descriptor paths as relative URLs.** The worker resolves them as
`self.origin + '/' + path` unless the string passes its own URL regex — and that regex
demands a dot and a TLD, which `http://localhost:8765` has neither of. An absolute URL gets
prefixed anyway and 404s, surfacing only as `Error while intizalizing arController`. Relative
paths work everywhere, but resolve against the **site root**, so this breaks if the page is
served from a subdirectory.

**The projection matrix comes from the worker, not `context.init()`.** The worker solves the
pose against its own ~320 px letterboxed canvas and sends back matching intrinsics on the
`loaded` message, which AR.js writes into `markerRoot.matrix`. Render with the main-thread
projection and the overlay will be misaligned.

**The far plane is 1000 — in millimetres.** ARToolKit hardcodes
`setProjectionNearPlane(0.0001)` / `setProjectionFarPlane(1000)`, and the worker's controller
is out of reach. Under a pattern marker that meant a thousand marker-widths; in NFT's
millimetres it is one metre. Patch the two depth terms of the captured matrix directly.

**Marker space is X–Z with +Y as the face normal** — the same basis as a pattern marker,
because `updateWithModelViewMatrix` post-multiplies `makeRotationX(+π/2)` for every marker
type. The origin is a **corner** of the tracked image, not its centre. Measured with
`nft-probe.html`: `+X` → image right, `−Z` → image up, centre at `(+W/2, 0, −H/2)`.

**Keep `<body>` transparent.** AR.js appends the webcam `<video>` and writes `zIndex="-2"`
onto it inline. A negative-z-index child paints *below* the in-flow block backgrounds of its
stacking context, so an opaque `background` on `<body>` hides the camera feed completely —
a black screen with the artwork floating on it. Put the backdrop on `<html>` instead. Note
that an inline style outranks a normal stylesheet rule, so overriding that `z-index` needs
`!important`.

---

## Requirements

- A browser with WebGL and `getUserMedia`, served over HTTPS or localhost.
- three.js r128 (CDN) and AR.js 3.4.7 (vendored — no build step, no package manager).

AR.js 3.4.7 ships no separate NFT build; `vendor/ar-threex.js` contains both the marker and
NFT paths, and its ARToolKit wasm is inlined, so there are no extra network fetches.

## Known limitations

- The large flat colour fields carry few trackable features, so tracking leans on the
  detailed hair and ocean regions.
- Detection runs on a ~320 px canvas — a hardcoded constant inside the AR.js bundle.
- Seven full-screen transparent layers is the heaviest part of the render. If frames drop,
  cap `setPixelRatio` before anything else.

## Credits

Artwork: **[TODO — artist name]**
Code: **[TODO]**
License: **[TODO — and consider licensing the artwork separately from the code]**

Built on [AR.js](https://github.com/AR-js-org/AR.js) (MIT) and [three.js](https://github.com/mrdoob/three.js) (MIT).
