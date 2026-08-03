# Anatomy V2 — 26 families, 25 core kinds

`Studio/components/flower/flowerAnatomy.ts`. This is the layer that makes the
engine more than a phyllotaxis rosette: it selects a **structural placement
renderer** and an optional **core organ** built from real geometry (discs,
filaments, spadices, seed globes, whiskers), not from petals.

> The project's own `references/engine-map.md` predates this file and claims the
> engine has "no separate stamen, disc, lip, sepal, or second petal system."
> That is stale. 35 of the 36 shipped flowers use Anatomy V2.

## Shape

```ts
type FlowerAnatomyV2 = {
  version: 2;                    // anything else → silently ignored (legacy path)
  family: FlowerStructureFamily; // one of the 26 below
  whorls?: number[];             // petals per concentric whorl, max 8 whorls, each 1–150
  floretCount?: number;          // cluster families only, 4–40
  petalsPerFloret?: number;      // cluster families only, 3–6
  variation?: {                  // inner→outer lerp applied to the shared petal
    innerLength?: number; outerLength?: number;   // 0.2–2
    innerWidth?: number;  outerWidth?: number;    // 0.2–2
    innerCup?: number;    outerCup?: number;      // 0.1–2
    wave?: number;                                // 0–2.5
  };
  core?: {
    kind: FlowerCoreKind;
    size?: number;    // 0.005–0.8
    height?: number;  // −0.2–0.8
    count?: number;   // 1–120 (each core kind caps lower internally)
    color?: [number, number, number];
    accent?: [number, number, number];
    secondary?: [number, number, number];
  };
};
```

`normalizeFlowerAnatomy()` fills every omitted field from the family's defaults and
clamps everything to the ranges above. **Omit a field to accept the default** —
that is the intended way to use this. Override only what the reference demands.

`whorls` is a *ratio*, not an absolute count: `scaledWhorls()` redistributes
`params.numPetals` across the whorls proportionally, so Fullness works on any family.

## Choosing a family

Classify by **visible structure**, never by botanical name. Three questions:

1. **Is the centre an organ or more petals?** An organ → pick a family whose core
   kind matches it. More petals → `spiral-rosette` / `ruffled-mass` / `quilled-sphere`.
2. **How many petals in the outermost visible whorl?** 1 (`calla-spathe`),
   3 (`six-whorl`), 5 (`hibiscus-column`, `columbine-spurs`), 6+ (most), 20+
   (`radial-disc`, `metamorphic-dandelion`).
3. **Is it one bloom or many small ones on an axis?** Many → `cluster-florets`,
   `spherical-umbel`, `foxglove-spike`, `snapdragon-spike`, `wisteria-raceme`,
   `bleeding-heart-raceme`.

## The 26 families

`whorls` / `floret` / core figures are the defaults. Variation is written
`length in→out · width in→out · cup in→out · wave`.

### Rosettes and masses — no core organ

| Family | whorls | Core | Variation | Reads as |
|---|---|---|---|---|
| `spiral-rosette` | 8,13,21 | none | 0.52→1.08 · 0.7→1.12 · 1.25→0.82 · 0.75 | Rose. Tight cupped centre opening to broad outer petals. |
| `ruffled-mass` | 8,13,21,34 | none | 0.55→1.0 · 0.8→1.12 · 1.18→0.86 · **1.8** | Carnation, peony. The high `wave` is the whole point. |
| `quilled-sphere` | 13,21,34,55 | `petal-cushion` 0.12 ×13 | 0.62→1.05 · **0.38→0.58** · 1.35→1.0 · 0.45 | Ball dahlia, chrysanthemum. Narrow width + high `sideCurl` = quills. |

### Flat and radial

| Family | whorls | Core | Variation | Reads as |
|---|---|---|---|---|
| `radial-disc` | 21,21 | `disc` 0.28, h 0.055 | 0.9→1.12 · 0.92→1.0 · 0.45→0.28 · 0.6 | Sunflower, daisy, aster. Seeded disc + strap rays. |
| `metamorphic-dandelion` | 32,32 | `seed-globe` 0.3 ×72 | 0.75→1.08 · 0.32→0.46 · 0.36→0.22 · 0.4 | Dandelion, including the pappus seed head. |
| `pincushion-rays` | 12,16 | `pincushion-needles` 0.24 ×72 | 0.38→0.62 · 0.42→0.58 · 0.44→0.28 · 0.28 | Leucospermum. Short rays under a needle cushion. |
| `nocturnal-star` | 12,18 | `star-stamens` 0.24 ×28 | 0.78→**1.42** · 0.42→0.62 · 0.62→0.35 · 0.48 | Night-blooming cereus. Long narrow rays. |
| `recurved-spider` | 6,6 | `filament-crown` 0.2, h 0.62 ×12 | 0.9→**1.38** · 0.32→0.46 · 0.6→0.36 · 1.05 | Spider lily. Reflexed straps + tall filaments. |

### Few broad petals

| Family | whorls | Core | Variation | Reads as |
|---|---|---|---|---|
| `six-whorl` | 3,3 | `stamen-crown` 0.13, h 0.18 ×6 | 0.88→1.12 · 0.92→1.05 · 1.08→0.72 · 0.55 | Lily, tulip, magnolia. The 3+3 tepal arrangement. |
| `broad-whorls` | 3,6,9 | `seedpod` 0.12, h 0.08 | 0.7→1.2 · 0.9→1.18 · 1.18→0.78 · 0.45 | Lotus, camellia, cherry blossom. |
| `hibiscus-column` | 5 | `hibiscus-stamen` 0.07, h 0.58 ×34 | flat 1.0 · flat 1.0 · 0.78 · 0.5 | Hibiscus. **Pinned to 5 petals at any Fullness.** |
| `columbine-spurs` | 5,5 | `spur-crown` 0.13, h 0.22 ×10 | 0.78→1.18 · 0.58→1.18 · 0.92→0.58 · 0.44 | Aquilegia. Backward spurs. |
| `calla-spathe` | 1 | `spadix` 0.075, h 0.54 ×42 | flat 1.0 · flat 1.18 · 1.34 · 0.18 | Calla. **One wrapped bract + spadix.** |
| `bilateral-orchid` | 3,2 | `lip` 0.18, h 0.08 | 0.88→1.18 · 0.82→1.2 · 1.1→0.65 · 0.8 | Orchid, iris. Mirror symmetry, not radial. |
| `bat-wing` | 2,2,2 | `bat-whiskers` 0.18, h 0.68 ×10 | 0.72→1.28 · **1.15→1.72** · 0.52→0.34 · 1.35 | Tacca. Wide dark bracts + long whiskers. |

### Crowns and cones

| Family | whorls | Core | Variation | Reads as |
|---|---|---|---|---|
| `passion-corona` | 10,10 | `corona` 0.28, h 0.12 ×48 | 0.72→1.08 · 0.72→1.02 · 0.48→0.34 · 0.58 | Passionflower. The filament corona is the subject. |
| `protea-crown` | 8,10,12 | `protea-cone` 0.24, h 0.34 ×56 | 0.58→1.18 · 0.64→1.18 · 1.18→0.82 · 0.42 | King protea. Stiff bracts around a cone. |
| `torch-ginger` | 5,7,9,11 | `torch-cone` 0.18, h 0.5 ×32 | 0.5→0.92 · 0.88→**1.42** · 1.28→0.9 · 0.42 | Etlingera. Waxy bracts stacked on a cone. |
| `bird-fan` | 2,3,2 | `bird-beak` 0.22, h 0.12 ×3 | 0.72→**1.42** · 0.48→0.82 · 0.5→0.28 · 0.35 | Strelitzia. Asymmetric fan out of a beak. |

### Clusters, spikes and pendants

| Family | whorls / floret | Core | Variation | Reads as |
|---|---|---|---|---|
| `cluster-florets` | 24 florets × 4 | `floret-centers` 0.018 | 0.42→0.58 · 1.12→1.18 · 0.42→0.28 · 0.3 | Hydrangea. Many tiny blooms over a dome. |
| `spherical-umbel` | 24 florets × 5 | `umbel-stalks` 0.018, h 0.42 ×24 | 0.34→0.5 · 0.48→0.66 · 0.36→0.24 · 0.28 | Allium. Florets on radiating stalks. |
| `foxglove-spike` | 5,5,4 | `bell-throats` 0.052, h 0.72 ×14 | 0.48→0.78 · 1.0→**1.42** · 1.38→1.08 · 0.34 | Digitalis. Bells up a vertical spike. |
| `snapdragon-spike` | 6,6,4 | `snapdragon-throats` 0.055, h 0.76 ×16 | 0.44→0.7 · 1.05→1.38 · 1.32→1.02 · 0.56 | Antirrhinum. Lipped throats up a spike. |
| `wisteria-raceme` | 8,8,6 | `wisteria-rachis` 0.035, h 0.88 ×22 | 0.38→0.66 · 0.88→1.16 · 1.15→0.82 · 0.48 | Wisteria. Hanging raceme. |
| `bleeding-heart-raceme` | 6,6 | `heart-chain` 0.09, h 0.62 ×6 | 0.42→0.72 · 1.0→**1.34** · 1.22→0.96 · 0.24 | Dicentra. Hearts along an arch. |
| `pendant-fuchsia` | 4,4 | `pendant-stamens` 0.12, h 0.42 ×8 | 0.75→1.2 · 0.72→1.05 · 1.05→0.72 · 0.72 | Fuchsia. Downward lantern with dangling stamens. |

## The 25 core kinds

Each is a distinct geometry builder in `flowerScene.ts` (~lines 2280–3360). They are
tied to their family's placement renderer — **do not mix a core kind onto an unrelated
family** without reading that builder first; several read `params.radius` and the
petal placements to size themselves.

`none` · `disc` · `stamen-crown` · `seedpod` · `lip` · `floret-centers` ·
`petal-cushion` · `bird-beak` · `corona` · `pendant-stamens` · `filament-crown` ·
`bat-whiskers` · `star-stamens` · `umbel-stalks` · `seed-globe` · `protea-cone` ·
`spadix` · `heart-chain` · `bell-throats` · `spur-crown` · `hibiscus-stamen` ·
`pincushion-needles` · `wisteria-rachis` · `snapdragon-throats` · `torch-cone`

**`core.count` is clamped twice.** `normalizeFlowerAnatomy()` clamps it to 1–120, then
the builder clamps again — usually much lower. Raising `count` past the builder's own
ceiling silently does nothing:

| Core kind | default | ceiling |
|---|---|---|
| `seed-globe`, `pincushion-needles` | 72 | 96 |
| `disc` (seeds), `protea-cone` | 55 / 56 | 80 |
| `corona`, `spadix` | 48 / 42 | 64 |
| `hibiscus-stamen`, `torch-cone` | 34 / 32 | 48 |
| `star-stamens` | 28 | 42 |
| `stamen-crown`, `petal-cushion` | 6 / 13 | 24 |
| `filament-crown` | 12 | 18 |
| `bat-whiskers` | 10 | 14 |
| `pendant-stamens` | 8 | 12 |

`bell-throats` and `snapdragon-throats` share one builder that clamps to 10–22 and
then **halves** it (`max(5, floor(n/2))`) for the non-snapdragon path. `heart-chain`
clamps into pairs. The rest derive their count from geometry rather than `core.count`
— read the builder before tuning them.

## Shipped examples

Read one before authoring in the same family — `data/flowers.json`.

| Family | Shipped flower |
|---|---|
| `spiral-rosette` | Scarlet Rose · 绯红玫瑰 |
| `six-whorl` | Tangerine Lily, Violet Tulip, Jade Magnolia |
| `radial-disc` | Sun Gold Sunflower, Golden Daisy, Lavender Aster |
| `broad-whorls` | Moonlit Lotus, Arctic Camellia, Sakura Cloud |
| `ruffled-mass` | Emerald Carnation, Sunset Peony |
| `quilled-sphere` | Velvet Dahlia, Ember Chrysanthemum |
| `bilateral-orchid` | Sapphire Orchid, Midnight Iris |
| everything else | exactly one each — see the family table for the name |

`Cobalt Ice Bloom` is the only flower with **no** `anatomy` block: it runs the legacy
phyllotaxis path. That path still works; use it only for a plain rosette where no
family fits.
