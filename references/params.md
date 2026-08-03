# Engine parameters

All 43 params of the `params` object in `createFlowerScene()`
(`Studio/components/flower/flowerScene.ts:123`). A `FlowerConfig.params` block is a
`Partial<>` of this, applied through `applyPreset()`.

Shipped flowers in `data/flowers.json` set 36 of them. The seven they never set —
`bloom`, `bloomMax`, `animate`, `stableLayout`, `waveFreq`, `stemWidth`, `stemEnd` —
are session/demo state or fine geometry you rarely need to override.

## Cost classes

The single most useful distinction when tuning. `applyPreset()` does all three, so
it is heavy — call it per preset, never per slider frame.

| Class | What happens | Live-draggable? |
|---|---|---|
| **rebuild** | `buildFlower()` — disposes and rebuilds the instanced mesh + layout | No. Batch these. |
| **bake** | `bakeRamps()` — re-bakes the 256×1 RGBA16F width/curl ramp texture | Yes, cheap |
| **uniform** | one uniform write | Yes, free |

## Phyllotaxis — every one is `rebuild`

The layout: where each petal instance sits, how big, how tilted.

| Param | Default | Range | Effect |
|---|---|---|---|
| `numPetals` | 36 | 5–150 (`MAX_LAYOUT_PETALS`) | Density. With V2 anatomy this is a *target* that `scaledWhorls()` distributes across the family's whorls proportionally. |
| `goldenAngle` | 137.5 | 90–180 | Divergence angle. 137.5 is the phyllotaxis default; move it only for a deliberately mechanical look. |
| `stableLayout` | false | bool | Demo-only. Keeps slots allocated for 150 petals while seeds are added/removed. Leave false. |
| `outwardPush` | true | bool | false → concentric rings instead of the spiral: the "paper rosette" look. Leave true unless you want that. |
| `radius` | 0.165 | 0.1–1.5 | Base spread of petal roots. The main size dial. |
| `radiusBias` | 1.15 | 0.3–3 | Exponent on radius across the layout. >1 crowds the centre; <1 pushes petals outward evenly. |
| `height` | 0.155 | 0–1 | Receptacle height — how domed the mound of petal roots is. |
| `heightBias` | 1.2 | 0.3–3 | Exponent on that height. |
| `scaleInner` | 0.46 | 0.1–1 | Size of the innermost petals relative to the outer. Low = tight bud core. |
| `tiltInner` | 0.2 | −0.5–1.5 | Tilt of the innermost petals. Negative leans them inward over the centre. |
| `outAngle` | 68 | 0–120 | Tilt of the outermost petals, in degrees. **>90 makes the rim droop.** 0 = flat disc. |
| `tiltBias` | 2.2 | 0.5–6 | How the tilt ramps from inner to outer. High = flat centre with a sharply flared rim. |

## Single petal

One petal geometry, instanced everywhere. `anatomy.variation` scales it per whorl.

### Outline — the flat silhouette (`bake`)

The width profile is a Catmull-Rom through `[stemWidth, w0, w1, w2, w3, w4]` sampled
at fixed `v` positions `0.225 / 0.41 / 0.595 / 0.78 / 0.965`
(`petalProfile.ts:PETAL_PROFILE_CONTROL_VS`), with a circular cap beyond `w4`.
Baked into the ramp texture's **R** channel.

| Param | Default | Range | Effect |
|---|---|---|---|
| `petalLen` | 0.95 | 0.3–1.5 | Petal length. Also writes `uLength`. |
| `w0` | 0.16 | 0–0.6 | Width near the base. |
| `w1` | 0.28 | 0–0.6 | Widths at 41% / 59% / 78%. **`w1`,`w2` carry the character**: bump them for a rounded petal, flatten for a strap/quill. |
| `w2` | 0.3 | 0–0.6 | ↑ |
| `w3` | 0.2 | 0–0.6 | ↑ |
| `w4` | 0.002 | 0–0.6 | Tip width. Near-0 = pointed lance; 0.1+ = blunt/rounded strap. |
| `stemWidth` | 0.03 | — | Width of the narrow claw below `stemEnd`. Rarely overridden. |
| `stemEnd` | 0.04 | — | Where the claw ends and the blade starts. Rarely overridden. |
| `curlBias` | 2.3 | 0.3–4 | Ramp **G** channel: `curlBias·v^(curlBias−1)`. Remaps *where* the lengthwise bend concentrates. High = bend crammed at the tip. |

### Surface — how the flat petal bends in 3D (`uniform`)

| Param | Default | Range | Effect |
|---|---|---|---|
| `curlOpen` | −0.35 | −1.5–1.0 | Lengthwise curl when open. Negative = recurved/reflexed backward. Positive = cupped forward. |
| `curlClosed` | 1.7 | — (API only) | Lengthwise curl when furled into a bud. Higher = tighter bud. |
| `cup` | 0.4 | 0–1.5 | Transverse spoon — the cross-section arch. |
| `sideCurl` | 0.45 | −3–3 | Edge roll. High positive = the involute "quilled" tube of a ball dahlia. |
| `waveAmp` | 0.035 | 0–0.08 | Edge undulation depth. This is the ruffle dial; the ceiling is low on purpose. |
| `waveFreq` | 11 | — (API only) | Ruffles per petal. |
| `asym` | 0.08 | −0.4–0.4 | Left/right imbalance. A little breaks the plastic symmetry. |
| `propagation` | 1.2 | — (API only) | How the bloom wavefront travels along one petal's length. |

## Bloom wavefront

| Param | Default | Range | Class | Effect |
|---|---|---|---|---|
| `bloom` | 0.0 | 0–1 | uniform | Current openness. Session state — omit from a saved config. |
| `bloomMax` | 0.78 | 0.5–1 | uniform | Ceiling the animation opens to. <1 keeps a flower that never fully flattens. |
| `transition` | 0.35 | 0.05–1 | uniform | Width of the wavefront across the layout. Low = petals open one ring at a time; high = all together. |
| `animate` | false | bool | — | Auto-loop flag. Session state. |
| `shellGap` | 0.18 | 0–0.5 | uniform | Radial gap between the nested petal shells **while closed**. Stops a bud reading as one solid onion. |
| `wrapWidth` | 0.35 | — (API only) | uniform | How much a furled petal inflates in width. Lower it for broad cupped petals that otherwise balloon. |
| `wrapCup` | 0.5 | — (API only) | uniform | Same, for cup depth. |

Bloom duration is fixed at `BLOOM_DURATION_S = 5` seconds; the export timeline reuses it.

## Natural detail and wind

| Param | Default | Range | Class | Effect |
|---|---|---|---|---|
| `jitter` | 0.04 | 0–0.4 | **rebuild** | Per-petal randomisation of the placement. The one detail param that is not free. |
| `noiseAmp` | 0.045 | 0–0.12 | uniform | Surface noise depth — organic waviness. |
| `noiseFreq` | 5.0 | 1–15 | uniform | Surface noise scale. |
| `windAmp` | 0.15 | 0–0.5 | uniform | Sway amplitude. Quick pane's "Movement 0–100" maps to `windAmp = strength/200`. |
| `windSpeed` | 1.5 | 0–4 | uniform | Sway rate. |
| `windHeading` | 35 | 0–360 | uniform | Wind direction in degrees; converted to radians for `uWindHeading`. |

## Render style

| Param | Default | Class | Effect |
|---|---|---|---|
| `flat` | true | uniform | true = flat tone shading (the house style). false = Lambert + subsurface. |

## Palette

Not in `params` — a separate `FlowerConfig.palette`, pushed via `setPalette()`.
Exactly **5 RGB stops, floats in 0–1**, ordered **outer rim → inner core**.
`upsert_flower.py` rejects any other shape.

```jsonc
"palette": [
  [0.05, 0.20, 0.65],   // rim
  [0.15, 0.55, 0.95],
  [0.85, 0.92, 1.00],   // midpoint — where the eye lands
  [1.00, 0.72, 0.00],
  [1.00, 0.25, 0.00]    // core
]
```

Core organs (`anatomy.core.color` / `.accent` / `.secondary`) are coloured
separately and are **not** sampled from this ramp.

## Camera

`FlowerConfig.camera: [x, y, z]`, optional. Default `[1, 3.5, 2.0]`; the Aurora Rose
preset widens to `[1.45, 4.45, 2.9]` so the stem stays in frame beside the design
sheet. OrbitControls clamps distance to 1.25–6.5 and polar angle to 0.04–0.49π, so
a camera below the flower is not reachable.

## Stem

Not in `params` — a separate `stemParams` object, API-only:
`show` (true), `length` (1.8, clamped 0.8–3), `leaves` (true). All `rebuild`.
