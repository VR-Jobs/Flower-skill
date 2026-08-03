# Reference image → FlowerConfig

The engine is a **visual translator**, not a botanical reconstruction. The target is
"reads as that flower at a glance", and every gap gets recorded, not hidden.

## Procedure

### 1. Look at the image at full resolution

If no reference is reachable, stop and ask for one. Never infer geometry from a
species name — inspect the visible form.

### 2. Answer six questions, in this order

| # | Question | Decides |
|---|---|---|
| 1 | Is the centre an **organ** (disc, stamens, spadix, cone) or just more petals? | `anatomy.family`, `core.kind` |
| 2 | How many petals in the **outermost visible whorl**? | `family`, `whorls`, `numPetals` |
| 3 | One bloom, or many small blooms on an axis? | cluster / spike / raceme families |
| 4 | Petal silhouette: rounded, lance, strap, quill, ruffled? | `w0..w4`, `petalLen`, `waveAmp` |
| 5 | Silhouette from the side: flat, domed, cupped, drooping? | `height`, `outAngle`, `tiltBias`, `curlOpen` |
| 6 | Where does colour sit — rim, midpoint, or core? | `palette` |

Then pick the family from [anatomy.md](anatomy.md) and read the closest shipped
flower in `data/flowers.json` as the starting point.

### 3. Fingerprint

```bash
python3 scripts/fingerprint_reference.py path/to/reference.png
```

Put the hash in `reference.fingerprint` and the filename in `reference.sourceName`.
`upsert_flower.py` de-dupes on fingerprint **or** id, so a second pass on the same
image updates the flower instead of adding a twin. Skip it and you get duplicates
whenever the id changes.

### 4. Write the config

```jsonc
{
  "id": "kebab-case-stable-id",
  "name": "English Name · 中文名",
  "source": "generated",
  "reference": { "sourceName": "ref.png", "fingerprint": "…" },
  "anatomy": { "version": 2, "family": "…", "whorls": [], "core": {} },
  "params": { },
  "palette": [[r,g,b], [r,g,b], [r,g,b], [r,g,b], [r,g,b]],
  "camera": [x, y, z]
}
```

Omit `bloom`, `bloomMax`, `animate`, `stableLayout` — those are session state.

### 5. Persist and look at it

```bash
python3 scripts/upsert_flower.py /tmp/flower.json
python3 scripts/start_studio.py
```

Reload the printed URL. Wrong? Fix the JSON and re-run `upsert_flower.py`. Do not
reach for the engine until the JSON is genuinely out of road.

## Conventions the 36 shipped flowers actually follow

Measured across `data/flowers.json`, not guessed. These differ sharply from the
engine defaults, which encode the demo rose's stylised look — **not** a good baseline
for a photo-derived flower.

| Convention | Evidence |
|---|---|
| **`flat: false` — always.** Lambert + subsurface, never flat tone shading. | 36/36 |
| **`camera` always set.** Typical `[1.7, 5.7, 3.8]` — further back and higher than the `[1, 3.5, 2.0]` default so stem and rim stay in frame. | 36/36 |
| **`outwardPush: true` — always.** | 36/36 |
| **Dial the stylisation down.** Median vs. default: `waveAmp` 0.014 vs 0.035 · `noiseAmp` 0.009 vs 0.045 · `jitter` 0.014 vs 0.04 · `shellGap` 0.05 vs 0.18 · `wrapWidth` 0.095 vs 0.35 · `wrapCup` 0.19 vs 0.5. | median of 36 |
| **Calmer wind.** `windAmp` median 0.055 (default 0.15), `windSpeed` 0.68 (default 1.5). | median of 36 |
| **Flatter tilt ramp, gentler curl focus.** `tiltBias` median 1.25 (default 2.2), `curlBias` 1.5 (default 2.3). | median of 36 |
| **Narrower petals.** `w1`/`w2` medians 0.20/0.29, but the lower quartile runs to 0.05/0.065 for straps and quills. | 36 |
| **Blunter tips.** `w4` median 0.018 vs the 0.002 default — a hairline tip reads as a spike. | 36 |

### The symmetry rule

`goldenAngle` is only 137.5 for spiral flowers. For a small exact petal count, set
**`goldenAngle = 360 / numPetals`** so the petals sit evenly instead of spiralling:

| Flower | `numPetals` | `goldenAngle` |
|---|---|---|
| Tangerine Lily, Violet Tulip, Midnight Iris | 6 | 60 |
| Sapphire Orchid | 5 | 72 |
| Jade Magnolia | 9 | 120 (3-fold) |

Spiral families (`quilled-sphere`, `spiral-rosette`, `radial-disc`,
`spherical-umbel`) keep 137.5 at any count — 128, 144, 146 all work.

### Palette

5 stops, **rim → core**, floats 0–1. Two working patterns:

- **Monochrome depth** (Velvet Dahlia): near-black rim → saturated mid → a shifted
  hue at the core. `[0.08,0.005,0.025] … [0.68,0.03,0.12] → [0.22,0.01,0.18]`.
  The dark rim is what makes petals separate visually.
- **Warm gradient** (Sun Gold Sunflower): deep brown rim → orange → yellow core.
  `[0.18,0.04,0.0] … [1.0,0.94,0.08]`.

Both start dark at the rim. A palette that is bright at both ends flattens the flower.

## Worked example — Sun Gold Sunflower

Reference: flat yellow bloom, one ring of straps, big brown seeded disc.

| Observation | Decision |
|---|---|
| Centre is a seeded disc organ | `family: "radial-disc"`, `core.kind: "disc"`, `size 0.33`, `count 80`, brown `[0.18,0.045,0.005]` |
| ~34 straps in two visual ranks | `numPetals: 34`, `whorls: [17,17]` |
| Completely flat | `height: 0.02`, `outAngle: 30`, `cup: 0.08`, `curlOpen: -0.02` |
| Narrow straps, blunt tips | `w: 0.035/0.10/0.125/0.095/0.018`, `petalLen: 1.08` |
| Petals crowd toward the rim | `radius: 0.1` + `radiusBias: 2.4` |
| Barely any ruffle or noise | `waveAmp 0.009`, `noiseAmp 0.014`, `jitter 0.045` |
| Brown → orange → yellow | palette `[0.18,0.04,0]` → `[1,0.94,0.08]` |
| Big flat head needs room | `camera: [1.8, 5.8, 3.9]` |

## Recording approximations

State the gap; never claim a feature the engine does not have. Mark each notable
reference feature as:

- **direct** — the engine expresses it.
- **approximate** — the impression is preserved by other means (colour, density,
  scale) but the mechanism differs.
- **unsupported** — visibly absent.

Common `approximate` cases: petals of two genuinely different shapes on one bloom
(one shared geometry, differentiated only by `anatomy.variation`); per-petal
markings, veining, spots or picotee edges (the 5-stop ramp is radial only); anything
needing a second petal *system* rather than a second whorl.
