# PGSimCity implementation teardown

Source read: `NikolayS/PGSimCity` @ `main` (shallow clone, 2026-07-27), version `0.9.0`,
Apache-2.0. ~53.5k lines of non-test TypeScript under `src/`, 42 test files / 210 tests.
Everything below cites files in that repo unless stated otherwise.

---

## 1. Stack and build

| Thing | Value |
| --- | --- |
| Runtime deps | `three` ^0.185.0 — **the only one** (`package.json`) |
| Dev deps | `typescript` ^5.9.2, `vite` ^7.1.5, `vitest` ^4.1.10, `@types/three` |
| Target | ES2022, TS strict, WebGL2 required (hard gate in `src/main.ts:88-96`) |
| Scripts | `dev`, `build`, `typecheck` (`tsc --noEmit`), `test` (`vitest run`), `preview`, `test:layout` |
| Node | 22 in CI, "20 or newer" locally |

`vite.config.ts`:

- `base: './'` — relative asset paths, which is what makes the GitHub Pages
  subpath (`/PGSimCity/`) work without a `base` env dance.
- Build-time constants via `define`: `__PGSIMCITY_VERSION__` (from package.json) and
  `__PGSIMCITY_GIT_SHA__` (from `PGSIMCITY_GIT_SHA` / `GITHUB_SHA`, else `git rev-parse
  --short=7`, else `'unknown'`). Surfaced through `src/core/build.ts` as `BUILD_LABEL`.
  **This is the pattern for our "print the pinned Temporal version in the footer" rule** —
  except ours is a data constant, not a git sha.
- **Two rollup inputs**, `city` (`index.html`) and `observability`
  (`observability/index.html`), and the second is only added *if its whole entry graph
  exists on disk* (`allExist('./observability/index.html', './src/observability/main.ts',
  './src/observability/style.css')`). Rationale in-file: a rollup input pointing at a
  half-landed file fails the build and takes down the deploy for everyone.
  `PGSIMCITY_ENTRIES=city` forces the single entry.
- `test.exclude` covers `.claude/**` because agent worktrees inside the repo caused vitest
  to run another agent's red tests.

CI (`.github/workflows/ci.yml`): PR-only, checkout → setup-node 22 with npm cache → `npm
ci` → `npm run typecheck` → `npm run build`. Note **CI does not run `npm test`** despite
CLAUDE.md claiming "CI runs `npm test`" — a documented rule that the workflow doesn't
actually enforce.

Pages (`.github/workflows/pages.yml`): push to `main` + `workflow_dispatch`; permissions
`contents: read, pages: write, id-token: write`; `concurrency: {group: pages,
cancel-in-progress: false}`; build job uploads `dist` via `actions/upload-pages-artifact@v3`;
separate `deploy` job with `environment: github-pages` and `actions/deploy-pages@v4`.
Straight copy for us.

---

## 2. Layering and the import rules

```
src/
  core/           contracts (types.ts), bus, registry, theme/themes, timebase, util, analytics, build, destinations
  sim/            pure TS PostgreSQL model — model.ts, scenarios.ts, mvcc.ts. Never imports three.js.
  world/          three.js geometry, one module per district + layout.ts (geography SoT)
  engine/         renderer, camera, flows, labels, picker, collision, walk, water, roads, audio
  ui/             HUD, inspector, tour, search palette, context menu, controls, boot, content
  observability/  second page over the same sim
```

Rules, stated in `CLAUDE.md` and actually observable in the code:

1. `src/core/types.ts` defines `SimState` — the sole contract between simulation and
   presentation.
2. `sim` owns and mutates state; **`world` may read `SimState` but never mutates it**.
   `WorldModule.update(dt, sim, t)` takes state as a read-only argument.
3. `world/layout.ts` is the single source of truth for geography. No district hard-codes a
   coordinate another district also needs.
4. `ui` and `observability` explain state; they never become a second simulation.
5. Cross-cutting files (`engine/renderer.ts`, `main.ts`, `world/layout.ts`,
   `core/types.ts`) are declared worktree-only for agents.

Debug surface: `window.PGSIMCITY` (alias `PGCITY`) = `{ sim, registry, bus, rig, gfx,
flows, walk, collision, audio, water, setThemeMode, themeMode }` — this is what the
screenshot driver uses to stage views (`bus.emit('focus', {id})`, `sim.setKnob()`,
`sim.runScenario()`). **Steal this wholesale**; it is what makes headless visual
verification possible.

### Boot order (`src/main.ts`, 445 lines — the whole wiring is one readable file)

`analytics → WebGL2 probe → bus → audio → registry → theme → renderer → camera rig → sim →
WorldContext → sky/ground → districts (shmem, access, clients, backends, wal, storage,
maintenance, replication, planner, continuity) → scene.updateMatrixWorld(true) → collision
build → water → walk controller → roads → flows → labels → picker → UI modules → bus wiring
→ resize → frame loop → finishBoot`.

Each step awaits `presentBootStep()` (`src/ui/boot.ts`) which paints a progress bar — the
awaits also yield to the browser so the bar actually renders. `BOOT_STEPS` is a named
record, so boot is a scripted narrative rather than a spinner.

Teardown is explicit and complete (`dispose()` over every module), hooked to `pagehide`
with a `persisted` check for bfcache and to `import.meta.hot.dispose`.

---

## 3. `world/layout.ts` — geography as source of truth

955 lines. Structure:

- A header ASCII map of the city with the z-bands of every district. Y is up, north is −Z,
  east is +X, **1 world unit ≈ 1 metre, a person is ~1.8**. That last convention is what
  makes the walk mode and the camera distances all mean something.
- `CITY` — dimensional constants per district (deck 156×124, buffer grid pitch 2.9, backend
  row z = −130 span 224, storage at y = −52, OS page cache slab at y = −24, the *durability
  boundary* at y = −27.4, the excavation `pit` 118×104, fog near/far).
- `ANCHOR` — named `[x,y,z]` points (`cityCenter`, `postmasterDoor`, `walBuffers`,
  `walVault`, `archiver`, `vacDepot`, `landfill`, `lockManager`, …) plus `v3()` / `at()`
  helpers. **Cross-district positions are anchors, never literals.**
- Index → position functions: `backendX(i)`, `backendPos(i)`, `backendPid(i)`, `conduitX(i)`,
  `tableX(t)`, `tablePos`, `indexPos`, `walSegZ(i)`, `vacBayPos(i)`, `bufferTilePos(idx)`.
- `TABLES: TableDef[]` — the demo schema (name, rows, page count, indexes, colour), and
  `N_TABLES`.
- `rid` — route-id *constructors*: `rid.query(i)`, `rid.ioRead(t)`, `rid.vacGo(t)` … so the
  sim can name a route without importing geometry.
- `ROUTES: Record<string, RouteDef>` built by a local `route()` helper: control points +
  colour + speed + size + `visible` (draw a faint static road) + `roadOpacity` + `tension` +
  `linear`. Per-backend and per-table routes are generated in `for` loops. Accessors
  `routeCurve/routePoint/routeTangent/routeLength` wrap a cached `CatmullRomCurve3`.
- `DISTRICT_BOUNDS: Record<string, Bounds>` — used by the minimap and district dimming.

The load-bearing idea: **the road network is data, defined once, consumed by both the
particle system and the static road geometry.** A flow is `bus.emit('flow', {route:
rid.query(3), count: 4})` — the emitter never knows where anything is.

---

## 4. The simulation and the tick loop

`src/sim/model.ts` is 3982 lines, one closure `createSim(bus): SimApi`. Public surface
(`core/types.ts:563`):

```ts
interface SimApi {
  state: SimState
  update(dt): void                 // dt already scaled by timeScale
  setKnob<K>(key, value, source?): void
  runScenario(id: string | null): void
  request(kind: QueryKind, table: number, opts?): void   // the "run a query" entry point
  setTraceMode(mode: TracePlayback): void
  endTrace(): void
  reset(): void
}
```

`SimState` (`types.ts:533`) is one flat record: `t` (sim seconds), `realT`, `knobs`, `xid`,
`xminHorizon`, `backends[]`, `buffers`, `wal`, `checkpoint`, `bgwriter`, `autovac`,
`tables[]`, `replication`, `locks[]`, `stats` (with ring-buffer histories for sparklines),
`scenario`, `trace`.

### Timebase (`src/core/timebase.ts`, 52 lines — small and worth copying verbatim in spirit)

Three separate clocks, deliberately:

- `MAX_VISUAL_DELTA_SECONDS = 0.1` — animation delta clamp.
- `MAX_WALL_DELTA_SECONDS = 2` — accepted wall time (software renderer can stall 2 s).
- `MODEL_STEP_SECONDS = 1/30` — **the model only ever advances in fixed steps**.
  `createFrameTimebase(sim.update).advance(elapsed, paused, timeScale)` accumulates a
  remainder and runs `floor(remainder / STEP)` steps of `STEP * timeScale`; pausing drops
  the fractional backlog. So the model is frame-rate independent and deterministic;
  `timeScale` scales the *size* of each step, not the count.
- `simulationAnimationDelta(dt, paused, timeScale)` gives the world modules their own
  "city dt", so animation stops on pause but camera/UI keep running on real dt.

`update()` itself sub-steps with `MAX_STEPS = 20` and `STEP_MAX = 1/30`.

### Frame loop (`main.ts:347`)

```
timer.update() → rawDt / dt / elapsed
1. frameTimebase.advance(elapsed, paused, timeScale)     // model
2. rig.update(dt); walk.update(dt); water.update(cityDt) // camera first
   LOD bucket from rig.altitude → module.setDetail(0|1|2) only on change
3. modules[i].update(cityDt, s, s.t); flows.update(cityDt); picker.update(dt)
4. gfx.render(dt, rawDt); labels.update/render
5. setCompassCamera(...); ui[i].update(dt, elapsed)
```

LOD buckets: `alt > 420 → 0`, `> 150 → 1`, else `2`; walk mode forces 2.

### How the numbers are scaled (`model.ts` header — the most transferable page in the repo)

Two declared distortions:

1. **Time is stretched ~100× for anything sub-second** (`MODEL_TIME_STRETCH = 100`,
   exported so the UI can disclose it). Durations are a *monotone* stretch, so ordering and
   ratios stay true (plan > parse; `exec_io` dwarfs `exec_cpu` on a miss). **Rates** (tps,
   bytes/s, LSNs) are *not* stretched and are reported in real units.
2. **The city is a scale model.** 1024 sampled buffer frames, 16 backend slots, 14 WAL
   segments. One trip through the backend state machine carries `batch` transactions, and
   all work is multiplied by `batch` so the pool and WAL feel real pressure.
   `batch = tps / NOMINAL_TRIPS` — a **fixed function of the offered rate, deliberately not
   a controller**. The header documents the bug they had when it *was* a controller: sizing
   batch from measured trip rate closed a feedback loop that cancelled every bottleneck in
   the model, so achieved tps tracked the knob even with `shared_buffers = 32`. Achieved
   throughput must be an **output**, not a tracked target.
   Corollary they call load-bearing: page visits are *sampled* (`MAX_VISIT_PAGES`) for the
   animation, but **time is charged on the unsampled amount** — otherwise every transaction
   past the cap is free.

There is a `src/sim/honesty.test.ts` and a `test/wrongnumbers.test.ts`; the model uses a
seeded RNG (`makeRng` in `core/util.ts`) so tests never depend on wall clock or GPU.

### The "run a query" trace flow

The most directly relevant mechanism for us — it is how a single logical unit of work is
followed through the whole city.

- Trigger: HUD button / keyboard → loose bus `'trace:open'` → the tour module's statement
  picker (`openTracePicker()` in `src/ui/tour.ts:579`) → user picks a
  `QueryKind × table × playback` → `bus.emit('trace:run', {statement, table, playback})` →
  `sim.request(kind, table, opts)`.
- The model *doesn't* create a special object: it flags the **next real backend trip** as
  the traced one. `SimState.trace: TraceRecord` carries `slot`, `sql` (from `sqlFor(kind,
  ti)`), current `stop`, `stopT`, a `visited` **bitmask** over stops
  (`traceStopBit(stop)`), and accumulated facts: `rowsSent`, `buffersHit`, `buffersRead`,
  `walBytes`, `walFpiBytes`, `deadMade`, `trips`, `lastPlan*`.
- `TraceStop = connect | parse_plan | fetch | work | wal | commit | send | done | blocked`.
  `traceStopFor(backendState)` maps the backend state machine onto those stops, so the
  trace is a *view of the real state machine*, not a parallel script.
- `TracePlayback = 'step' | 'slow' | 'live'` — playback is implemented by the model
  (`setTraceMode`), not by the UI, so the camera and the card can't drift out of sync.
  `src/ui/trace-dwell.ts` + `trace-copy.ts` own the dwell timing and the prose.

**For TemporalSimCity this is the direct analogue of "follow one workflow execution".**

---

## 5. Rendering

`src/engine/renderer.ts` (913 lines). Two rendering models, one pipeline:

- **Night** (authoring baseline): matte PBR structure lit by cold key+fill; meaning is
  `neon()` (`toneMapped: false`, emissive > 1) and is the *only* thing above the bloom
  threshold (0.85).
- **Day**: sun key with real shadows, hemisphere fill, bloom effectively off, tone mapping
  switches ACESFilmic → **Khronos PBR Neutral** (ACES at noon exposure washes saturated
  colour to pastel), structure cel-shaded and ink-outlined.

The colour-pipeline note is the kind of thing that costs a week to rediscover: on the
composer path `RenderPass` draws into a HalfFloat target, and three.js forces
`NoToneMapping` + LinearSRGB whenever `_currentRenderTarget !== null`, so the buffer stays
linear HDR for `UnrealBloomPass` to threshold and `OutputPass` applies tone mapping + sRGB
once at the end. Consequence: **the same renderer settings are valid on both the direct and
composer paths; never toggle `renderer.toneMapping` when switching quality.**

Passes: `EffectComposer` + `RenderPass` + `UnrealBloomPass` + `SMAAPass` + `OutputPass`,
all from `three/examples/jsm`.

Quality: `QualityLevel = low | reduced | medium | high | ultra`, default `high`.
`QUALITY_PRESETS` set `pixelRatio` (DPR cap 1/1/1.5/2/2), `bloom`, `shadows`,
`maxParticles`, `maxLabels`, `antialias`. **Adaptive**: drop a level if fps < 45 for 3 s,
raise if fps > 58 for 12 s, with a 3 s warmup (shader compile) and 4 s settle after any
change. The `quality` object is mutated in place so consumers can hold the reference; a
toast offers the user the level change with an action button.

Theme (`core/theme.ts` 1057 + `core/themes.ts` 666): `COLOR` is a **live object mutated in
place** by `setThemeMode()`. The city is always *built* in night values and translated to
day, because every night value has a day translation but not vice versa. Documented trap:
a module that snapshots `const RED = COLOR.bufDirty` at import time holds a night value
forever. `ThemeApi` is also the shared cache — `mat()`, `neon()`, `line()`, `edges()`,
`textTexture()`, `box()`, `cyl()` — which is how they keep material/geometry counts sane.
Rule: **only emissive > 1.0 crosses the bloom threshold, so glow always carries
information**; hue is semantic per district and never reused for looks.

`index.html` sets `data-theme` before first paint from `localStorage` in an inline script,
and syncs `color-scheme` + `theme-color`.

---

## 6. The particle system (`src/engine/flows.ts`, 643 lines)

Every moving thing in the city — a query, an 8 KiB page, a WAL record, a dead tuple — is
one instance of **one `InstancedMesh`**, i.e. one draw call for all city traffic.

- Routes are **baked once** to a polyline: `SAMPLES = 96` points, plus per-sample unit
  tangent, horizontal binormal, normal, and cumulative arc length. Reason given:
  `CatmullRomCurve3.getPointAt()` walks an arc-length LUT and allocates internally —
  unusable thousands of times a frame. Lookup is `segFor(cum, d)` + lerp.
- Fixed pool with a free list; `PENDING_CAP = 512` staggered emissions, oldest dropped on
  overflow; `MAX_BURST = 256` sanity clamp. `flows.active` / `flows.dropped` are exposed to
  the HUD.
- Zero allocation in `update()`: every Vector3/Quaternion/Matrix4/Color is module-scope
  scratch.
- `COLOR_GAIN = 2.0` pushes instance colours past the bloom threshold. `FADE_IN 0.05` /
  `FADE_OUT 0.12`.
- **Meaning is carried by bulk, never elongation**: per-`FlowKind` width/length tables
  (`KIND_W`, `KIND_L`) with no kind allowed a length:width ratio above 1.28, so nothing
  reads as a dart however fast it moves. A page is a wide pallet; a stats ping is a small
  crate.
- Lateral/vertical lane offsets + `spread` jitter keep bidirectional traffic on one spine
  visually separated (query and result share the same conduit, reversed).
- `reduceMotion()` is respected.

---

## 7. Labels (`src/engine/labels.ts`, 1295 lines) — the "district plate + header + +N" system

Uses `CSS2DRenderer`/`CSS2DObject` into `#labels-root`. Four mechanisms:

1. **Zoom hierarchy**, scored per-anchor against *its own* distance to camera, so the far
   side of the city stays coarse: `city` (model name, above ~420 units) → `district` (one
   chip at the centroid of its members) → `tier 0` landmarks → `tier 1` structures →
   `tier 2` details. Neighbouring levels overlap in a fade band. A district whose name a
   member already carries **promotes that member** instead of drawing a duplicate chip.
2. **Screen-space collision**: project candidates to pixels, sort by priority, place
   greedily against a uniform grid of already-placed rects; a chip that doesn't fit at its
   home offset is tried at **seven alternates** (other side, lifted, below) before being
   dropped, and its leader line is redrawn to whichever corner ends up nearest the anchor.
3. **Hysteresis**: a shown label outranks an equal-tier hidden one, tolerates a few pixels
   of real overlap, and cannot be dropped inside a minimum dwell; a hidden one needs clear
   space plus a cooldown to return. Without this, boundary labels strobe as the camera
   drifts — worse than overlap.
4. **Collapse, never silence**: anything on screen that is *not* labelled (level-gated or
   collided out) is counted into its district's **"+N" pill** (`lbl__more`, `has-more`), so
   an area can never read as empty. This exists because `wal_buffers` and CLOG sat on the
   plaza unlabelled at normal viewing distance with nothing to say they were there.

Cost control: the full pass runs at ~9 Hz, never per frame; chip boxes are measured once at
construction in both forms and re-measured only when content width really changes; **every
read in a pass happens before every write** (one layout per pass); between passes the
compositor interpolates chip offsets via a CSS transform transition.

Detail levels per chip (`engine/label-detail.ts`, 30 lines): `Name → Readout → Role`.
`Readout` when the component has a live readout and distance ≤ 130; `Role` when selected or
hovered; at most `EXPANDED_LABEL_CAP = 2` expanded at once, prioritised
selected → hovered → nearest screen centre.

`ComponentDef.readout?: (s: SimState) => string` is the hook — each component publishes one
live one-line metric, used by both the label and the inspector header.

---

## 8. Picking, selection marker, collision

**Registry** (`core/registry.ts`, 87 lines): `register(def)` indexes by id and by object;
`resolve(hit)` walks up parents (guard 64) to the nearest registered ancestor, so a district
can register a whole group and still pick precisely; `roots()` is the raycast candidate set;
`district(id)`, `tier(t)`, and a fuzzy `search(q, limit)` for the palette. Duplicate ids
warn and the later wins.

**Picker** (`engine/picker.ts`, 703 lines): raycast at **25 Hz**, never while the camera is
being dragged, never when the pointer is over the HUD; emits `hover` / `select` / `focus`.
Click discrimination: ≤ 5 CSS px travel, ≤ 350 ms press, double-click < 320 ms (double-click
= fly to). The selection marker is deliberately *not* a wireframe box or corner brackets —
it is an architect's setting-out drawing: ground circle, squared footprint, crown repeated
at roof level, two staffs, dimension lines on the two exposed sides, and a soft ground
light that breathes on a 3.2 s period. Hover is the same drawing at ⅓ intensity.
Registry AABBs are **not trusted**: districts park unused instanced-mesh instances far below
the world (postmaster's box runs y = −1000 → 47), so an impossible box is split into
children, and the survivors are unioned.

**Collision** (`engine/collision.ts`, 1164 lines) exists for walk mode. Built **once** after
`scene.updateMatrixWorld(true)`, from registry boxes, bucketed into a uniform grid.
Three refinements worth knowing:
- oversized *containers* are **split** by recursing into children (max depth 4);
- oversized *leaves* (districts that merge structure into one childless mesh) are
  **voxelised**: triangles binned into a coarse XZ grid, each occupied cell tightened and
  re-classified, so a painted apron stays flat rather than becoming a kerb;
- explicit exclusions (`DEFAULT_EXCLUDE_IDS` + `shmem.deck` in `main.ts`): the ground is a
  *walkable*, not a blocker; the 1024 buffer tiles change height every frame so they are
  walked *through* — and only the deck slab, not the whole shmem group, is a walkable,
  because a floor that moves rises into your feet.

---

## 9. Camera (`src/engine/camera.ts`, 1251 lines)

`CameraMode = orbit | fly | focus | tour | walk`.

- **orbit** — pivot in world + spherical offset. Drag is 1:1, release glides (`SPIN_DECAY`
  / `PAN_DECAY` = 13 /s ≈ 0.25 s settle), and the wheel **dollies toward the cursor ray,
  not the pivot** — described in-file as the difference between a CAD toy and an
  architectural walkthrough. `MIN_DIST 24`, `MAX_DIST 1650`, `PHI` 0.03→3.05 (you can get
  under the plate to look up into the excavation).
- **fly** — pointer-locked yaw/pitch, accelerated view-space translation, 4→400 speed,
  shift boost ×3, alt precision ×0.25.
- **focus** — scripted tween to a `FocusSpec {target, distance, dir?}`, default 1.05 s,
  with a 25° upward framing bias when the direction is auto-derived.
- **tour** — scripted CatmullRom path + a **parallel look-at path**, `flyPath(points,
  lookAt, duration): Promise<void>`, 18 % ease at each end.

**The trick worth stealing outright:** the orbit state is kept continuously valid *during*
scripted moves — the pivot is re-derived from the live camera transform every frame — so
`release()` is a pure mode flip with zero snap. The user can grab the camera at any instant
of any animation and nothing jumps. `update()` allocates nothing (all scratch hoisted).

Presets: `home()` (a named establishing shot from the north-west, `HOME_POS = (-218, 216,
-342)` looking at `(-18, 0, -16)`) and `plan()` (straight down, 0.1 tilt, framed for the
Slonik outline, with a HUD vertical allowance of 152 px). `reduceMotion()` honoured.
Mode ping-pong between HUD and rig is guarded by an `applyingMode` flag in `main.ts:279`;
leaving walk mode flies back out to a vantage point behind the walker rather than cutting.

---

## 10. UI layer

`UiContext` (`ui/uikit.ts`) is tiny: `{bus, sim, registry, getFps, getQuality,
getFlowStats, getAudioState?}`. `UiModule` is `{update(dt, elapsed?), dispose()}`. Every
module builds DOM with the same `el()/setText()/setClass()/icon()/metricTile()/sparkline()`
helpers, and **every DOM write goes through `setText`/`setClass` so an unchanged value
costs nothing**.

- **HUD** (`ui/hud.ts`, 1712 lines): four surfaces in one module because they share a clock
  and a keymap — `#hud-top` (identity + health, **five vital signs with sparklines**,
  checkpoint countdown, tool buttons), `#hud-bottom` (play/pause, speed, reset, scenario
  rack), `#toast-stack`, `#compass` (2D minimap of `DISTRICT_BOUNDS` with the camera's view
  cone; `main.ts` pushes camera x/z/yaw in via `setCompassCamera()` each frame). Owns the
  single global keydown handler for *application* keys; camera keys stay with the rig.
  Escape is offered to overlays first via a shared `{handled}` payload, so the palette
  closes before the tour behind it.
- **Inspector** (`ui/panel.ts`, 669 lines, `#hud-right`): one component at a time — live
  metrics **above** the prose ("they are the reason any of the prose is interesting"),
  then sections, then the GUCs that change its behaviour, then "Go deeper". Refresh 6 Hz.
  Content lives in `ui/content.ts` + `ui/docs-*.ts` as `ComponentDoc` records: `tldr`,
  `sections[]`, `metrics[] {label, get(s), hint}`, `knobs[]`, `see[]`, `source[]`, `refs`.
  Citation discipline is encoded in the *types*: `DocRef.url` is optional because a guessed
  link is a fabricated citation; `verified?: false` renders a visible "†"; `BookRef`
  deliberately has **no** `url` field so the renderer cannot link a paper book.
- **Tour** (`ui/tour.ts`, 1073 lines): 14 chapters, one story (connect → process → plan →
  page read → WAL → commit → checkpoint → MVCC corpse → vacuum → xmin horizon → stream →
  lag → whole city). A chapter is `TourChapter {id, title, body, focus? | camera?,
  duration, knobs?, scenario?}` plus two private extensions in the runner: `at: [second,
  Partial<Knobs>][]` for mid-chapter knob beats and `look: [second, componentId][]` for
  mid-chapter camera moves. Chapters snapshot `{knobs, scenario}` on entry and restore on
  exit, so the tour hands the city back exactly as it found it. The same lower-third card
  renders scenario narration when the tour is not running — "the city only ever speaks in
  one voice".
- **Command palette** (`ui/search.ts`, 711 lines): `/` or Ctrl+K, searches five kinds —
  components (`registry.search`), settings (activating scrolls the console to the dial and
  flashes it), scenarios, tour chapters, anatomy instruments.
- **Context menu** (`ui/context-menu.ts`): right-click, made possible because rotation moved
  to shift-drag. Actions are derived from the hit (`contextActionIds`) and never invent a
  command that has no instrument behind it.
- **Zoom context** (`ui/zoom-context.ts`): below 40 units orbit distance a "You're looking
  at <name>" caption + "Back to city" appears; releases at 46 (hysteresis); centre-screen
  raycast at 5 Hz.
- **Boot** (`ui/boot.ts`), **help** (`ui/help.ts`), **controls/console** (`ui/controls.ts`,
  the GUC dials with collapsible sheets), **touchpad** (`ui/touchpad.ts`, on-screen walk
  controls), **anatomy** (`ui/anatomy.ts`, 1854 lines, lazily `import()`ed after first
  frame — the page and data-directory instruments), **legal** (`ui/legal.ts`,
  `TRADEMARK_NOTICE` / `NO_EA_CONTENT`), **mode-exits** (`ui/mode-exits.ts`, one registry of
  "how do I get out of this mode" ids shared by every overlay).

`BusEvents` (`core/types.ts:609`) is the whole UI protocol and is deliberately **frozen** —
`flow`, `focus`, `select`, `hover`, `knob`, `scenario`, `toast`, `narrate`, `tour:*`,
`trace:*`, `panel:open`, `anatomy:open`, `camera:mode|preset|gesture`, `quality`,
`sim:reset`, `fx:pulse`, `checkpoint:*`, `ui:layout`. Anything not in the contract travels
as a *loose string channel* on the same bus (`emitLoose`, e.g. `ui:palette`,
`audio:toggle`) with a local `LooseBus` cast. Pragmatic, and it keeps the typed surface
honest.

---

## 11. Scenarios (`src/sim/scenarios.ts`, 434 lines)

`ScenarioDef {id, name, blurb, icon, knobs: Partial<Knobs>, focus?, duration, beats?}` where
a beat is `[atSecond, title, body]`, fired via the `narrate` bus event and held for
`SCENARIO_NARRATION_SECONDS = 9`. Shipped: `steady-state`, `checkpoint-storm`,
`cache-thrash`, `bloat-and-vacuum`, `replication-lag`, and others. Each is "a set of knobs
that provokes one specific, real behaviour, a place to point the camera, and narration
timed to when the city actually shows it".

Copy rules stated in the file header: say what is happening, say why, say what an operator
would do; no hedging, no marketing; the reader is a strong engineer who has never had to
run a database. The beats also tell you which real view to check on your own server
(`pg_stat_checkpointer.num_requested` vs `num_timed`) — the "leave and grep the docs" move.

The `focus` field must be a registered component id, and the header lists the valid ones;
this is only enforced by a test, not by the type.

---

## 12. The second entry point (`src/observability/`, ~4.5k lines)

A separate page ("the symptom console") over **the same `createSim(bus)`**: you pick a
complaint in plain language, it puts the model into a state that produces it, then walks a
decision tree to the `pg_stat_*` view and column that proves it, reading live rows out of
the running model at each step. Modules: `catalog.ts` (the views), `paths.ts` (symptoms +
decision nodes), `views.ts` (projections of `SimState` into rows), `collector.ts`,
`flow2d.ts` (2D flow view with URL-serialisable query), `shell.ts` (exit back to the city).

The relevant lesson: **the sim is reusable headless**. Any second surface — a 2D schematic,
a docs page, a test — can `createSim(createBus())` and drive it without three.js. That is
what the `sim` / `world` boundary buys.

---

## 13. Testing and tooling

210 tests over 42 files, `vitest`, jsdom-ish DOM helper in `test/dom.ts`. Categories:
- **model correctness**: `sim/model.test.ts`, `honesty.test.ts`, `knob-response.test.ts`
  (directional properties: raise this knob, that metric must move this way),
  `sample-scale-boundary.test.ts`, `workingset.measure.test.ts`, `mvcc.test.ts`
- **geometry as fact**: `engine/collision.test.ts`, `camera-floor.test.ts`,
  `labels-occlusion.test.ts`, `labels-text.test.ts`, `world/slonik.test.ts` (silhouette),
  `swimming.test.ts`
- **UI invariants**: `mode-exits.test.ts`, `tour-visibility.test.ts`, `ui-bugs.test.ts`,
  `context-menu.test.ts`, `zoom-context.test.ts`, `trace-dwell.test.ts`
- **content/compliance**: `trademark-notice.test.ts`, `source-link.test.ts`,
  `analytics-entrypoints.test.ts`, `build-metadata.test.ts`, `wrongnumbers.test.ts`

Stated test doctrine (CLAUDE.md): assert behaviour and properties, never existence or
opaque snapshots; exact answers for formulas, durable *directional* properties for scaled
simulation behaviour; seeded RNG only; never depend on wall clock, browser or GPU when the
claim is pure. Red/green TDD is mandatory for bug fixes, motivated by a real regression
(the Slonik plate silhouette degraded over four commits with no test able to bisect it).

Tools: `tools/shoot.mjs` (headless CDP screenshot driver, `CDP_PORT` 9500–9900, software
WebGL at 1–3 fps so 45–70 s settle, optional JS evaluated before the shot), guarded by a
**three-slot directory semaphore** at `/tmp/claude-1000/cdp-gate` with a ten-minute reap
(`tools/reap.sh`) — ten concurrent browsers OOM-killed the machine twice;
`tools/plot-plate.mjs` (prints silhouette/bbox/segment count with no browser — build the
cheap feedback loop before the expensive one); `tools/verify-hud-layout.mjs`.

---

## 14. Licence and attribution obligations

PGSimCity is **Apache-2.0** (`LICENSE`, `NOTICE`, `package.json`), Copyright 2026 Nikolay
Samokhvalov.

If we borrow structurally (and we will — architecture, `layout.ts` pattern, label placement
algorithm, flow pool, camera rig, boot sequence, workflows), the obligations are:

1. **Keep the licence and the notice.** Apache-2.0 §4: include a copy of the licence with
   any distribution of derivative work, keep all copyright/patent/trademark/attribution
   notices in the source we reuse, and **carry forward the `NOTICE` file's relevant
   attribution text** in our own `NOTICE`.
2. **State the changes.** §4(b): files we modify must carry prominent notices that we
   changed them. In practice: a header line in any file that is a recognisable derivative,
   plus one honest paragraph in our README.
3. **Apache-2.0 is compatible with re-licensing our own additions** under Apache-2.0
   (simplest) — recommendation: **ship TemporalSimCity as Apache-2.0** so no compatibility
   analysis is ever needed, with a `NOTICE` that credits PGSimCity by name and URL.
4. **Attribution beyond the licence is the decent move anyway** — PGSimCity is the reason
   this project has a shape. A visible "Inspired by PGSimCity" credit in the footer and the
   README, linking the repo and the live site.
5. **Trademark discipline is a copyable pattern, not an obligation we inherit.** Apache-2.0
   §6 grants no trademark rights. PGSimCity's `NOTICE` explicitly disclaims affiliation with
   the PostgreSQL project and states that the elephant mark is used only to refer to
   PostgreSQL; `package.json.description` and every `og:description` also carry "Not
   affiliated with Electronic Arts" (the *SimCity* name), and `test/trademark-notice.test.ts`
   asserts the notice is present. **We need the same two disclaimers, differently aimed:
   Temporal Technologies Inc. and Electronic Arts.** That is the model for map ticket
   "Trademark and attribution" in Not-yet-specified.
6. Do **not** copy the PGSimCity elephant/Slonik artwork, the `og.png`, or the favicon —
   those are PostgreSQL-mark-adjacent and specific to that project. The *technique*
   (ground plate cut to a silhouette) is fair game; the mark is not.

---

## 15. What we steal, adapt, and drop

**Steal as-is (structural, low risk, high value):**

- The five-layer split and the two hard rules: `sim` never imports three.js; `world` reads
  `SimState` and never mutates it. Meeting point is one `types.ts`.
- `layout.ts` as the single geographic source of truth: `CITY` dims, named `ANCHOR`s,
  index→position functions, route table, `DISTRICT_BOUNDS`. 1 unit = 1 m.
- Routes as data consumed by both the flow pool and the drawn roads.
- The fixed-step timebase (`MODEL_STEP_SECONDS`, remainder accumulator, three separate
  deltas) — this is what makes the sim testable and pause/speed correct.
- One `InstancedMesh` flow pool with baked polylines, free list, zero per-frame allocation.
- The label system: zoom hierarchy → screen-space collision with alternates → hysteresis →
  **"+N" collapse pill**. This is ~1300 lines of hard-won behaviour we should not reinvent.
- The camera rig's "orbit state stays valid during scripted moves" invariant.
- `WorldContext` / `WorldModule` / `ComponentDef` (with `readout`, `tier`, `focus`) and the
  `Registry` with parent-walking `resolve()`.
- The typed bus with a frozen `BusEvents` contract.
- `window.<APP>` debug handle + a headless screenshot driver with a concurrency gate.
- Build-time version constant printed in the footer, `base: './'`, the two Pages workflows.
- Conditional rollup inputs so a half-landed second page cannot break the deploy.
- Test doctrine: seeded RNG, directional properties for scaled behaviour, geometry
  assertions, a content-compliance test for the trademark notice.

**Adapt:**

- **Scaling doctrine.** Our version of "time is stretched ~100×, rates are real" and "batch
  is a function of offered rate, never a controller" needs restating for Temporal: workflow
  task latencies, long-polls, and history event counts are the sub-second things; task
  queue backlog, tasks/sec and shard counts are the rates. The `batch` anti-pattern warning
  transfers directly — **achieved throughput must be an output**.
- **Trace → workflow follow-cam.** `sim.request()` flagging the *next real trip* and
  exposing a `TraceRecord` with a `visited` bitmask over named stops is exactly the shape of
  "follow one workflow execution through gate → depot → river → records hall". Our stops are
  Temporal's real state transitions, not invented ones.
- **Tour chapter contract** (`focus`/`camera`, `duration`, `knobs`, `scenario`, plus private
  `at`/`look` beats, snapshot-and-restore on exit).
- **Scenario contract** (knobs + focus + timed beats) — our scenarios are things like worker
  fleet death, sticky-queue miss storm, history growth, partition.
- **Palette inversion.** They authored in night and translate to day; we are dark-first with
  bold neon, so we author in *our* night and — if we ever want a day mode — must decide the
  direction of translation *before* any district is authored. Their warning about
  import-time colour snapshots applies to us verbatim.
- **Inspector citation types.** `DocRef.url?` optional, `verified?: false` rendered as a
  visible "†", `BookRef` with no url field. Our "no Temporal facts from model memory" rule
  wants exactly this — plus a `sourceTag` field pinning the Temporal Server release the
  citation was read at.

**Drop or defer:**

- Walk mode + collision world + water + audio (~3.5k lines). PGSimCity's own camera notes
  say the aerial view is the teaching view; our map already locks "aerial, SimCity-style, no
  avatar". Collision only exists to serve the pedestrian, so dropping walk drops all of it —
  a large, real saving. (Ground-level camera is already parked in Not-yet-specified.)
- The Slonik/elephant plate silhouette and its plotter tool.
- Plausible analytics (their whole dependency-boundary section exists to police it) —
  a call for the destination spec, not a default.
- The second `observability` entry — worth *knowing* the sim is headless-reusable, but a v1
  concern at most.
- `anatomy.ts`-scale instruments (page/directory dissection) until the mechanism inventory
  says which Temporal object deserves one.

**Open questions this teardown raises for the map:**

- Their five vital signs + sparklines strip is the direct precedent for the parked
  "HUD / metrics strip" fog. The constraint worth carrying: a slot is earned by a number
  that *moves visibly* when a knob moves — that is what `knob-response.test.ts` asserts.
- They have no deep-link/shareable-state system in the city page (only `flow2d.ts` in
  observability serialises its query). If we want `?chapter=`/`?focus=`, we are designing it
  ourselves, not porting it.
- 53.5k lines / 42 test files is the size of the thing we are drawing inspiration from at
  v0.9. Our vertical slice must be explicitly scoped against that, not measured by it.
