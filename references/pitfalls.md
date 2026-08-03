# Pitfalls

Each of these is already encoded as a constraint in the source, usually with a
comment explaining what broke. Read the relevant one before touching that code.

## Rendering

**Every petal collapses into a thin spike (iOS Safari).** The width/curl ramp
texture must be `HalfFloatType` (RGBA16F), not 32-bit float. WebGL2 guarantees
RGBA16F is linear-filterable everywhere; 32-bit float linear filtering needs
`OES_texture_float_linear`, absent on iOS Safari, where the vertex-shader sample
then returns 0. `flowerScene.ts:185`.

**Petals vanish when viewed edge-on.** The shell has a floor on half-thickness
(`0.006 + 0.014·smoothstep(0, 0.28, bladeWidth)`). Don't drive it to zero for a
"thinner petal" — thin the profile widths instead. `flowerScene.ts:470`.

**A whole family opens toward the stem.** Positive local Z must face the open cup.
An inverted `crossVectors` order pointed the normal below the bloom, so cup
deformation exposed the petal backs — lily / tulip / camellia all faced down.
`flowerScene.ts:1729`.

**Changing Fullness flips the flower or makes it face the ground.** Camera-facing
families need a stable design-space basis. It is captured once as
`layoutViewPosition` at scene construction. **Never derive it from the live
OrbitControls camera during a layout rebuild** — that bakes the user's current orbit
into the petals. `flowerScene.ts:576`.

**A closed bud reads as a smooth onion.** Two separate dials: `shellGap` keeps the
nested shells apart while closed, and `wrapWidth`/`wrapCup` control how much a furled
petal inflates. Broad cupped petals (dahlia-like) need both lowered well below the
defaults — the shipped flowers run `wrapWidth` around 0.095 vs. the 0.35 default.

**Flat "paper rosette" silhouette on a disc flower.** Rays start beneath the disc rim
(`discRadius · (0.54 + ring·0.14)`), not at the exact centre, so the disc hides their
roots while the blades stay individually readable. `flowerScene.ts:928`.

## Parameters and data

**A param change has no visible effect.** Three different update paths, and the wrong
one silently does nothing:

- `w0..w4`, `curlBias`, `stemWidth`, `stemEnd`, `petalLen` need `bakeRamps()`.
- Layout params need `buildFlower()`.
- The rest are uniform writes.

`applyPreset()` does all three — use it. Mutating `params` directly does not.

**`applyPreset()` in a slider `onChange` tanks the framerate.** It disposes and
rebuilds the instanced mesh. Use the targeted setters (`setCup`, `setSideCurl`,
`setWind`, …) for live drags; `applyPreset()` is for whole-preset handover.

**Anatomy silently ignored.** `normalizeFlowerAnatomy()` returns `null` for anything
that isn't `version: 2` with a known `family`, and the flower falls back to the legacy
phyllotaxis path — no error, just the wrong shape. Check the family string against
`FLOWER_STRUCTURE_FAMILIES`.

**A palette that looks wrong.** Stops are floats **0–1**, not 0–255, and run rim →
core. `upsert_flower.py` validates the shape (5 stops × 3 numbers) but not the range,
so 0–255 values pass validation and blow out to white.

**Duplicate flowers piling up.** `upsert_flower.py` de-dupes on
`reference.fingerprint` **or** `id`. Omit the fingerprint and rename the id, and you
get a second copy. Only 18 of the 36 shipped flowers carry a fingerprint — that is
the gap, not the rule.

**Fullness produces extra petals on a fixed-count family.** By design:
`hibiscus-column` is pinned to 5 and `calla-spathe` to 1 regardless of `numPetals`.
`flowerScene.ts:2023`.

**Petals spiral when they should sit evenly.** `goldenAngle` stays 137.5 only for
spiral families. A 5- or 6-petal flower needs `360 / numPetals` — see
[translate.md](translate.md).

## Studio and camera

**Wheel zoom does nothing.** `controls.enableZoom = false` by default: the wheel is
reserved for the page's scroll story. Studio re-enables it
(`setWheelZoomEnabled(true)`); elsewhere use `zoomBy()` behind a modifier key.

**Cannot look at the flower from below.** `minPolarAngle 0.04`, `maxPolarAngle 0.49π`,
distance clamped 1.25–6.5. Deliberate — an under-view makes every bloom read as
downward-facing and exposes the petal undersides.

**Scroll rotation fights the orbit.** Scroll-driven rotation lives on `flowerGroup`,
never on the camera, so OrbitControls and ScrollTrigger don't contend for the same
transform. `flowerScene.ts:606`.

**Exported PNGs come out blank.** `preserveDrawingBuffer` is off unless the scene was
created with `exportSurface: true` — it's a per-frame cost every non-Studio page
shouldn't pay. `flowerScene.ts:566`.

**Exported video stutters or drifts from the preview.** The live loop must be paused
first: `pauseLoop()` → `renderFrame(t, bloom)` per frame → `resumeLoop()`. Realtime
`MediaRecorder` capture is only used for the solid-background MP4 path; transparent
exports are deterministic PNG frames because H.264 cannot carry alpha.

## Build and tooling

**`node_modules/next` not found by the bundler.** The single-HTML builder borrows
Next's vendored webpack. Run `prepare_studio.py` first.

**The rebuilt HTML doesn't contain the flower you just saved.**
`build-flower-hua-html.cjs` reads `data/flowers.json` (the seed), not
`~/.codex/flower-hua/flowers.json` (the live store). Copy it over or set
`FLOWER_HUA_STORE_FILE`.

**AR is broken in the single file / it fetches from the network.** Two shims make
`file://` work, and both are easy to lose: the `window.fetch` interception that
answers `/api/flowers` from the inlined store, and `LimitChunkCountPlugin({maxChunks: 1})`
which folds the AR dynamic `import()` into the one script — a `file://` page cannot
fetch a sibling chunk. `verify-ar.cjs` asserts zero external requests; run it.

**Don't turn on minification** in the single-HTML build. Next's vendored minimiser
points at a private Next build-time module.

**The verifiers won't run on another machine.** Each hardcodes an absolute Playwright
path under `~/.cache/codex-runtimes/…` plus a browser binary (Chrome, or Edge for
`verify-layout.cjs`). Edit the constants at the top. They use software WebGL
(`--enable-unsafe-swiftshader`); never add `--disable-gpu`.

**The skill installs where Claude Code can't see it.** `install_skill.py` targets
`~/.agents/skills` (Codex). For Claude Code pass
`--skills-dir ~/.claude/skills`.

## Stale documentation

**Do not under-promise the engine.** The repo's root `SKILL.md` still says to mark
"distinct stamens … irregular orchid-like anatomy" as `approximate` because the
"current one-petal engine cannot model" them. That predates Anatomy V2: there are 25
core-organ builders, `bilateral-orchid` is a shipped family, and 35 of the 36 shipped
flowers use one. Judge each reference against `flowerAnatomy.ts` and
[anatomy.md](anatomy.md), not against that sentence.

(`references/engine-map.md` carried the same stale claim plus a pointer to a
non-existent `Studio/components/flower/generated/` directory. Both were corrected when
this Skill was written — if you find that file describing a one-petal engine again,
something reverted it.)
