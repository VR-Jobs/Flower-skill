---
name: flower-engine
description: Author, tune, extend, or debug flowers for the Flower-HUA procedural bloom engine (Studio/components/flower/flowerScene.ts). Use when translating a reference photo into a FlowerConfig, picking an anatomy family or core organ, tuning phyllotaxis/petal-profile/bloom/wind parameters, adding a new structure family or core kind to the engine, rebuilding the offline single-HTML deliverable, or diagnosing a flower that renders wrong (spikes, flipped petals, missing core, ballooned bud). Triggers on "调花", "花朵参数", "FlowerConfig", "anatomy family", "petal profile", "phyllotaxis", "bloom wavefront", "flower-hua 引擎", "加一个新科属", "单文件重新构建".
---

# Flower-HUA engine

Everything needed to make this engine produce a specific flower — the parameter
surface, the 26 anatomy families, the reference-image translation procedure, the
build/verify pipeline, and the failure modes.

Scope: **the engine and its data.** The user-facing product contract (how to talk
to a designer, when to open Studio, what not to expose) lives in the project's
root `SKILL.md` and is not repeated here.

## Where things live

Paths are relative to the project root (this file is at `skills/flower-engine/`).

| What | Path |
|---|---|
| Engine | `Studio/components/flower/flowerScene.ts` (4887 lines) |
| Anatomy V2 (26 families, 25 core kinds) | `Studio/components/flower/flowerAnatomy.ts` |
| `FlowerConfig` shape | `Studio/components/flower/flowerConfig.ts` |
| Petal outline math | `Studio/components/flower/petalProfile.ts` |
| Built-in presets (3) | `Studio/components/flower/presets/` |
| Seed library (36 flowers) | `data/flowers.json` |
| **Live** library the app reads | `~/.codex/flower-hua/flowers.json` |
| Persist a flower | `scripts/upsert_flower.py <config.json>` |
| Start / reuse Studio | `scripts/start_studio.py` → prints URL |
| Offline single file | `tools/single-html/build-flower-hua-html.cjs` |

## Reference files

Read the one you need; do not read all five.

- [references/params.md](references/params.md) — all 43 engine params: default, range,
  what they do visually, and **which ones rebuild geometry vs. are live-draggable
  uniforms**. Read before writing or tuning any `params` block.
- [references/anatomy.md](references/anatomy.md) — the 26 structure families and 25 core
  kinds with their whorl counts, variation curves and core defaults. Read before
  choosing `anatomy.family`.
- [references/translate.md](references/translate.md) — reference image → `FlowerConfig`,
  step by step, with the decision rules and a worked example. Read when starting
  from a photo.
- [references/build.md](references/build.md) — data flow, Studio lifecycle, the
  single-HTML bundler and the three Playwright verifiers. Read when building or
  verifying a deliverable.
- [references/pitfalls.md](references/pitfalls.md) — the failure modes that are already
  encoded in the source as hard-won constraints. Read when something renders wrong,
  **and before changing shader, ramp, layout-basis or bundler code.**

## Authoring loop

1. **Classify** the reference by visible structure → pick `anatomy.family`
   ([anatomy.md](references/anatomy.md)). Never pick by botanical name.
2. **Fingerprint** it: `python3 scripts/fingerprint_reference.py <image>` → goes in
   `reference.fingerprint` so re-runs update instead of duplicating.
3. **Write** the `FlowerConfig` JSON — `id`, `name`, `source: "generated"`,
   `reference`, `anatomy`, `params`, 5-stop `palette`, optional `camera`
   ([params.md](references/params.md), [translate.md](references/translate.md)).
4. **Persist**: `python3 scripts/upsert_flower.py /tmp/flower.json`.
   Writes `~/.codex/flower-hua/flowers.json` atomically, de-duping on
   fingerprint-or-id. Never write flowers into `Studio/` source.
5. **Verify visually**: `python3 scripts/start_studio.py` → open the printed URL.
   The API route is `force-dynamic` + `no-store`, so a reload picks up the new
   flower with no restart.
6. **Iterate on the JSON**, not on the engine. Only touch `flowerScene.ts` when the
   reference needs a structure the 26 families genuinely cannot express.

## Hard limits

- `numPetals` ≤ **150** (`MAX_LAYOUT_PETALS`), ≥ 5 in the GUI (`PETAL_MIN`).
- One shared petal geometry, instanced. Per-whorl differences come from
  `anatomy.variation` (inner→outer lerp), not from a second petal mesh.
- `palette` is exactly 5 RGB stops, **0–1 floats**, ordered outer rim → inner core.
  `upsert_flower.py` rejects anything else.
- Anatomy is `version: 2` only. Any other value is silently dropped to the legacy
  phyllotaxis path (`normalizeFlowerAnatomy` returns `null`).
- `hibiscus-column` is pinned to 5 petals and `calla-spathe` to 1 regardless of
  Fullness — by design, not a bug.
