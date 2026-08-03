# Data flow, Studio lifecycle, build and verification

## Where a flower actually lives

Three stores, easy to confuse:

| Store | Path | Role |
|---|---|---|
| **Presets** | `Studio/components/flower/presets/*.ts` | 3 built-ins compiled into the app. Never add user flowers here. |
| **Seed** | `data/flowers.json` | 36-flower snapshot shipped with the repo. Copied to the live store on first prepare, **only if the live store does not exist**. |
| **Live** | `~/.codex/flower-hua/flowers.json` | What the running app reads. `upsert_flower.py` writes here. |

`CODEX_HOME` overrides `~/.codex` for all of them (`prepare_studio.py`,
`upsert_flower.py`, `start_studio.py`, and the API route each resolve it
independently).

Read path: `app/api/flowers/route.ts` → reads the live JSON per request
(`runtime: "nodejs"`, `dynamic: "force-dynamic"`, `Cache-Control: no-store`).
A browser reload picks up a new flower — **no server restart needed.** A missing or
malformed file returns `{flowers: []}` rather than erroring.

Write path: `upsert_flower.py` validates (`id`, `name`, `params` object, 5×3 palette),
forces `source: "generated"`, drops any existing entry matching **`reference.fingerprint`
or `id`**, prepends the new one, and writes via a `.tmp` + `replace()` atomic swap.

## Studio lifecycle

```bash
python3 scripts/prepare_studio.py    # idempotent; npm install only when needed
python3 scripts/start_studio.py      # prints the URL; reuses a running server
python3 scripts/start_studio.py --stop
```

- `prepare_studio.py` requires Node **≥ 20.9**. It fingerprints `package-lock.json`
  into `Studio/.flower-hua-dependencies`; if that stamp matches and
  `node_modules/next` exists, it skips `npm install` entirely.
- `start_studio.py` reads `~/.codex/flower-hua/flower-hua-runtime.json`, reuses the
  recorded server if the URL answers 200 **and** the project path matches, else
  scans ports **3000–3020**, spawns `npm run dev` detached with
  `start_new_session=True`, waits up to 60 s, and logs to
  `~/.codex/flower-hua/flower-hua-studio.log`. That log is the first place to look
  when startup fails.
- The URL always ends in `/studio`. Other routes exist for demos: `/`, `/demo`,
  `/dahlia`, `/glass`, `/hero`.
- `Studio/.claude/launch.json` defines a `dev` preview config on port 3001 with
  `autoPort` — use that instead of Bash if you want the browser pane.

## Export pipeline

`Studio/components/flower/flowerExport.ts` — entirely client-side, no ffmpeg, no
server. Background mode picks the path:

| Background | Output | How |
|---|---|---|
| Solid | **MP4** (falls back to WebM) | realtime `MediaRecorder` on `captureStream()`; probes `avc1.640028 → avc1.42E01E → avc1 → mp4 → vp9 → vp8 → webm` |
| Transparent | **PNG sequence in a store-only ZIP** | deterministic offline render, frame by frame. H.264 cannot carry alpha. |
| either | **PNG still** | one high-res frame |

Determinism relies on the scene API: `pauseLoop()` → `setExportSize(w,h)` →
`renderFrame(t, bloom)` per frame → `restoreSize()` → `resumeLoop()`.
`createFlowerScene` must have been constructed with `exportSurface: true`
(`preserveDrawingBuffer`) or `toBlob`/`readPixels` returns blank frames. The
timeline is always: bloom once over `scene.bloomDuration` (5 s), then live wind sway.

## Single-HTML deliverable

```bash
node tools/single-html/build-flower-hua-html.cjs
```

Writes `双击打开 Flower-HUA.html` (~43 MB) at the repo root. Env overrides:
`FLOWER_HUA_STUDIO_DIR`, `FLOWER_HUA_STORE_FILE`.

It reads flowers from **`data/flowers.json`**, not the live store — so
`upsert_flower.py` alone does not change the deliverable. Copy the live store over
the seed first, or point `FLOWER_HUA_STORE_FILE` at it.

How it works:

1. Borrows Next's vendored webpack from
   `Studio/node_modules/next/dist/compiled/webpack/webpack.js` — so `prepare_studio.py`
   must have run.
2. Entry is `single-html-entry.tsx`, which renders `<StudioCanvas />` directly and
   shims two things a `file://` page cannot do:
   - **`window.fetch`** intercepts exactly `/api/flowers` and answers from the
     inlined `__FLOWER_STORE__`; every other request passes through untouched.
   - **`window.__FLOWER_MEDIAPIPE_ASSETS__`** materialises the base64 wasm + gesture
     model into Blob URLs lazily, only when the user starts AR, so double-click
     startup stays fast.
3. `LimitChunkCountPlugin({maxChunks: 1})` collapses the AR dynamic `import()` into
   the single script — a `file://` page cannot fetch a sibling chunk.
4. `minimize: false` deliberately: Next's vendored minimiser reaches for a private
   Next build-time module.
5. CSS is concatenated from five stylesheets with `@import` lines stripped, so
   startup never waits on a font CDN.
6. `</script` inside the bundle is escaped, then re-checked; the build throws rather
   than emit a broken file. It also throws if more than one JS chunk was emitted or
   if either template placeholder is missing.

## Verifiers

Three headless Playwright checks against the built HTML. They are the regression
suite for the deliverable.

| Script | Asserts |
|---|---|
| `verify-full-studio.cjs` | ≥39 flower cards; all 8 Advanced groups present (Colors, Render Style, Petal Geometry, Petal Arrangement, Wind, Natural Detail, Stem & Leaves, Animation Preview); Show Leaves toggles; Export tab has both Image and Video controls; zero page errors |
| `verify-layout.cjs` | title starts with `Flower-HUA`; sheet is flush full-height at the left edge; stage geometry; flower list scrolls; AR does not float over the canvas on QUICK; panel resize and off-canvas collapse |
| `verify-ar.cjs` | `file://` runs as a secure context; `getUserMedia` available; embedded MediaPipe provider present; AR default speed 2.7× / max 3×; AR initialises; **zero external file requests** |

```bash
node tools/single-html/verify-full-studio.cjs
node tools/single-html/verify-layout.cjs
node tools/single-html/verify-ar.cjs
```

**They are not portable.** Each hardcodes an absolute Playwright path under
`~/.cache/codex-runtimes/…` and a browser binary
(`/Applications/Google Chrome.app`, Edge for `verify-layout`). Both exist on this
machine; on any other, edit the two constants at the top of each file. They run with
`--enable-unsafe-swiftshader --use-angle=swiftshader-webgl`, i.e. software WebGL —
expect them to be slow, and never add `--disable-gpu`.

## Installing the skill

`scripts/install_skill.py` links or copies the **repo root** into
`~/.agents/skills/flower-hua` (Codex convention). Claude Code reads
`~/.claude/skills`, so it will not pick this up by default — pass
`--skills-dir ~/.claude/skills`, or symlink this `skills/flower-engine/` folder there
directly.
