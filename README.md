# flower-engine

An Agent Skill for the **Flower-HUA** procedural bloom engine — the parameter
surface, the anatomy catalogue, the reference-image translation procedure, the
build pipeline and the failure modes, distilled from the engine source.

It exists because the knowledge was only readable as 4887 lines of
`flowerScene.ts` plus 761 lines of `flowerAnatomy.ts`. This is that, as tables.

## What's in it

| File | Contents |
|---|---|
| `SKILL.md` | Trigger description, code map, the 6-step authoring loop, hard limits |
| `references/params.md` | All **43 engine parameters** — default, range, visual effect, and the rebuild / bake / uniform cost class that decides which are safe to drag live |
| `references/anatomy.md` | The **26 structure families × 25 core organ kinds** — whorl counts, variation curves, core defaults, and the double clamp on `core.count` |
| `references/translate.md` | Reference image → `FlowerConfig`, plus the authoring conventions measured across the 36 shipped flowers, and one worked example |
| `references/build.md` | The three data stores, Studio lifecycle, export pipeline, single-HTML bundler, three Playwright verifiers |
| `references/pitfalls.md` | ~20 failure modes already encoded as constraints in the engine source |

Every figure was verified against the source rather than recalled: all 43
parameter defaults, all 26 family rows (whorls, core kind, core numbers, the 7
variation values each), and every `count` ceiling.

## Install

Claude Code:

```bash
git clone https://github.com/VR-Jobs/Flower-skill.git
ln -s "$(pwd)/Flower-skill" ~/.claude/skills/flower-engine
```

Codex / other Agent Skills hosts read `~/.agents/skills` instead — link it there.

## It documents a codebase it does not contain

Paths in these files (`Studio/components/flower/flowerScene.ts`,
`data/flowers.json`, `scripts/upsert_flower.py`, `tools/single-html/`) are
relative to the **Flower-HUA project root**, not to this repository. The Skill is
a reading and authoring guide for that codebase; check it out alongside, or read
these files on their own as a design record of how the engine works.

## Highlights

A few things the tables make visible that the source does not:

- The engine's parameter defaults encode the demo rose's stylised look. All 36
  production flowers set `flat: false` and dial `waveAmp`, `noiseAmp`, `jitter`,
  `wrapWidth` down to roughly a third of their defaults.
- `goldenAngle` is only 137.5 for spiral flowers. Small exact-count blooms use
  `360 / numPetals` — 6 petals → 60°, 5 → 72°, a 3-fold magnolia → 120°.
- Three different update paths (`buildFlower()`, `bakeRamps()`, a uniform write).
  Change a parameter through the wrong one and nothing happens, silently.
