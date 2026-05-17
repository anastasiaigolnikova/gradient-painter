# Holistic Generator — Architecture

Single-file web tool that turns photos and videos into graphic posters. All
processing runs in the user's browser (no server, no API costs).

**Repo:** https://github.com/anastasiaigolnikova/gradient-painter
**Public URL:** https://anastasiaigolnikova.github.io/gradient-painter/
**Local file:** `/Users/anastasiaigolnikova/gradient-painter/index.html`

## Files

| File | Purpose |
|---|---|
| `index.html` | Everything: HTML markup, CSS, all JS (filters, animations, export). |
| `mp4-muxer.min.js` | Bundled local copy of `mp4-muxer` (MIT) for WebCodecs MP4 export. No CDN dependency. |
| `ARCHITECTURE.md` | This file. |

---

## PHOTO STYLE (graphic filters)

Selected via `#patternSelect` dropdown. Stored on the photo object as `p.pattern`.

| value | label | What it does | Metadata? |
|---|---|---|---|
| `none` | Original (no filter) | Photo/video as-is | — |
| `dots` | Halftone (dots) | Brightness → black circles of varying radius | ✅ `dotsMetadata` |
| `displace` | Displace (3D) | Scattered particles with depth | ✅ `displaceMetadata` |
| `weave` | Lines | Interwoven thin lines | ❌ no per-element data |
| `shapes` | Abstract | Random triangles / squares / circles | ✅ `shapesMetadata` |
| `ascii` | Signs | ASCII characters of varying density | ✅ `asciiMetadata` |
| `gradient` | Gradient | Vertical strips of varying gray | ✅ `gradientMetadata` |
| `dispersion` | Dispersion | Particles scattered radially | ✅ `dispersionMetadata` |

**Filter sliders:**
- `SIZE` — element size (dot radius, font size, shape size, strip width). Range 3–100.
- `STRENGTH` (`detailLevel`) — density / detail level of the filter.
- `INVERT` — invert brightness mapping.

---

## ANIMATION MODE

Selected via `#animMode` dropdown. Stored in `animMode` global. Drives `applyAnimationEffect()`.

| value | label | Type | Description |
|---|---|---|---|
| `off` | — Off (static) — | — | No animation. Default. |
| `flow` | Flow | pixel | 2D sine displacement of the filter result |
| `liquid` | Liquid (Perlin) | pixel | Perlin-like 2-octave smooth distortion |
| `smoke` | Smoke / Fluid | pixel | Multi-octave large-amplitude drift |
| `glitch` | Glitch / Datamosh | pixel | Horizontal strip displacement + RGB shift bursts |
| `chromatic` | RGB Chromatic Split | pixel | R/B channels diverge horizontally + pulse |
| `pixelate` | Pixelate | pixel | Large mosaic pixels with pulsing size |
| `particle` | Particle Dissolve | pixel | Tile-based explosion + reassembly cycle |
| `pixelhd` | Pixel HD | pixel | Image revealed through black grid cells |
| `aiscan` | AI Scan | pixel | OpenCV-style debug overlay with motion-tracked markers |
| `shuffle` | Shuffle | per-element | **Positions stay fixed (grid preserved). Element ATTRIBUTES (radius/alpha/char/shape/gray/height) lerp toward a partner element's attributes via sin wave.** |
| `reveal` | Build / Reveal | hybrid | Filter builds up element-by-element and back |

**Animation sliders:**
- `SPEED` — overall animation rate. Range 0.2–5. Every animation mode multiplies its time variable by `Math.max(0.2, animSpeed)` (either via the shared `t = tSec * animSpeed` or an explicit factor / `cycleTime / animSpeed`). When adding a new mode, make sure its time term is speed-scaled — using raw `tSec` is a regression.
- `AMPLITUDE` — effect intensity (mode-specific meaning)
- `FREQUENCY` — pulse / waveform frequency (mode-specific)

---

## Compatibility matrix

This is the canonical rule for which animation modes are enabled per combination.

### Animation enabled / disabled

| Animation \ Source | Photo + no filter | Photo + any filter | Video + no filter | Video + any filter |
|---|---|---|---|---|
| Off | ✅ | ✅ | ✅ | ✅ |
| Flow, Liquid, Smoke, Glitch, Chromatic, Pixelate, Particle, Pixel HD | ✅ | ✅ | ✅ | ✅ |
| **AI Scan** | ❌ | ❌ | ✅ | ✅ |
| **Shuffle** | ❌ | ✅ | ❌ | ✅ |
| **Build / Reveal** | ❌ | ✅ | ❌ | ✅ |

**Disable rules** (logic in `syncAnimControlsForPattern`):
- **AI Scan** — disabled when source is **not video** (it relies on frame-to-frame motion).
- **Shuffle** — disabled when **no filter** is selected (it animates filter elements).
- **Build / Reveal** — disabled when **no filter** is selected (it builds filter elements).

When the user changes filter and the currently-selected animation becomes disabled, the dropdown auto-resets to `Off`.

### Quality on each filter (for sanity-check, not enforced)

| Filter \ Animation | Works well | Works (basic) | Avoid |
|---|---|---|---|
| `none` | Flow, Liquid, Smoke, Glitch, Chromatic, Pixelate, Particle, Pixel HD, AI Scan (video) | — | Shuffle, Reveal |
| `dots`, `ascii`, `displace`, `dispersion`, `shapes`, `gradient` | Shuffle, Reveal, all pixel-level | — | — |
| `weave` | All pixel-level | Shuffle (no per-element data → falls back to drawImage) | — |

---

## Photo vs Video

- **Photo:** `p.image = ImageBitmap`, `p.isVideo = false (undefined)`.
- **Video:** `p.video = HTMLVideoElement`, `p.videoUrl = blob URL`, `p.isVideo = true`. Auto-plays muted, looped.
- **Render pipeline** is identical (`renderPhotomaker`); helpers `getPhotoSource(p)` and `getPhotoSize(p)` abstract the difference.
- **Render driver**: `tickFilters` for photos, `videoLoop` for videos. When `_recordingActive`, both yield to a dedicated record loop.

---

## Export pipeline (video download)

| Layer | What | Why |
|---|---|---|
| Codec | H.264 High Profile (`avc1.640028`) with fallbacks | Mainstream, predictable, plays in QuickTime/Premiere |
| Container | MP4 via **`mp4-muxer`** (bundled locally) | Industry standard, opens in everything |
| Encoder | **WebCodecs `VideoEncoder`** (offline, frame-by-frame) | Avoids MediaRecorder's wall-clock-timestamp bug that caused stuttery exports |
| Frame timing | `i × frameDurUs` (constant-rate timestamps) | Even spacing → smooth playback |
| Frame source | `v.currentTime = i / fps; await seeked` (seek-based) | Render can take arbitrarily long per frame; output is still 30 fps |
| FPS detection | Pre-measure 8 frames via `requestVideoFrameCallback` | Match output fps to source fps (24/30/60) |
| Resolution cap | 1920 px on the longest side | Cap render budget; preserve 4K → 1080p downscale |
| Bitrate | `vW × vH × fps × 0.15`, clamped to [12, 40] Mbps | High-motion procedural animation needs more than YouTube minimum |
| Composite | Background gradient + filter overlay → composite canvas in DOM | Without this, filters with transparent backgrounds (Signs) encode as black |
| Download MIME | `application/octet-stream` | Force Chrome to save instead of opening inline |
| Filename | `gradient-painter-{w}x{h}-{ts}.mp4` | Visibility of recorded resolution |

Fallback path (`exportFullVideo`) using MediaRecorder is kept for browsers without WebCodecs (very old Safari/Firefox).

For photo + animation: legacy 5-second synthetic-time recorder via MediaRecorder (`RECORD VIDEO (5S)` button).

---

## Architectural decisions (don't change without reason)

1. **No external CDN dependencies in runtime.** `mp4-muxer.min.js` is bundled locally.
2. **Offline frame-by-frame export.** WebCodecs is the only API that reliably produces smooth canvas-to-MP4.
3. **Single-file HTML.** Everything in one `index.html` for portability and zero build step.
4. **No telemetry, no analytics, no backend.** All processing is client-side.
5. **GitHub Pages hosting.** Free. No deploy script — `git push origin main` is the deploy.

---

## UI structure

```
┌─ HOLISTIC GENERATOR (title)
│  ┌─ Hex inputs (3 colors)
│  ├─ Color pickers
│  ├─ Mid-position slider
│  ├─ GRAIN checkbox
│  ├─ UPLOAD PHOTO  |  UPLOAD VIDEO  (primary actions, row 1)
│  ├─ UPLOAD SVG | UNDO | RESET  (secondary, row 2)
│  └─ #photomakerSection (shown when a photo/video is loaded):
│     ├─ PHOTO STYLE dropdown
│     ├─ SIZE, STRENGTH sliders
│     ├─ INVERT checkbox
│     ├─ #animationControls (shown when a photo/video is selected):
│     │  ├─ ANIMATION MODE dropdown (disable rules above)
│     │  └─ SPEED, AMPLITUDE, FREQUENCY sliders
│     └─ DOWNLOAD button (single, adaptive label — see Export pipeline)
└─ Brand logo link → holisticbrandlab.com (pinned to sidebar bottom)

**DOWNLOAD button behaviour** (single button, `#download` with `#downloadLabel`):
- Static photo, animation off → `DOWNLOAD PNG` → PNG snapshot.
- Photo + animation on → `DOWNLOAD MP4 (5S)` → 5-second synthetic-time MediaRecorder loop.
- Video loaded (with or without filter / animation) → `DOWNLOAD VIDEO` → full-length WebCodecs export (or MediaRecorder fallback).

The legacy `#exportVideo` button is kept in markup for backwards compatibility but always hidden by `updateExportButton()`. All export paths funnel through `runVideoExport(btn, lbl)`.
```

---

## Shuffle behaviour — critical rule

**True positional swap via reciprocal pairs.** Elements with adjacent metadata indices (0↔1, 2↔3, …) are paired. For grid-based filters, adjacent indices are also adjacent in the visual grid, so the swap travels only one cell.

For each pair:
- Both members share the same animation phase (keyed on the PAIR seed, not the individual element).
- At `t=0` each occupies its own slot; at `t=1` they've swapped slots; oscillates via `sin × 0.5 + 0.5`.
- A perpendicular arc offset (`sin(t·π) × segLen × 0.35 × (0.4 + AMP·0.6)`) makes one member swing one way and the other the opposite way — they orbit around each other instead of overlapping in the midpoint.
- Attributes (radius, char, shape, gray, h) lerp along the same `migT`, so the dot that lands in slot B looks like the dot that used to be there.

**Result:** the grid stays fully populated at both extremes of every cycle (each slot is always occupied — sometimes by its original dot, sometimes by its pair partner). Mid-cycle, dots swap places with their adjacent neighbour with a slight arc.

**Why reciprocal pairs:** if A's partner is B but B's partner is C, then A leaves slot A heading to slot B while B leaves slot B heading to slot C → at `t=1`, slot A is empty and slot C is doubly-occupied. Reciprocal pairs guarantee that whatever leaves a slot, the partner is moving INTO that slot.

**Why adjacent-index pairing:** in grid-based filters (`dots`, `gradient`, `shapes`), the metadata array is filled row-by-row, so `meta[i]` and `meta[i+1]` are next to each other in the picture. Local swap = imperceptible grid disturbance. Random pairing would send a forehead dot to a foot dot and back, destroying the silhouette mid-cycle.

**AMPLITUDE** controls the arc magnitude only (not where the swap target is — that's always the immediate pair). Higher AMP → bigger orbital swing around the midpoint.

## Known limitations

- **`weave` + `Shuffle`** — no per-element metadata → falls back to drawImage. Visually no effect.
- **Very long videos with heavy filters** — export can take many minutes (offline frame-by-frame).
- **No mobile-optimized UI** — sidebar is desktop-first. Mobile work is on a separate copy at `/Users/anastasiaigolnikova/gradient-painter-mobile/`.
- **No audio in exported video** — `canvas.captureStream` doesn't carry the source's audio track.
- **Browser support:** WebCodecs requires Chrome 94+, Edge 94+, Firefox 130+, Safari 16.4+. Older browsers get the MediaRecorder fallback path.

---

## When adding a new filter

1. Implement `renderXxx(ctx, imgData, size, detail, w, h, invert)` next to the other render functions.
2. Push per-element data into a new `xxxMetadata = []` array if you want Shuffle to work with it.
3. Add `<option value="xxx">Label</option>` to `#patternSelect`.
4. Add an `else if(pattern==='xxx') renderXxx(...)` branch in `renderPhotomaker`.
5. Add `xxx` to the metadata array dispatch in `applyShuffleAnimation` if applicable.
6. Update this file's compatibility matrix.

## When adding a new animation mode

1. Add `<option value="xxx">Label</option>` to `#animMode`.
2. Implement `effectXxx(...)` in the appropriate place:
   - Pixel-level effect → add to `applyModernEffect` dispatch.
   - Per-element effect → add a branch in `applyShuffleAnimation` or `applyASCIIAnimation`.
3. If it has disable rules, add them to `syncAnimControlsForPattern`.
4. Update this file's compatibility matrix.
