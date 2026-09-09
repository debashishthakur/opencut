# OpenCut — Rewrite Progress & Architecture

> Living document. §1–§7 is the state of the world. §9 is the running progress log — append to it as goals land.
>
> Last full audit: 2026-07-14 (rewrite at `bab8af83`; `classic/` at `cf5e79e9`)

---

## 1. This repository is two codebases

Confusing them is the fastest way to get lost.

| | Location | In git? | Size | Status |
|---|---|---|---|---|
| **The rewrite** | repo root (`apps/`, `.moon/`, `Cargo.toml`) | yes, on `main` | ~400 lines of app code | **Scaffold.** Renders `hello world!` |
| **Classic** | `classic/` | **no** — untracked, own `.git` | ~91,000 lines / 685 files | Feature-complete video editor |

`classic/` is the previous OpenCut, archived upstream as `opencut-app/opencut-classic`. It is **not part of the build** — it's checked out here as a reference implementation. It is the answer key.

The README states the target: an Editor API, first-class third-party plugins, one Rust core serving desktop/mobile/browser, an MCP server, headless/batch rendering, and an in-editor scripting tab. `opencut.app` still serves classic; the rewrite is bound for `new.opencut.app`.

---

## 2. The rewrite as it stands

Three apps under [moon](https://moonrepo.dev), with tool versions pinned by proto.

### `apps/web` — TanStack Start on Cloudflare
React 19, TanStack Start + Router (file-based), Vite 8, Tailwind 4, TypeScript 6, deployed to Workers via `@cloudflare/vite-plugin`, routed at `new.opencut.app`.

There is exactly **one route** — [index.tsx](apps/web/src/routes/index.tsx) — rendering `<p>hello world!</p>`. There is **zero editor code**. What exists besides: ~60 shadcn/ui components in [src/components/ui/](apps/web/src/components/ui/), none imported by anything. Pre-installed inventory, not architecture.

### `apps/api` — Elysia on Cloudflare Workers
Three placeholder routes: `GET /` → `{status:"ok"}`, `GET /health`, `POST /echo`. No auth, no DB, no storage bindings. A heartbeat, not a backend.

### `apps/desktop` — GPUI (Rust)
A native window drawing "OpenCut" and "desktop shell scaffold". 51 lines, one dependency (`gpui 0.2.2`).

Worth internalising: **this is not Tauri or Electron.** It's a native Rust UI that cannot render React. It is the forcing function for the whole architecture — anything shared between web and desktop *must* be Rust. (Classic's desktop app was the same GPUI hello-world, so this is a continuation, not a new bet.)

### Tooling
- **proto** pins `moon 2.3.3`, `bun 1.3.11`, `rust 1.97.0` ([.prototools](.prototools)) — identical versions on every dev machine and in CI.
- **moon** auto-discovers `apps/*` and owns `dev`/`build`/`test`/`deploy`. Turbo and the root `package.json` were deliberately deleted in favour of it.
- **bun** sets `minimumReleaseAge = 7d` ([bunfig.toml](bunfig.toml)) — a supply-chain guard.
- **CI** ([.github/workflows/bun-ci.yml](.github/workflows/bun-ci.yml)) runs `moon ci` on Ubuntu, Windows, macOS.

---

## 3. Defects in the scaffold

Verified, and cheapest to fix now.

1. **No linter or formatter is configured at all.** No biome.json, no eslint config, no prettier config anywhere. Yet [.vscode/settings.json](.vscode/settings.json) still instructs the editor to run `source.fixAll.biome`, and [.github/copilot-instructions.md](.github/copilot-instructions.md) is a ~300-line Ultracite/Biome rulebook inherited from classic. **The conventions are documented and enforced by nothing.** Worse, classic had this exact drift already (see §6.3) — we're inheriting a bug in the *docs*, not just the config.
2. **`moon ci` will likely fail on `web:test`** — the task runs `vitest run` and the repo has no test files; Vitest exits non-zero on no tests unless `passWithNoTests` is set.
3. **`public/manifest.json` is still the Create-TanStack-App sample** (`"short_name": "TanStack App"`, referencing `logo192.png`/`logo512.png`, which don't exist).
4. **Dependencies are not installed** — nothing has been run yet.
5. **Dependency bloat before first use:** `recharts`, `embla-carousel`, `input-otp`, `react-day-picker`, `vaul`, `cmdk`, dragged in by shadcn. A video editor needs approximately none of them.
6. **Two path aliases (`#/*` and `@/*`) for the same target.** Pick one now or the codebase uses both forever.
7. **`Cargo.toml` declares only `apps/desktop`**, with a comment anticipating `crates/`. **The shared Rust core has no home yet — this is the most important structural decision still unmade.**
8. `__root.tsx` loads a Google Fonts *Playfair Display* stylesheet while `@fontsource-variable/inter` sits unused in dependencies.

---

## 4. Classic — the reference implementation

Far more sophisticated than a demo. The good ideas here should be carried forward deliberately, not rediscovered.

### 4.1 `EditorCore` — a singleton of 12 managers

[classic/apps/web/src/core/index.ts](classic/apps/web/src/core/index.ts):

```ts
export class EditorCore {
  private static instance: EditorCore | null = null;
  public readonly timeline: TimelineManager;
  public readonly command: CommandManager;
  public readonly playback: PlaybackManager;
  public readonly scenes: ScenesManager;
  public readonly project: ProjectManager;
  public readonly media: MediaManager;
  public readonly renderer: RendererManager;
  public readonly save: SaveManager;
  public readonly audio: AudioManager;
  public readonly selection: SelectionManager;
  public readonly clipboard: ClipboardManager;
  public readonly diagnostics: DiagnosticsManager;
}
```

**This is not a Zustand app.** Zustand v5 is a dependency but owns only UI chrome — panel sizes, `snappingEnabled`, expanded rows. The timeline store says so itself: *"UI state for the timeline. For core logic, use EditorCore instead."* The entire editing domain lives in these managers, each a hand-rolled observable (`Set<() => void>` + `subscribe()` + `notify()`).

React binds via `useSyncExternalStore` in `editor/use-editor.ts`, with a shallow-equality snapshot cache. Note the coarse granularity: `subscribeAll` subscribes to **all nine** observable managers at once, and the selector cache is the only thing preventing re-render storms. A known scaling edge.

### 4.2 Time is an integer. This is the standout idea.

`TICKS_PER_SECOND = 120_000`, defined in Rust (`rust/crates/time/src/media_time.rs`) as `MediaTime(i64)`. 120,000 is not arbitrary — **every broadcast frame rate divides it exactly**: 23.976 → 5005 ticks/frame, 24 → 5000, 25 → 4800, 29.97 → 4004, 30 → 4000, and 48/50/59.94/60/120 all land clean. `FrameRate::ticks_per_frame()` returns `None` when `120000 * den % num != 0` — **non-representable rates are rejected, never approximated.**

FPS is a rational `FrameRate { numerator, denominator }`, so 29.97 is exactly 30000/1001.

On the TS side ([wasm/index.ts](classic/apps/web/src/wasm/index.ts)):

```ts
/** Integer-tick time. Mirrors `MediaTime(i64)` in `rust/crates/time/src/media_time.rs`. */
export type MediaTime = number & { readonly __mediaTime: unique symbol };
```

A branded type: **reading is free** (assignable to `number`), **writing is gated** — a bare `number` cannot become a `MediaTime` except through `roundMediaTime` / `mediaTimeFromSeconds`, which round inside the WASM boundary (half-away-from-zero, matching Rust, normalising `-0`). Float drift across split/trim/retime is eliminated *by the type system*, not by discipline.

**Carry this forward on day one.** Retrofitting it is agony.

### 4.3 The domain model

A project is **multi-scene**. Each scene owns a structured track triple — not a flat list:

```ts
export interface SceneTracks {
  overlay: OverlayTrack[];   // video | text | graphic | effect, z-ordered above main
  main: VideoTrack;          // exactly one, always present
  audio: AudioTrack[];
}
```

`main` being a single privileged track (rather than one of N) is what makes ripple editing and magnetic-timeline behaviour tractable. It shows up everywhere: drop targets, z-order, ripple.

Elements are a discriminated union over a shared base:

```ts
interface BaseTimelineElement {
  id: string; name: string;
  duration: MediaTime; startTime: MediaTime;
  trimStart: MediaTime; trimEnd: MediaTime;
  sourceDuration?: MediaTime;
  animations?: ElementAnimations;
  params: ParamValues;        // <- ALL visual properties live here
}
// → Audio | Video | Image | Text | Sticker | Graphic | Effect
```

**The key design decision: there are no `x`/`y`/`scale`/`opacity` fields on elements.** Everything is a flat, string-keyed `params` bag with dotted keys (`"transform.positionX"`, `"opacity"`, `"blendMode"`). That uniformity is exactly what lets the keyframe system, effects, properties panel, and graph editor all be generic. The cost is type-safety on element properties — a real trade, made deliberately.

Capabilities are derived at the type level, not duck-typed:

```ts
export type MaskableElement = Extract<TimelineElement, { type: (typeof MASKABLE_ELEMENT_TYPES)[number] }>;
```

Keyframes are a full After-Effects-grade model: bezier handles (`leftHandle`/`rightHandle` as `{dt, dv}`), `segmentToNext: "step"|"linear"|"bezier"`, `tangentMode: "auto"|"aligned"|"broken"|"flat"`, and per-channel `extrapolation`. Colours decompose into **linear-RGBA composite channels** so colour keyframes interpolate correctly in linear space.

### 4.4 Rendering — a wgpu compositor, not Canvas2D

I initially misread this; correcting for the record. **The wgpu compositor is the main path.**

```
SceneTracks + MediaAsset[] → buildScene() → RootNode
  → resolveRenderTree()      (decode/rasterize each node at time t)
  → buildFrameDescriptor()   (flatten to a wire-format IR + texture list)
  → wasmCompositor.render()  (Rust/wgpu)
  → canvas mounted directly in the DOM
```

`PreviewCanvas` **appends the compositor's own canvas into the DOM** — *"wgpu renders straight into this element, so there is no intermediate copy."* No blit.

The TS↔Rust contract is a per-frame `FrameDescriptor` (serde, camelCase, via `serde-wasm-bindgen`):

```rust
FrameDescriptor { width, height, clear: {color}, items: Vec<FrameItemDescriptor> }
enum FrameItemDescriptor { Layer(LayerDescriptor), SceneEffect { effect_pass_groups } }
LayerDescriptor {
    texture_id, transform: QuadTransformDescriptor, opacity,
    blend_mode,                       // 17 CSS/PDF modes, incl. non-separable hue/sat/color/luminosity
    effect_pass_groups, mask: Option<{texture_id, feather, inverted}>,
}
```

Deserialisation cost is explicitly instrumented (`wasm.deserialize` perf span) — a known hot spot. Textures are id-keyed uploads from `OffscreenCanvas`, with a cache that skips re-upload for static content via a `contentHash`. A `TexturePool` recycles render targets per frame; the whole frame is one command submit.

Playback ([playback-manager.ts](classic/apps/web/src/core/managers/playback-manager.ts)) is a `requestAnimationFrame` **wall-clock transport**. It does *not* drive `<video>` elements — it advances a clock, quantises to the frame grid via `roundFrameTime`, and everything else samples it. Video decode is a mediabunny `CanvasSink` with frame prefetch and a seek-generation counter to drop stale seeks.

Degradation is graceful: if `initializeGpu()` rejects, `gpuAvailable = false`, effects become pass-through, and `RendererManager.isDegraded` surfaces it in the UI.

### 4.5 Export — mediabunny/WebCodecs, sharing the preview's render path

No ffmpeg.wasm anywhere. [scene-exporter.ts](classic/apps/web/src/services/renderer/scene-exporter.ts) drives a **deterministic frame-by-frame loop**, not a realtime capture:

```ts
const ticksPerFrame = Math.round((TICKS_PER_SECOND * fps.denominator) / fps.numerator);
for (let i = 0; i < frameCount; i++) {
    await this.renderer.render({ node: rootNode, time: i * ticksPerFrame });
    await videoSource.add(mediaTimeToSeconds({ time: i * ticksPerFrame }), 1 / fpsFloat);
}
```

It feeds a mediabunny `Output` (MP4/`avc` or WebM/`vp9`) from a `CanvasSource` wrapping **the same `CanvasRenderer`/wgpu compositor the preview uses**. One render path for preview and export means the entire "looks different when exported" class of bugs cannot exist. Audio is pre-mixed to one `AudioBuffer` via `OfflineAudioContext`, with a runtime `AudioEncoder.isConfigSupported` probe that falls back from AAC to Opus.

### 4.6 Persistence — local-first, and 31 migrations deep

**No project or media data ever touches the server.** Two adapters behind one `StorageAdapter<T>` interface:

| DB | Kind | Holds |
|---|---|---|
| `video-editor-projects` | IndexedDB | full serialized project (metadata + scenes + tracks + elements + keyframes) |
| `video-editor-media-{projectId}` | IndexedDB | media metadata |
| `media-files-{projectId}` | **OPFS dir** | the raw `File` blobs |

Per-project DB/directory naming means deleting a project drops a whole DB and OPFS dir.

`CURRENT_PROJECT_VERSION = 31`, with 31 sequential migration classes (`V0toV1` … `V30toV31`), each a pure transformer with a test, run lazily on first load behind a progress UI. **Keybindings have their own separate chain (v2→v7).** This is the accumulated cost of evolving a schema in-place against users' IndexedDB — design the new format knowing that bill arrives.

Autosave is an 800ms debounce that refuses to run mid-migration and re-queues if a change lands during a save.

> ⚠️ **Verified bug in classic** — do not copy this path. [migrations/runner.ts:41](classic/apps/web/src/services/storage/migrations/runner.ts#L41) constructs `new IndexedDBAdapter<ProjectRecord>("video-editor-projects", "projects", 1)` with **positional** args, and line 98 calls `projectsAdapter.set(projectId, result.project)` — but [indexeddb-adapter.ts:8](classic/apps/web/src/services/storage/indexeddb-adapter.ts#L8) takes a **destructured object** (`{dbName, storeName, version}`), as does `set({key, value})`. Destructuring a string yields `undefined`, so this would call `indexedDB.open(undefined)` at runtime. It also shouldn't typecheck — meaning either the path is dead or the migration runner is broken. Confirmed by reading both files.

### 4.7 Commands, ripple, and actions — the layering that makes an Editor API possible

Two distinct layers, and the split is the reason the rewrite's plugin/scripting/MCP ambitions are credible rather than hand-waving.

**Commands** (`commands/`) are the *only* legal way to mutate the document — a textbook Command pattern over ~40 command classes. `CommandManager.execute()` runs: snapshot tracks → snapshot selection → `command.execute()` → **apply ripple adjustments** → apply selection patch → **run reactors** → push history → clear redo.

Three refinements worth stealing:
- **Ripple editing is a cross-cutting post-pass**, not per-command logic — `computeRippleAdjustments({beforeTracks, afterTracks})`, toggled globally.
- **Reactors** run after every execute/redo (currently garbage-collecting empty tracks).
- **Selection is only restored on undo if the command declared selection intent**, so UI-driven clicks and box-selects aren't clobbered.

Undo is **snapshot-based and coarse**: most commands stash the entire `SceneTracks` and restore it wholesale. Simple, but memory-heavy, and `CommandManager` has **no history size cap**.

**Actions** (`actions/`) are the input/trigger layer — a typed registry with per-action arg maps, keybindings (persisted, with their own migrations), and a command palette. The rule from [docs/actions.md](classic/docs/actions.md) is explicit and worth adopting verbatim:

> *"Avoid calling `editor.xxx()` directly from UI components — that bypasses the action layer (toasts, validation feedback, keybinding support)."*

The same principle recurs in the keyframes doc (`onReset` must call `commitValue`, not `editor.timeline.updateElements`). **UI → Action → Command → State.** That chain, plus the `params` bag, *is* the Editor API in embryo.

### 4.8 The Rust core exists — but is only ~15% of the intended migration

`classic/rust/crates/`, compiled to the `opencut-wasm` npm package (v0.2.10):

| Crate | Purpose |
|---|---|
| `bridge` | **proc-macro** — the `#[export]` attribute; the entire FFI convention |
| `time` | `MediaTime`, `FrameRate`, timecode. **The only real domain logic migrated.** |
| `gpu` | wgpu device/adapter/surface, canvas↔texture interop, blit |
| `effects` | effect-pass pipeline registry |
| `masks` | jump-flood SDF + mask feathering |
| `compositor` | full-frame layer compositor: transform → effects → mask → blend |

Reality check on that table:
- **`effects` has exactly one registered shader** (`gaussian-blur`), and `pack_effect_uniforms` hard-codes reads of `u_sigma`, `u_step`, `u_direction` and **errors on any other uniform name**. It is not shader-generic. Adding a second effect means touching Rust.
- Effects are **half-migrated**: TS owns the *what* (definitions, params, pass resolution), Rust owns the *how*.
- The web app depends on the **published npm package**, not a workspace path — local Rust dev requires `bun link`.
- Classic's CI test step is literally `echo "No tests implemented yet"` with `continue-on-error: true`.

### 4.9 The two conventions, machine-enforced on both sides of the FFI

This is the team's signature, and it's enforced twice:

- **TypeScript:** a custom ESLint rule, `opencut/prefer-object-params` — every function takes a single destructured options object. Carve-outs for direct callbacks (`reduce`, `new Promise`), inline option-object callbacks, and type predicates. Fully unit-tested.
- **Rust:** the `bridge` proc-macro emits a **hard compile error** if an `#[export]`ed fn has more than one positional parameter: *"must accept a single options struct… Wrap parameters in a struct."*

Alongside it: maximum type safety — `no-unsafe-type-assertion`, no `any`, no `!`, no `@ts-ignore`, no enums/namespaces, plus the branded `MediaTime`.

**Adopt both in the rewrite from the first commit**, or the codebase will diverge from every line of reference code we're porting.

### 4.10 The primitives-vs-domains lesson

[classic/notes/primitives-vs-domains.md](classic/notes/primitives-vs-domains.md) diagnoses a smell we should not reintroduce: primitive value types (`Transform`) parked in domain folders (`rendering/`), so unrelated domains take misleading dependencies. *"The dependency graph lies."* The test:

> *"If a type can be described without mentioning clips, tracks, effects, layers, keyframes, or any other product concept — and it has no behavior beyond shape — it's a **primitive**."*

Classic never executed this refactor in TS (its `src/` is ~45 flat domain folders with no `primitives/`). But note: **`rust/crates/time` is this note already executed in Rust** — time as a leaf primitive crate. The rewrite should do the same for geometry and colour, and establish the bucket *before* the first `Transform` is written.

---

## 5. Backend — essentially vestigial

Classic has no separate backend service. It's a thin slice of the Next.js app: Postgres + Drizzle with **five tiny tables** (the better-auth quartet plus `feedback`), better-auth with email/password and **no actual auth flows wired up** (the schema carries the comment *"we don't have any auth flows currently so this is fine for now"*), Upstash Redis rate limiting, and four API routes — auth, health, feedback, and a Freesound proxy for the sound library.

The editor is **local-first**. That's a genuine product decision, and the rewrite's `apps/api` currently reflects it correctly by being nearly empty.

---

## 6. Documentation drift found in classic (don't inherit it)

1. **`copilot-instructions.md` describes a Biome/Ultracite setup, but classic actually lints with ESLint 9 + Prettier.** `biome.json` exists and nothing invokes it. **The rewrite copied this file over — and has no linter at all.**
2. `docs/effects-renderer.md` references `rust/crates/gpu/src/shader_registry.rs`, **which does not exist**; the shader map is a `HashMap` in `effects/src/pipeline.rs`. Its bloom/glow/colour-grading discussion is aspirational — only `gaussian-blur` is implemented.
3. Classic's root `package.json` self-depends (`"opencut": "."`) and references a `@opencut/tools` workspace that isn't in the tree.

---

## 7. Gap analysis — rewrite vs. classic

| Subsystem | Classic | Rewrite |
|---|---|---|
| Editor core / managers | 12 managers | — |
| Timeline (tracks, drag, snap, ripple, placement) | ~105 files | — |
| Commands + undo/redo | ~53 files | — |
| Actions + keybindings | registry + palette + migrations | — |
| Preview + playback transport | yes | — |
| Renderer (wgpu compositor) | yes | — |
| Export (mediabunny → MP4/WebM) | yes | — |
| Persistence (IndexedDB + OPFS + 31 migrations) | yes | — |
| Media pipeline / decode / waveforms | yes | — |
| Effects, masks, keyframes/animation | yes | — |
| Text, stickers, subtitles, transcription | yes | — |
| Backend (auth, feedback, sounds proxy) | thin | — |
| **Rust core** | **6 crates** | **empty — no `crates/` dir** |

---

## 8. Open questions to settle before building

1. **Where does the Rust core live?** Root `crates/`, per the comment in `.moon/workspace.yml`. **`time` is the obvious first port** — `MediaTime` underpins the entire domain model, and the crate is self-contained, well-tested, and already correct.
2. **Does the domain model itself move into Rust this time?** Classic kept the model in TS and called Rust only for compute (time math, GPU). The stated goals — plugin-first, Editor API, headless, MCP, one core for desktop+mobile+web — push hard toward the model living in Rust, with TS as a projection. This is *the* architectural fork, and it should be decided before any timeline code is written.
3. **What is the Editor API, concretely?** It's the headline feature and everything else consumes it (plugins, MCP server, scripting tab, headless render). Classic's UI → Action → Command → State chain is the seed. Design it before the UI.
4. **How does the web shell subscribe to a Rust-owned model?** Classic's `useSyncExternalStore` + `subscribeAll` was already a known scaling edge with a *TS* model; across a WASM boundary it needs a real answer.
5. **Effects must be made shader-generic.** The current Rust `pack_effect_uniforms` hard-codes three uniform names. A plugin architecture that can't add a shader without editing the core isn't a plugin architecture.
6. **Is the shadcn inventory kept?** A pro editor's surface (timeline, tracks, scrubbers, canvas handles, graph editor) is almost entirely custom. Most of those 60 components will never be used.

---

## 9. How to run things

```sh
proto use              # installs moon 2.3.3, bun 1.3.11, rust 1.97.0

moon run web:dev       # localhost:5173
moon run api:dev       # localhost:8787
moon run desktop:dev   # cargo run — first build compiles GPUI, takes a while
```

---

## 10. Progress log

Newest first. One entry per goal.

### 2026-07-16 — Goal 7: influencer-ready landing page + production deployment — **DONE, verified**

~20 external testers are getting the link, so `/create` went from an internal tool page to a product page, and the deployment went from dev server to production build.

#### The page ([`src/app/create/page.tsx`](classic/apps/web/src/app/create/page.tsx))

Hero ("Steal the vibe. Keep your footage."), honest capability chips (beat-synced cuts, grading, moment selection, no cropping, 3–5 min), an animated cuts-on-the-beat waveform, guided numbered steps with per-step hints, an editor-status chat panel, a celebration + gradient download on finish, and a trust strip stating the two true privacy facts: full videos never leave the device (the AI sees only small key frames) and the render happens on the user's own GPU. All decoration is CSS/SVG — zero new deps, self-contained through the tunnel. A capability banner tells Safari/phone users to open Chrome on desktop (`VideoEncoder` detection). "Create again" now also shows in the done phase, so testers can iterate after the first render.

**Contract kept:** `main[data-ready]`, both file-input `accept` signatures, the exact composer placeholder, `data-testid="reference-error"`, and the page's ONLY `<video>` element is the finished export. Chat bubbles gained `data-role` so tests stop depending on styling classes (the old E2E matched `bg-white/5`, which the restyle broke).

#### A real race, caught by the E2E on the new page

The interview auto-started while `analyzeUserMedia` was still running, so the agent's first `get_media_profile` returned "not analyzed yet" and it interviewed blind on footage it was supposed to have watched. Fix: a `mediaAnalysisPending` gate on `canStartInterview`, raised **synchronously before the first await** so React batches it with the reference/media state updates — the interview-start effect can never see "ready" with the analysis still due.

#### Production deployment

Screenshots exposed that guests would have hit the DEV server through the tunnel: react-scan overlay boxes on every component, dev badges, per-route compile stalls. Now: `next build` (standalone) served on **port 3006** — the dev server on 3005 untouched — with static/public/env copied into the standalone tree, and the cloudflared quick tunnel repointed. `ignoreBuildErrors` documented as covering only the 18 pre-existing dependency-drift errors; `bunx tsc` (0 errors in agent code) stays the source of truth.

**Verification:** 127 unit tests green; the FULL UI E2E run against the production build passed end to end (upload → analysis → interview → Create → 8.00s 1080×1920 MP4 downloaded); the redesigned page confirmed serving through the tunnel.

**Caveats for the trial:** `/api/agent/*` still has no auth or rate limit — a handful of semi-trusted testers on a quick tunnel is the accepted risk, a public launch is not. The tunnel URL dies with the cloudflared process. `/api/agent/render` (the headless backend route) returns 501 on the standalone build (playwright isn't traced into it) — irrelevant to the browser flow the testers use.

**Trial additions (same day):** a **feedback widget** (floating bottom-LEFT — the bottom-right corner is where the composer and Create button live, and a floating element there could intercept their clicks) posting `{rating, message, handle}` to `/api/preview/feedback`, and a **unique-visitor counter** at `/api/preview/visit` (IPs stored as truncated SHA-256 hashes, append-only JSONL deduped at read — concurrent-append-safe where a JSON blob would race). Both persist to `~/.opencut-preview/` — deliberately OUTSIDE `.next/standalone`, which every rebuild wipes. Footer shows "you're one of N creators" once N ≥ 2. Verified through the tunnel end to end: browser click-through of the widget (4 stars → JSONL line lands), beacon dedupe confirmed.

---

### 2026-07-15 — Goal 6: cut to the beat — **DONE, verified**

The output stops being "paced like the reference" and starts being "in time with the user's music" — cuts land a frame before the beats of THEIR track, in the reference's rhythm. This is the audio-visual sync idea from the brainstorm, built end to end.

#### The idea, in one line

Project both modalities into the same shape — an "event strength over time" curve — measure the coupling rule between them in the reference (not the timestamps, the RULE, in beat units), then re-instantiate that rule on the beat grid of the user's own track.

#### What was built — [`src/agent/audio/`](classic/apps/web/src/agent/audio/)

Three pure cores (plain math over Float32Arrays, all `bun test`-able against synthesized PCM where ground truth is exact), one browser shim, one tool:

- **[`beats.ts`](classic/apps/web/src/agent/audio/beats.ts)** — PCM → 3-band biquad filter bank → log-RMS envelopes → novelty curve → onsets → tempo (autocorrelation, harmonic reinforcement, gentle 120 BPM prior, parabolic interpolation) → dynamic-programming beat tracking (Ellis-style: recovers phase, not just period). `hasTempo: false` is a first-class result — speech and ambience must NOT hallucinate 120 BPM, and the tests pin that.
- **[`sync.ts`](classic/apps/web/src/agent/audio/sync.ts)** — circular statistics over cut-to-beat offsets: resultant length R, Rayleigh significance gate (with ~12 cuts, apparent alignment happens by luck — a false "beat-locked" would be worse than none), mean anticipation (editors cut a frame BEFORE the beat), and the shot-length histogram in beat units.
- **[`schedule.ts`](classic/apps/web/src/agent/audio/schedule.ts)** — the transfer: walk the USER track's beat grid, choose shot lengths by largest-deficit weighted round-robin over the reference's unit histogram (deterministic — same input, same schedule, byte for byte), force a cut on every drop, apply the anticipation, quantize to frame boundaries and **never round past the beat**.
- **[`decode.ts`](classic/apps/web/src/agent/audio/decode.ts)** — mono PCM out of anything (Web Audio for audio files, mediabunny `AudioBufferSink` for video containers), capped at 2 minutes of analysis.
- **`plan_beat_grid`** — a READ-ONLY tool (legal during the interview, so the agent's plan can say "124 BPM, cut every 2 beats"). The model decides WHAT fills each slot; this decides exactly WHEN the slots are. An LLM hand-emitting twelve floats that each hit a beat within ±20ms was never going to happen — same division of labour as trimStart and the transition planner.

Plus **cut refinement**: the 10fps shot detector knows a cut to ±100ms, but the offsets being measured are ~20ms — the correlation would have measured our own detector's noise. `refineCutTimes` re-decodes ±120ms around each coarse cut at 60fps (merged into one linear pass) → ±8ms.

#### One real bug found by the unit tests

A held sine tone produced a *periodic ripple* in block RMS (the sine phase drifting against the hop boundary) which, after normalization, read as strong rhythm. Fix: a **sparsity gate** — rhythm is silence punctuated by arrivals (median flux ≈ 0); continuous flux (tones, rain, applause) is not rhythm, and the median exposes it.

#### Verification

**158 unit tests** (31 new): 120→120.0, 90→90.0, 100→100.0 BPM on synthesized clicks; beats within 30ms of truth including phase; silence/tones/aperiodic speech-like bursts all correctly refuse a tempo; R≈1 for on-beat cuts and R low for off-beat; anticipation recovered to the millisecond; scheduler slots tile exactly, drops always get a cut, determinism pinned by equality.

**The E2E that matters** ([`run-beat-sync-e2e.mjs`](classic/scripts/agent-e2e/run-beat-sync-e2e.mjs)): a 120 BPM reference cut on every 2nd beat — beat-locked *by construction*, beeps and scene switches driven off the same AudioContext clock — plus a **100 BPM** user music track. Different tempo on purpose: matching output proves grammar transfer, not timestamp copying.

```
reference tempo  : 120 BPM (truth: 120)     reference locked : true
tools: ... plan_beat_grid -> add_clips ... add_audio ...
clip starts : 0.00, 1.33, 2.53, 3.73, 4.93, 6.13, 7.33, 8.53, 9.73
deltas      : 1.200s = 2.00 beats — on grid, all 7 of 7
```

Every cut interval is **exactly 2.00 beats of the user's track** (1.200s), from a reference whose own grid was 0.5s. Two consecutive runs produced identical cut times — the scheduler is deterministic in production, not just in tests. Slot 0 absorbs the pre-first-beat gap (music's first beat sits at ~0.13s after opus priming), which the E2E initially flagged as a failure until the assertion was corrected to measure between cuts.

No-music regression E2E: unchanged, green. Biome clean; typecheck 0 in my code (repo baseline 18).

#### Honest limits

- Only helps music-driven references; a talking-head reel measures as not-locked and everything falls back to today's visual pacing (that fallback is itself measured, not assumed).
- Tempo drift: the beat tracker follows produced (click-locked) music well; live rubato performances would need beat-grid interpolation smarter than linear.
- Energy→cut-rate coupling (double-time in the chorus) was consciously cut from v1 — the unit histogram + drop forcing carries most of the perceptual weight.
- The music bed is laid from `trimStart` as given; choosing WHERE in the track to start (so a downbeat hits the opening) is a natural v2 of `plan_beat_grid`.

---

### 2026-07-14 — Goal 5: reframing is banned — **DONE, verified**

The quality budget (below) showed the reframe was destroying **~68% of the user's pixels** — by far the biggest loss in the pipeline. So it is now forbidden. Not discouraged; **impossible**.

#### The guarantee

`scale: 1` is CONTAIN: the whole source frame fits inside the canvas. Anything above 1 pushes the picture past the edge and the overflow is cropped away. There were exactly **three routes** to a scale above 1, and all three are now closed in the executor, not in the prompt:

| Route | Enforcement |
|---|---|
| `add_clips.scale` | clamped to ≤ 1, and the clamp is *reported* to the model, not silent |
| `update_elements.scale` | same clamp |
| `animate_elements` (preset **or** raw keyframes) | `capScaleKeyframes` |

The keyframe route is the subtle one — a zoom preset crops just as effectively as a static scale, only gradually. Naively clamping each keyframe to 1 would turn `ken-burns-in` (1.0 → 1.12) into 1.0 → 1.0 and **silently delete the animation**. So the curve is *shifted down* to peak at exactly 1: 1.0 → 1.12 becomes **0.893 → 1.0**. Same 12% push, same easing, nothing cropped — the shot now grows *into* the full frame instead of past it.

#### The canvas is filled by the background, not by cropping

Classic already had `background: { type: "blur" }` — a cover-scaled, gaussian-blurred copy of the clip's own frame, drawn beneath it. The agent had simply never used it (`set_project_settings` hardcoded a flat colour). It is now exposed and **defaults to blur**, at intensity 200 (classic's own default of 10 is a sigma of 3.5 — sharp enough to read, which looks like a bug rather than a design).

Two real defects in that backdrop had to be fixed for it to be usable:

1. **The backdrop was pinned to `opacity: 1`.** A `fade-to-black` would fade the picture away and leave the blurred backdrop sitting there at full strength — the shot never actually reached black. It now inherits its clip's opacity and keyframes.
2. **It was built from the `main` track only.** A cross-dissolve puts its two clips on *different* tracks so they can overlap, so every other clip in a dissolving sequence had **no backdrop at all**. It now reads every visible video track.

#### Chasing a blur that wasn't there

The first render came back with a backdrop that measured as sharp as the clip. Four hours of the wrong suspects (uniform slots, `u_direction`, multi-pass chaining, stale wasm) before the truth: **the blur was working the whole time, and the measurement was wrong — twice.**

- The *mean* gradient is useless here: both regions are mostly smooth gradient, so the flat areas dominate and the few real edges vanish. Switched to the 99th percentile.
- The strips are then contaminated by things that are *legitimately* sharp: the agent's captions render on top, and a `pan-up`/`ken-burns` keyframe slides the clip itself off-centre and into the strip. Fixed by taking the *softest* of the two strips — an intruder rarely covers both, but a broken blur leaves both sharp.

Measured properly: backdrop edge strength **1.9–6.0**, clip **16–48**. The backdrop carries no hard edges at all.

The lasting win from that hunt is [`test-gpu-effects.mjs`](classic/scripts/agent-e2e/test-gpu-effects.mjs): it loads the compiled wasm compositor **straight from `rust/wasm/pkg`**, uploads a hard black/white edge, runs one effect pass, and reads the pixels back. The Rust unit tests cannot link on this machine (no MSVC toolchain), so the shaders had **zero** test coverage; this covers them from the browser, where they actually run. It proves the gaussian blur blurs, scales with sigma, and respects `u_direction` — in ~5 seconds, with no agent and no API cost.

#### Verification

```
scales chosen : 1x, 1x, 1x, 1x, 1x, 1x, 1x, 1x, 1x, 1x, 1x, 1x
canvas fill   : blur
lit           : 100% on all 20 sampled frames
```

`litFraction` is the load-bearing number, and it now catches both failure modes at once: a contained 16:9 clip fills only **31.6%** of a 9:16 canvas, so **100% lit is only possible if the blurred backdrop is genuinely rendering behind it**. A letterboxed reel would read ~32%; dead footage would read ~0%.

**96 unit tests** (7 new, pinning the no-crop guarantee across every preset at every strength). Biome clean. Typecheck: **0 errors in my code**, repo total unchanged at 18.

#### What this costs, honestly

A 16:9 clip in a 9:16 canvas now occupies a band across the middle, with a blurred wash above and below. That is the standard Reels/TikTok look and **not one pixel of the user's footage is thrown away** — but it is a different frame from an edge-to-edge crop, and some people will prefer the crop. The way to get a full-bleed frame with no loss is to shoot vertical, which the agent is now told to prefer.

---

### 2026-07-14 — Quality budget: where the pipeline actually loses fidelity — **MEASURED**

Asked "what's the loss percentage from upload to output". There is no single percentage — but every stage is either exactly computable or measurable, so: [`measure-quality-loss.mjs`](classic/scripts/agent-e2e/measure-quality-loss.mjs).

| Stage | Loss | How we know |
|---|---|---|
| Upload / ingest | **none** | `processMediaAssets` stores the original `File`; no transcode |
| Decode | **none added** | WebCodecs decode is deterministic; the phone's compression is already baked in |
| Composite | 8-bit per pass | `GPU_TEXTURE_FORMAT = Bgra8Unorm`; grade + vignette + sharpen = 3 extra 8-bit round trips |
| **Reframe 16:9 → 9:16** | **~68% of source pixels discarded** | geometry, exact |
| Export (H.264, `QUALITY_HIGH`) | **~35 dB PSNR-Y / 0.93 SSIM** | measured, below |
| Platform re-encode | outside our control | IG/TikTok re-encode to ~2–4 Mbps |

**Export, measured** — pristine RGB frames pushed through the exact export config (`CanvasSource` → `avc` → `QUALITY_*`), decoded back, compared to ground truth at 1080×1920:

| preset | bitrate | PSNR-Y | SSIM |
|---|---|---|---|
| low | 1.78 Mbps | 30.34 dB | 0.767 |
| medium | 4.11 Mbps | 32.57 dB | 0.861 |
| **high (default)** | **5.13 Mbps** | **34.97 dB** | **0.926** |
| very_high | 6.63 Mbps | 37.08 dB | 0.956 |

`QUALITY_HIGH` is factor 2 against mediabunny's 3 Mbps @ 1920×1080 reference → **6 Mbps** for a 9:16 export; measured 5.1–6.3 Mbps depending on content. Comfortably above what Instagram will accept, which is where you want to be. Audio is AAC 192 kbps — transparent.

**The dominant term is the reframe.** A 9:16 slice of a 1920×1080 frame is 607×1080 = 656k pixels out of 2.07M: **you keep 31.6%**. That crop is then enlarged 3.16× in area to fill 1080×1920, which adds no information. Everything else in the pipeline is a rounding error next to this.

> **Superseded — see Goal 5 above.** Reframing is now **banned outright**: clips are never scaled past contain, and the canvas is filled with a blurred backdrop instead. This loss is gone.

#### Latent bug found and fixed: the frame-fill number was wrong

The renderer uses **contain**: `containScale = min(canvasW/sourceW, canvasH/sourceH)` ([resolve.ts:162](classic/apps/web/src/services/renderer/resolve.ts#L162)). The system prompt told the agent to *"punch in with scale 1.2-1.5 so the frame is filled rather than letterboxed"* — but at 1.5 a 16:9 clip is 911px tall in a 1920px canvas. **That is 53% black bars.** The scale that exactly fills is:

> **scale = (canvasHeight / canvasWidth) × (sourceWidth / sourceHeight)** → **3.16** for 16:9 into 9:16

The agent had been quietly compensating (every prior render came out 100% lit, zero bars), so the output was right *by luck* — a weaker model, or a different framing, would have followed the bad advice and shipped a letterboxed reel. The prompt and the `add_clips` schema now carry the formula; the agent picks **3.2× on every clip**, verified. The E2E now asserts fill scale, so a letterboxed reel is a **test failure** rather than something we hope the model avoids. Chosen scales are also exposed on the render response as `X-Opencut-Scales`.

#### The one fixable precision loss

Intermediate render targets are **8-bit** (`Bgra8Unorm`), and each effect is its own pass. The colour-grade shader converts sRGB → linear, grades in linear light, converts back, and quantizes to 8 bits — **once per pass**. With grade + vignette + sharpen that is three quantizations, which is where banding in gradients and shadows comes from. The fix is `Rgba16Float` intermediates, blitting to 8-bit only at the end. Not attempted here, and deliberately **not** given a dB figure — measuring it honestly needs the Rust change to A/B against.

---

### 2026-07-14 — Goal 4: the agent looks at the user's footage — **DONE, verified**

Prompted by a deep read of an AI-video-editing research paper the user shared. The paper itself is not a source of technique — its results section evaluates a different system than its methods section describes (Places365/YOLOv8 appear only in §4), its ROC figure reports **AUC = 0.01** for activity recognition while the prose calls it "high confidence", its confusion matrix (n=100) and classification report (n=500) cannot both be true, it names WER as its primary metric and never reports one, and its Whisper citation fabricates the author list. But the *problems it names* were real gaps in our system. This closes the largest one.

#### The gap

`add_clips` had always exposed `trimStart` — "use this to pick the interesting moment" — while the agent **had never seen a single frame of the user's footage**. `analyzeReference` ran only on the reference reel; the user's own clips went straight from `processMediaAssets` to `addMediaAsset` and nobody decoded them. The agent was choosing which second of your video the viewer sees **by guessing**.

#### What was built

A second analysis pass, mirroring the reference one, in [`src/agent/media/`](classic/apps/web/src/agent/media/):

- **Measured (exact).** Reuses `decodeReference`/`detectShots` verbatim — the same code path whose shot detection we validated at 12/12 against ground truth — then scores every candidate window on exposure, motion and colour. Pure, so it's unit-tested.
- **Seen (judgment).** Shows Claude the frames it is *actually considering using* and asks what is in them. New route [`/api/agent/describe`](classic/apps/web/src/app/api/agent/describe/route.ts), one call for every clip. Degrades gracefully: if vision fails, the measured in-points still stand.
- **New tool `get_media_profile`** (read-only, so it is legal during the interview — the agent's *questions* get better too).
- Window length is taken from the **reference's average shot**: the best half-second of a clip is the wrong question when the reference holds for three.

We deliberately do **not** claim focus detection. Blur lives in the high frequencies and a 16×16 thumbnail throws exactly those away. Heavy shake *does* show up as very high motion, and is penalized.

#### Two bugs the E2E caught — both mine, not the model's

The trap: three user clips that are **black for their first 3 seconds**. A blind agent takes `trimStart: 0` and cuts the black straight in.

1. **Mean-based blankness let a black frame through.** A 0.5s window that opens on one black frame and is bright for the rest has a perfectly healthy *average*, so it sailed past the check — and the agent took `trimStart: 3.0`, landing exactly on the boundary. One frame of the output was black. **Fix:** a window is judged on its *worst* frame, not its mean. One dead frame poisons the window. Plus a boundary guard: candidate windows are inset by one sample interval, because a 10fps detector only knows a cut to ±0.1s and the frames sitting on a seam are the ugly ones anyway.

2. **Offering one window per clip made the agent invent the rest.** With 12 timeline slots and 3 clips, a single suggestion each is not enough, so it padded with made-up in-points (1s, 2s — straight back into the black). **Fix:** `rankWindows` returns a *menu* — non-overlapping distinct moments chosen by greedy non-maximum suppression. Without the suppression the top of the list was a dozen near-identical windows sliding a quarter-second apart across the same instant, which is a menu with one dish on it. The tool and prompt now also say plainly: **never invent an in-point; reuse a window instead.**

A third bug was in the *test*: white caption text on a black frame has low mean luma but huge variance, so it read as "has picture". The verifier now measures what fraction of the frame is lit, which a caption cannot fake.

#### Verification — nothing mocked

- **89 unit tests** pass (up from 62). The new ones pin the scoring semantics: one black frame rejects a window however healthy its average; violent motion scores *below* a frozen shot; a frozen-but-lit shot still scores, because a ken-burns can save it; windows never straddle a boundary and never open flush on a seam.
- **New E2E** ([`run-media-profile-e2e.mjs`](classic/scripts/agent-e2e/run-media-profile-e2e.mjs)) renders through the full stack from the black-first-half footage and asserts three things: the agent called `get_media_profile`; **every in-point it chose lands past 3s**; and **0/20 sampled output frames are dead** (100% lit on all 20).
- The chosen in-points came back **3.2, 3.7, 4.2, 4.2, 4.45, 4.7, 4.7, 4.95, 5.2s** — spread across the good half, which is exactly what a menu of distinct windows should produce.
- Base and transitions E2Es re-run green. Typecheck: **0 errors in my code**; repo total unchanged at 18.

The in-point audit trail is now on the render response as `X-Opencut-Trims`, so which moments the agent chose is inspectable from outside without re-running anything.

#### Known limitation

A **dissolve** extends the incoming clip's source read *forward* past its out-point (it needs spare source after it — it never reads backwards, so a cut can still never *open* on black). That tail can run up to `transitionDuration` beyond the scored window. Bounded to a few hundred ms, and transitions default to `none` for fast reels, so it is a real but mild gap.

---

### 2026-07-14 — Goal 3: press Create from the backend (headless render) — **DONE, verified**

One HTTP call in, one finished MP4 out. No browser tab, no Create button, no human.

```sh
node scripts/render.mjs \
  --reference reel.mp4 \
  --media clip1.mp4 --media clip2.mp4 --media photo.jpg \
  --brief "Promo for my coffee shop, punchy and fast" \
  --aspect 9:16 --out promo.mp4
```

```sh
curl -X POST http://localhost:3005/api/agent/render \
  -F reference=@reel.mp4 -F media=@clip1.mp4 -F media=@photo.jpg \
  -F brief="Punchy coffee shop promo" -F aspectRatio=9:16 \
  -o promo.mp4
```

#### The constraint, stated plainly

**The backend cannot render a frame.** The editor engine — `EditorCore`, the wgpu compositor, OPFS, WebCodecs — exists *only* in a browser. There is no compositor in Node to draw with.

So "press Create from the backend" cannot mean "render server-side". It means **the backend drives a headless browser**. [`/api/agent/render`](classic/apps/web/src/app/api/agent/render/route.ts) launches headless Chromium against our own [`/headless`](classic/apps/web/src/app/headless/page.tsx) page, which runs the *same* EditorCore, the *same* tool executors, the *same* agent loop and the *same* compositor the UI does, and hands the encoded bytes back.

This is not a workaround — it is the architecture telling the truth about itself. It's the same property that forces the rewrite's desktop app to need a Rust core.

#### Autonomous mode

Headless has no human, so **the interview cannot happen** — a question would hang the render forever. `AUTONOMOUS_TRIGGER_MESSAGE` tells the agent there is nobody there, forbids `ready_to_create`, and requires it to decide for itself where the brief is silent. `allowEditing` starts `true`: the Create button is, in effect, already pressed.

#### What came back

A real run, 112 seconds, from four files and one sentence of brief:

```
tools : get_editor_state, get_reference_profile, set_project_settings, add_clips,
        get_editor_state, apply_look, add_effects, add_effects, animate_elements,
        add_texts, get_editor_state, animate_elements, export_video
file  : 1505 KB
```

Verified: **6.00s, 1080×1920 vertical, picture on every one of 8 sampled frames.** The tool trace comes back in the `X-Opencut-Tools` response header, so the edit is auditable.

Note the agent verified its own work **three times** (`get_editor_state`) and graded, added two effects, animated, and captioned — unprompted, from a one-line brief.

#### Caveats worth knowing

- **Node hosts only.** Playwright and a real browser binary are required, so this route cannot run on Cloudflare Workers (how classic deploys). The import is dynamic and returns a clear **501** there rather than breaking the whole deploy.
- **One browser per request.** Fine for automation and batch jobs; a production service would pool browsers or move to a queue.
- **No auth and no rate limit** on the endpoint. It burns API tokens and CPU on demand — do not expose it publicly as-is.

---

### 2026-07-14 — Goal 2: transitions, colour grading, keyframe animation — **DONE, pixel-verified**

The agent could cut, place, scale and caption. It could not dissolve, grade, or animate. Now it can.

#### The hard part: the GPU could only run one shader

The audit had flagged that classic's Rust effects pipeline was "not shader-generic". Building this proved exactly how badly:

- The compiled `opencut-wasm` contained **one** fragment shader, `gaussian-blur`.
- `pack_effect_uniforms` **hardcoded** its three uniform names (`u_sigma`, `u_step`, `u_direction`) and returned `UnsupportedUniform` for anything else.

So colour grading was not "unimplemented" — it was **unreachable**. No amount of TypeScript could add it. It needed new WGSL shaders *and* a rewrite of the uniform packing, then a rebuild of the wasm.

**Toolchain, from nothing.** Rust wasn't installed. Installed it, hit `link.exe failed` — the `link.exe` on PATH is Git Bash's GNU coreutils `link`, and real MSVC Build Tools are absent. Rather than pull down multi-GB build tools, switched to rustup's GNU host toolchain (self-contained linker). The **wasm32 target builds fine**; only the *host* build is blocked (the `windows` crate needs a `dlltool` that ships incomplete), which costs nothing but the Rust unit tests — and a shader is far better proven by its pixels anyway.

**The fix, in `rust/crates/effects/src/pipeline.rs`:** a shader now *declares* its own uniforms.

```rust
struct ShaderSpec { id, source, uniforms: &[UniformSpec] }
struct UniformSpec { name, slot: Slot, default: f32 }   // Slot::Direction | Slot::Scalar(0..11)
```

Adding an effect is now a table entry plus a `.wgsl` file. Unknown uniforms are still rejected — but "known" is per-shader instead of one global hardcoded list. Every uniform has a **neutral default**, so an effect with no params set changes not a single pixel.

**New shaders** (`rust/crates/effects/src/shaders/`):

| Shader | What it does |
|---|---|
| `color_grade.wgsl` | exposure → white balance → lift/gamma/gain → contrast → **split-toning** → saturation → **vibrance** |
| `vignette.wgsl` | aspect-corrected corner falloff (a naive one becomes a horizontal band on 9:16) |
| `sharpen.wgsl` | unsharp mask — puts back the detail that punching in destroys |

The grade works in **linear light** (exposure and white balance are physical operations and are simply wrong on gamma-encoded values), and **un-premultiplies alpha** before grading and re-premultiplies after — grading premultiplied colour darkens edges as a function of alpha, which shows up as a dark fringe around faded or masked layers.

**Split-toning** was added deliberately: lift/gamma/gain are achromatic, so they *cannot* produce cool-shadows-against-warm-highlights — the signature modern look. **Vibrance** saturates muted pixels hardest and leaves already-saturated ones alone, which is what stops skin going radioactive when you push colour.

The shared uniform block grew to 16 floats (64 bytes — still a multiple of 16, which WebGL requires). All four shaders share it, so all four `.wgsl` structs had to stay in lockstep.

#### Transitions: there is no transition primitive

Elements on one track cannot overlap, and there is no transition element type. So transitions are **synthesized** ([`agent/tools/transitions.ts`](classic/apps/web/src/agent/tools/transitions.ts), pure and unit-tested):

- **dissolve** — the incoming clip is pulled *earlier* to overlap the outgoing one, and placed on a second track. Its **out-point is preserved**, so adding a dissolve does not shift every downstream clip. Compositing order (verified in `scene-builder.ts`) puts `main` at the bottom and overlay tracks above it, so the planner fades whichever clip is **on top** — fading the incoming one *up*, or the outgoing one *out* to reveal the incoming underneath. Both read as a true cross-dissolve.
- **dip-to-black** — no overlap: the outgoing fades into the canvas and the incoming fades out of it.
- **fade-from-black / fade-to-black** — opacity ramps against the background.

A transition is automatically shortened, or dropped, when the clips are too short to support it or the incoming clip has no spare source tail left to give.

#### Motion

[`agent/tools/motion.ts`](classic/apps/web/src/agent/tools/motion.ts) — 12 presets (ken-burns in/out, punch-in, four pans, fades, pop, drift-rotate) built on classic's real keyframe system with bezier easing. Times are normalized (0..1 of the element's own duration), so one preset works on a 0.4s cut and a 6s hold. A uniform zoom writes **both** scale axes — writing one stretches the image. A fade stays a fixed **wall-clock** length rather than scaling with the clip, and never eats more than a third of a very short one.

#### New tools

`apply_look` (10 graded looks — cinematic, warm-film, punchy, vintage, noir, golden-hour, bleach-bypass…, each scalable by intensity), `add_effects` (vignette / sharpen / blur), `animate_elements` (presets or raw keyframes), and `transitionIn` on every clip in `add_clips`.

The model picks a look by **name**, not by inventing lift/gamma/gain numbers — it isn't sitting in front of a scope.

#### Two more bugs found by building this

1. **`EditorCore`'s reactor prunes empty overlay tracks after every command.** Creating a track then inserting into it fails with `Track not found`, because the track is deleted in between. Fixed by never pre-creating it: layer-1 clips use auto-placement, and because they overlap the main track the placer is *forced* to make an overlay track for them.
2. **`bun add` silently un-linked the local wasm.** A later install replaced the `bun link` symlink and the app quietly went back to the *published* wasm — the giveaway was `Missing uniform 'u_sigma' for shader 'color-grade'`, an error only the old hardcoded code can produce. Fixed properly by pinning `"opencut-wasm": "file:../../rust/wasm/pkg"` in `package.json`, which survives installs.

#### Verified — at the pixel level, nothing mocked

**Colour grade** ([`run-effects-e2e.mjs`](classic/scripts/agent-e2e/run-effects-e2e.mjs)): the agent is asked for a **noir** look. The source clips are vivid colour gradients (chroma ≈ 60–120). The exported video measures **chroma 0 on all 10 sampled frames** — perfectly greyscale. That is impossible unless the new shader ran on the GPU.

**Vignette**: corners measurably darker than centre in **10/10 frames** (ratios 0.23–0.86).

**Cross-dissolve** ([`run-transitions-e2e.mjs`](classic/scripts/agent-e2e/run-transitions-e2e.mjs)): the video is sampled every 0.15s and consecutive colour distance plotted. A hard cut is a single spike between flat runs. The export shows **two sustained ~1-second ramps landing exactly on the two cut points** — the colour interpolating between clips.

**62 unit tests** (shot detection, tool schemas, definition↔executor coupling, phase gate, transition overlap arithmetic, motion presets). Zero new typecheck errors: with my source removed entirely the same `node_modules` produces 20 errors, and with it 18 — the 6 in `components/ui` are pre-existing dependency drift (classic pins `"@types/bun": "latest"`, so any install shifts them).

#### Still not there

- **Rust unit tests for the packing logic can't run on this machine** — the GNU host build needs a working `dlltool`. The tests are written and correct; the wasm target compiles them. Proven by pixels instead.
- **No LUT support.** A real grading suite loads `.cube` files. The primary grade is here; LUTs are the natural next step.
- **Split-toning is a warm/cool axis, not a full colour wheel.** Enough for teal-orange; not enough for a per-channel shadow/highlight wheel.
- **Speed ramps, masks and text animation presets** remain unexposed to the agent.

---

### 2026-07-14 — Goal 1: the agentic editor (`/create`) — **DONE, E2E verified**

**Built in `classic/`, not the rewrite.** The rewrite has no timeline, renderer, or export — there is nothing there to make agentic. Classic has `EditorCore`, a command system, a wgpu compositor and a working mediabunny export path. Those became the agent's tools.

**What it does.** At `/create`: the user drops in a reference reel (an Instagram/TikTok-style video whose *feel* they want), then their own footage and photos, then has a short conversation with Claude. When Claude has what it needs it calls `ready_to_create`, which reveals the **Create** button. Pressing it makes Claude build the timeline itself, tool call by tool call, and export a finished video.

#### The load-bearing architectural decision

**The browser owns the agent loop; the server is a dumb proxy.**

The editor engine is browser-only — `EditorCore`, the wgpu compositor, IndexedDB, OPFS. Tools therefore *cannot* execute server-side. So [`/api/agent/chat`](classic/apps/web/src/app/api/agent/chat/route.ts) is stateless: it holds the API key, attaches the tool definitions and the cached system prompt, and returns Claude's raw turn. [`agent/loop.ts`](classic/apps/web/src/agent/loop.ts) runs in the browser: call Claude → execute `tool_use` blocks against `EditorCore` → post `tool_result` back → repeat. It's the standard manual tool-use loop, split across the network boundary.

#### The tool surface (10 tools)

Fine-grained enough to compose freely, but the build tools take **arrays**, so a 16-cut reel is one round trip, not sixteen.

| Tool | Purpose |
|---|---|
| `get_editor_state` | media ids, timeline contents, canvas settings |
| `get_reference_profile` | the measured + visually-read reference analysis |
| `ready_to_create` | **reveals the Create button** — a structured readiness signal, not parsed prose |
| `set_project_settings` | aspect ratio, fps, background |
| `add_clips` | the whole cut sequence in one call |
| `add_texts` | hooks and captions |
| `add_audio` | music bed |
| `update_elements` / `delete_elements` | refinement |
| `export_video` | render + deliver |

**The model speaks in seconds (float); executors convert to `MediaTime` ticks at the boundary.** The model never touches tick math, and the integer-time invariant survives intact.

#### Reference analysis — measured, then read

Two halves, deliberately separable:
- **Measured (exact):** decode the reel, sample at 10fps, fingerprint each frame, diff consecutive frames to find hard cuts → shot list, average shot length, cuts/minute. Pure math in [`reference/shots.ts`](classic/apps/web/src/agent/reference/shots.ts), unit tested.
- **Read (judgment):** send 8 key frames — one per shot, spread across the video — to Claude vision with **structured outputs**, returning a validated style profile (pacing, techniques, text usage, mood).

If the vision call fails, the analysis still succeeds on the measured half. The cut rhythm is the load-bearing part; style notes are a bonus.

#### Two-phase rule, enforced in code

The prompt tells the model not to edit during the interview. `runTool` **guarantees** it: every editing tool is refused until the user presses Create, and the gate short-circuits *before* the executor can touch the editor. A prompt is a request; this is an invariant. Proven in [`__tests__/phase-gate.test.ts`](classic/apps/web/src/agent/__tests__/phase-gate.test.ts).

#### Bugs found and fixed while building

1. **`InsertElementCommand` silently clobbers the canvas.** When the *first* visual element lands it snaps `canvasSize`/`fps` to the source asset ([insert-element.ts:83-111](classic/apps/web/src/commands/timeline/element/insert-element.ts#L83-L111)). So setting 9:16 and *then* adding a landscape clip would reset the canvas to landscape. `add_clips` now snapshots and restores the intended settings.
2. **Reference decoding took minutes.** I called `sink.getSample(t)` per sample — each one a random *seek* that re-decodes from the preceding keyframe. Switched to mediabunny's `canvasesAtTimestamps()`, which decodes each packet at most once: **minutes → ~3 seconds.**
3. **Shot detection under-counted badly** (5 shots in a 12-cut video). The grayscale-luma fingerprint couldn't see colour cuts — a saturated red and a saturated green have nearly identical luma. Switched to an RGB signature and finer sampling: now **12/12, 0.499s average vs 0.5s ground truth.**
4. **Export died with "GPU context not initialized".** The GPU and font atlas are initialized by `EditorProvider`, which only wraps `/editor`. `/create` isn't inside it, so nothing booted them — and it failed at the *last* step, after the agent had done all its work. The session hook now boots the runtime itself.
5. **Errors were invisible.** A failed analysis only fired a `toast`, and the `<Toaster>` isn't mounted on this route. Errors now render inline in the panel.

#### Verification — nothing mocked

- **37 unit tests** (shot detection, tool schemas, definition↔executor coupling, phase gate). Note `bun test` cannot import `opencut-wasm` (a pre-existing condition: classic's *own* tests fail the same way and its CI never ran them), so there's a faithful wasm mock with real 120,000-tick semantics.
- **A full E2E** ([`scripts/agent-e2e/`](classic/scripts/agent-e2e/)) drives a real browser against the real Claude API and the real wgpu compositor: generates test video, uploads, interviews, answers, presses Create, and waits for the download. It produced a **1080×1920, 8.00s MP4**.
- **Output verified at the pixel level**: 0/16 sampled frames blank, 14/16 distinct. The `stdDev` spikes at 0.25–1.25s and 6.25–7.75s are the text overlays — the pixels independently confirm the agent built what it said.

**Result of the E2E run:** from a 6s/12-cut reference and 3 landscape clips + 1 photo, the agent chose **16 clips across 8 seconds (0.5s per cut)** — matching the reference's measured 0.499s average — punched in on the landscape footage to fill vertical, put "NEW SUMMER MENU" up front and "OPEN 7AM" at the close, then verified its own timeline before exporting.

**Prompt caching works:** the frozen system prompt + tool definitions are a byte-stable prefix (4,690 tokens), read from cache on every turn after the first.

#### Known gaps (not blockers)

- **No streaming.** Each agent turn is a single non-streaming request. Turns are short and tool-heavy so it's a reasonable trade, but the chat would feel snappier with SSE.
- **No rate limiting on `/api/agent/*`.** Fine locally; the route holds an API key, so it needs a limiter before it's ever exposed. Classic already has Upstash wired for `/api/feedback`.
- **`add_audio` is untested end-to-end** — the E2E ran without a music track.
- **Transitions, effects, and keyframes are not exposed as tools yet.** The agent can cut, place, scale, and caption; it cannot yet dissolve, colour-grade, or animate. Those are the obvious next tools.

---

### 2026-07-14 — Baseline audit
- Mapped the full repository. Established that the root is a scaffold (~400 lines of app code) and `classic/` is an untracked 91k-line reference implementation.
- Documented classic end to end: `EditorCore` + 12 managers; the 120,000-tick `MediaTime` invariant and rational `FrameRate`; the scene/track/element model and its `params` bag; the wgpu compositor and its `FrameDescriptor` wire format; deterministic frame-by-frame mediabunny export sharing the preview render path; local-first IndexedDB+OPFS storage at schema v31; the Command/Action layering; six Rust crates.
- Identified the team's two machine-enforced conventions (single destructured options object, enforced by a custom ESLint rule *and* a Rust proc-macro compile error; plus maximum type safety) — **adopt from commit one.**
- Logged 8 defects in the scaffold (§3). Headline: **no linter is configured despite the config and a 300-line rulebook claiming one is**, and `moon ci` will likely fail on `web:test` (no test files).
- Verified a real bug in classic's storage migration runner (§4.6) — positional args against a destructured-object signature. Do not port that path.
- Corrected an early misreading of my own: classic renders through a **wgpu compositor mounted directly in the DOM**, not Canvas2D with a GPU fast path.
- **No goal assigned yet.** Awaiting direction.
