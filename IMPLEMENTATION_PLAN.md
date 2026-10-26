# IMPLEMENTATION PLAN — "The Best Is Yet to Come"

> Audit date: 26 October 2026  
> Auditor: Senior Three.js / Creative Frontend Engineer  
> Status: **Phase 1 — Stability, Navigation & Performance Baseline — COMPLETE** (branch: `feature/stability-navigation`)

---

## Phase 1 Completion Record

> Implemented 26 October 2026. All changes are in `index.html` only. `original-prototype.html` was not touched.

| Task | Status | Notes |
|---|---|---|
| B1 — Tree placement | ✅ Fixed | Trees now sampled at uniform t values; true tangent perpendicular used for lateral offset. Seeded PRNG added for determinism. |
| B2 — Seasonal interpolation | ✅ Verified correct | Exponential smoothing is frame-rate independent (confirmed mathematically). Only optimization applied: pre-allocated `_blendCol` Color to eliminate per-frame GC allocation. |
| B3 — Dead code (`targetT`) | ✅ Removed | Variable was declared but never read. Only appears in an explanatory comment. |
| B4 — Unused material (`rockMat`) | ✅ Removed | Material declared, no geometry ever created. Only appears in an explanatory comment. |
| B5 — Idle camera sway | ✅ Fixed | Sway now uses transient `displayT` offset — `camT` itself is never mutated while parked. Amplitude 0.00012 t, period ~7 s. Respects `prefers-reduced-motion`. |
| B6 — Error reporting | ✅ Implemented | `showError()` helper writes to `#errorMessage` (styled red alert). Handles: WebGL unavailable, `content.json` fetch fail (including `file://` warning), bad HTTP status, no enabled stops. |
| B7 — Keyboard navigation | ✅ Implemented | `ArrowRight`/`ArrowLeft` navigate stops; `Enter`/`Space` opens current stop panel; `Escape` closes modal. Suppressed during travel and inside form controls. |
| B8 — Per-frame no-op loop | ✅ Removed | `dashGroup.children.forEach(d => d.visible = true)` was setting always-visible meshes to visible every frame. Loop eliminated. |
| Mobile: iOS safe-area | ✅ Applied | `viewport-fit=cover` added to meta; `.progress` and `.hud` use `env(safe-area-inset-bottom, 0px)` with responsive fallbacks. |
| Mobile: canvas touch-action | ✅ Applied | `touch-action: none` on `#scene` prevents browser intercepting canvas drags as scroll gestures. |
| Rendering: tab restore dt spike | ✅ Fixed | `visibilitychange` resets `lastFrame` on tab show; `dt` clamped to 100 ms maximum per frame. |
| Rendering: single loop guard | ✅ Confirmed | Single `requestAnimationFrame(render)` entry point; `render()` self-registers. No duplicate loop risk. |
| Seeded PRNG for scenery | ✅ Added | `mulberry32(0xBEEFCAFE)` used for all mountains and trees — layout is now deterministic across reloads. |

**Syntax check:** `node --check` on the extracted JavaScript block returns exit code 0 (no syntax errors).  
**Server test:** `python -m http.server 8000` served `index.html` (HTTP 200, 34 KB) and `content.json` (HTTP 200, valid JSON, 5 stops).  
**Browser test:** Not performed in this session — no headless browser is available in this environment. Visual verification must be done manually in Chrome/Safari/Firefox at `http://localhost:8000`.

### Phase 1 deferred items — resolved in Phase 2
- `InstancedMesh` for trees and mountains — ✅ completed in Phase 2 (see below).

---

## 1. Repository Inventory

| File | Size | Purpose |
|---|---|---|
| `index.html` | ~20 KB | Main experience (evolved prototype) |
| `original-prototype.html` | ~19 KB | Baseline prototype — preserved, not to be changed |
| `content.json` | ~5.5 KB | All editable story content and configuration |
| `README.md` | ~2.8 KB | Usage, editing, and publishing instructions |
| `assets/audio/` | empty | Reserved for ambient audio track |
| `assets/models/` | empty | Reserved for future GLTF models |
| `assets/photos/{career,celebrations,childhood,family}/` | empty | Reserved for family photo drops |
| `assets/future/` | empty | Reserved for AI-generated future illustration images |

All asset directories contain only `.gitkeep` placeholders. **No personal photos or media are present.**

---

## 2. How the Three.js Scene Is Constructed

### 2.1 Renderer
```
THREE.WebGLRenderer → antialias, no alpha
Pixel ratio clamped to 2
Shadow map: PCFSoftShadowMap, 1024×1024
Output: SRGBColorSpace
Tone mapping: ACESFilmicToneMapping at exposure 1.15
```
This is a solid, modern renderer configuration.

### 2.2 Scene Graph (in construction order)
1. **Background + Exponential Fog** — `scene.background` Color + `FogExp2` density 0.0095
2. **HemisphereLight** — sky `#c9dcff` / ground `#64513b`, intensity 2.0
3. **DirectionalLight (sun)** — position `(-70, 100, -130)`, warm gold, intensity 3.2, casts shadows
4. **Ground plane** — `PlaneGeometry(1000, 1000)`, receives shadows
5. **Road ribbon** (custom geometry — see §3)
6. **Dashed centre-line** — 260 `BoxGeometry` meshes
7. **Mountain ranges** — 6 `Group`s of cone meshes (330+ cones total)
8. **Pine trees** — 320 `Group`s of cylinder trunk + 3 cone tiers (1,280 geometries total)
9. **Sun disc** — `SphereGeometry(7)` with `MeshBasicMaterial`
10. **Milestone marker rings** — `TorusGeometry` + floating `SphereGeometry` orb per stop

---

## 3. How the Curved 3D Road Is Generated

### Spline construction
```javascript
// 241 control points, flat (y = 0), winding sinusoidally in X, linear in Z
for (let i = 0; i <= 240; i++) {
  const z = -i * 2.8;                                    // extends to z ≈ -672
  const x = Math.sin(i * 0.035) * 23 + Math.sin(i * 0.012) * 19; // dual-frequency wind
  points.push(new THREE.Vector3(x, y, z));
}
const path = new THREE.CatmullRomCurve3(points, false, 'catmullrom', 0.35);
```
The road is entirely **flat** — there is no vertical elevation in the spline. This is the single biggest visual limitation.

### Road surface (ribbon geometry)
A custom function `ribbon()` samples the spline at `N` points, computes the perpendicular side vector from the tangent, and emits two vertices per sample to form a quad strip. Width = 9 units, 1200 segments. This produces smooth, correctly-curved road geometry without the caps/normals issues of `TubeGeometry`.

---

## 4. How the Camera Follows the Road Spline

```javascript
function cameraPose(t, lookAhead = 0.035) {
  const p    = path.getPointAt(t);             // camera position on curve
  const look = path.getPointAt(t + lookAhead); // look-at point slightly ahead
  camera.position.set(p.x, p.y + 2.15, p.z);  // 2.15 units above road
  camera.lookAt(look.x, look.y + 1.5, look.z); // look slightly above horizon
}
```

Travel between stops uses an **ease-in/out cubic** (`ease(t)`) over a duration of `3000 + distance×2400 ms`, capped by `reducedMotion` detection (600 ms). The camera parameter `t` is a normalised `[0, 1]` value along the CatmullRom spline.

A micro-sway is applied while stopped (`camT += Math.sin(now * 0.00035) * 0.000001`) — this value is so tiny it has essentially **no visible effect**.

---

## 5. How Milestones Are Positioned and Navigated

Each stop's `roadPosition` value from `content.json` maps directly to spline `t`. The camera travels to `t - 0.025` (slightly before the marker) to create an "arriving" feel.

Markers are placed **off to the side** (alternating left/right, 7 units) using the curve tangent's perpendicular vector. Each marker is a torus ring (glowing gold) with a bobbing orb above it. The active marker pulses with higher `emissiveIntensity` (3.4 vs 1.5).

Navigation modes:
- **Next / Previous buttons** — sequential `goTo(index)`  
- **Progress dots** — direct jump to any stop  
- **Raycast click/tap on rings** — if same stop, opens modal; if different stop, travels there  
- **Modal "Continue"** — advances to next stop and closes panel  

---

## 6. How Seasonal Lighting Transitions Work

Five `seasonLooks` palettes are defined (indexed 0–4, matching `sceneType` order):

| Index | sceneType | Sky | Mood |
|---|---|---|---|
| 0 | beginning | Warm amber | Golden sunrise |
| 1 | building | Soft blue | Clear midday |
| 2 | family | Deep amber | Late afternoon |
| 3 | renewal | Cool grey-blue | Overcast/mist |
| 4 | future | Rich gold | Sunset/horizon |

Each palette blends: `scene.background`, `scene.fog.color`, `ground.material.color`, `sunDisc.material.color`, and `sun.color`.

In `index.html` (evolved version), blending uses **exponential smoothing** per frame:
```javascript
const alpha = 1 - Math.exp(-dt * 1.2);
scene.background.lerp(new THREE.Color(s.sky), alpha);
```
In `original-prototype.html`, the switch is **instant** (no lerp).

---

## 7. How Content Configuration Is Loaded

```javascript
const content = await fetch('./content.json').then(r => { ... return r.json(); });
const stops = content.stops.filter(s => s.enabled).sort((a, b) => a.roadPosition - b.roadPosition);
```

Stops are filtered (`enabled: true`), sorted by `roadPosition`, then flattened into a `chapters` array that maps to Three.js scene state. The final stop appends `content.site.finalLetter` and `content.site.finalTelugu` to its modal body at runtime. Future scene illustrations are looked up by `futureScene` ID against `content.site.futureScenes`.

The fetch requires an HTTP server — the `file://` protocol will fail due to CORS.

---

## 8. Existing Bugs and Incomplete Functionality

### 8.1 Critical Bugs

| # | Bug | Location | Impact |
|---|---|---|---|
| B1 | **Tree z→t mapping is wrong.** `Math.abs(z)/pathLength` assumes all path length is in Z, ignoring the sinusoidal X wind. Trees near path centre use the wrong lateral offset. | `index.html` line ~44 | Trees clip into road on bends |
| B2 | **Season blend is framerate-dependent and stalls.** `blendSeason(dt)` is called with a per-frame delta (~0.016 s). At 60 fps, `alpha ≈ 0.019` per frame — colours never fully reach the target, they asymptotically approach it over many seconds. The lerp never "completes". | `index.html` line ~51 | Season never fully transitions |
| B3 | **`targetT` variable is declared but never used.** Dead code left from refactoring. | `index.html` line ~54 | Minor — dead code only |
| B4 | **`rockMat` is defined but no rocks exist.** Material is created, geometry never instantiated. | Both files | Minor waste |
| B5 | **Camera micro-sway multiplier is effectively zero** (`0.000001`). The idle camera animation is invisible. | `index.html` line ~57 | No idle motion |
| B6 | **`errorMessage` div exists but is never written to.** Error handling is present for WebGL check but no runtime errors surface to the user. | `index.html` | Silent failures |
| B7 | **`goTo()` doesn't guard against navigating while modal is open** from a dot-click. Clicking a progress dot while the modal is showing will start travel without closing the modal first (original prototype only — `index.html` does close it). | `original-prototype.html` | UX regression |
| B8 | **`dashGroup.children.forEach(d => d.visible = true)` runs every frame** doing nothing useful — all dashes are always visible. This is a no-op leftover. | Both files | Minor perf waste |

### 8.2 Incomplete Features

| # | Feature | Status |
|---|---|---|
| F1 | **Audio** — `audioSrc` in `content.json` is empty; audio is intentionally disabled per README | Skeleton only |
| F2 | **Future illustrations** — all `assets/future/*.jpg` paths are missing; gallery shows broken images | Placeholder paths only |
| F3 | **Family photos** — all `assets/photos/**` are empty; photo gallery shows alt-text fallback | No content yet |
| F4 | **Keyboard navigation** — no left/right arrow key handling for moving between stops | Not implemented |
| F5 | **Touch swipe navigation** — no horizontal swipe support on mobile to advance stops | Not implemented |
| F6 | **Loading progress** — loading screen shows static text, no indication of actual load state | Static only |
| F7 | **`prevBtn` missing in prototype** — `index.html` has it, `original-prototype.html` does not | Diverged |
| F8 | **Marker label tooltips** — `markerLabel` from `content.json` is available but only used in `dot.title` (hover tooltip), not rendered in 3D space | Partial |
| F9 | **`chapterCount` init text** — set to `"READY TO BEGIN · 1 OF N"` but then immediately overwritten by `updateCopy()` in the same init block | Race condition in text |

---

## 9. Mobile Responsiveness and Performance Concerns

### 9.1 Draw Call Budget

| Object Type | Count | Geometry Instances |
|---|---|---|
| Pine trees | 320 groups | 1,280 cone/cylinder meshes |
| Mountain cones | ~360 cones | 360 meshes |
| Road dashes | 260 boxes | 260 meshes |
| Milestone rings + orbs | 10 | 10 meshes |
| Ground, road, sun disc | 3 | 3 meshes |
| **Total** | **~1,913 draw calls** | No instancing |

On mid-range mobile (iPhone SE, Android mid-tier), sustained 60 fps is unlikely above ~400 draw calls without instancing. **`InstancedMesh` should replace trees and mountains.**

### 9.2 Mobile-Specific Issues

- `backdrop-filter: blur()` on modal is GPU-expensive on older Android WebViews — causes jank during scroll
- `pointer-events` on the HUD `hero` section has `pointer-events:none` (correct) but this means the hint text can't be selected
- `max-scale=1, user-scalable=no` in viewport meta prevents accessibility zoom
- Progress dots at `bottom: 95px` can overlap with iOS home indicator bar (safe area not respected)
- No `touch-action: none` on the canvas — browser may intercept swipes as scroll gestures, causing visual judder

### 9.3 Memory / GC Concerns

- 320 `tree()` calls each create 4 `new THREE.ConeGeometry` and 1 `new THREE.CylinderGeometry` — 1,600 unique geometry objects on the heap
- `makeMountains()` uses `Math.random()` without a seed — terrain layout changes on every reload (inconsistent experience)
- No geometry disposal on unmount / visibility changes

---

## 10. Opportunities to Improve Cinematic Visual Quality

| Priority | Opportunity | Effort | Impact |
|---|---|---|---|
| HIGH | **Road elevation** — add `y` variation to spline control points (gentle hills, a crest, descent) | Low | Huge — transforms flat into cinematic mountain road |
| HIGH | **InstancedMesh for trees and mountains** | Medium | Major perf gain; enables far more density |
| HIGH | **Bloom post-processing on sun disc and ring markers** — use `THREE.UnrealBloomPass` | Medium | Sun feels real; markers glow beautifully |
| HIGH | **Fix camera idle sway** — increase multiplier from `0.000001` to `0.00008`; add gentle yaw component | Trivial | Immediate sense of life while parked |
| MEDIUM | **Stars in night/dusk sky** — points geometry for seasonal 4/5 (renewal, future) | Low | Adds romance to late scenes |
| MEDIUM | **Sun glow / lens flare** — additive `Sprite` with radial gradient texture behind sun disc | Low | Cinematic atmosphere |
| MEDIUM | **Depth of field effect** — `BokehPass` or manual near-field blur for foreground objects | High | Strong cinematic quality, but expensive |
| MEDIUM | **Terrain heightmap** — replace flat `PlaneGeometry` with subdivided mesh displaced by noise | Medium | More believable ground |
| MEDIUM | **Road surface material** — add normal map or bump detail to asphalt to catch light | Low | Subtle realism |
| MEDIUM | **Camera look-ahead varies with speed** — wider during fast travel, tighter when stopped | Low | Motion feels more purposeful |
| LOW | **Wind animation on trees** — apply `ShaderMaterial` vertex displacement with time uniform | High | Beautiful but expensive |
| LOW | **Roadside props** — stone km markers, low guard rails using instanced geometry | Medium | Environmental storytelling |
| LOW | **Transition wipe effect** — brief film-burn or fade-to-black when season flips sharply | Low | Cinematic polish |

---

## 11. Recommended Implementation Sequence

### Phase 1 — Bug Fixes and Stability ✅ COMPLETE

> Goal: everything that exists works correctly. No new features.

1. ✅ **Fix tree z→t mapping** (B1) — trees now use uniform spline t-sampling + tangent perpendicular for correct lateral placement.
2. ✅ **Fix season blend** (B2) — verified frame-rate independent; per-frame Color allocation eliminated.
3. ✅ **Fix camera idle sway** (B5) — transient `displayT` offset prevents drift; `reducedMotion` respected.
4. ✅ **Remove dead code** (B3, B4, B8) — `targetT`, `rockMat`, redundant visibility loop all removed.
5. ✅ **Add keyboard navigation** (F4) — `ArrowRight`/`ArrowLeft`/`Enter`/`Space`/`Escape` implemented.
6. ✅ **iOS safe-area** — `viewport-fit=cover` + `env(safe-area-inset-bottom, 0px)` on `.progress` and `.hud`.
7. ✅ **Guard `goTo()` while modal open** — `closeModal()` called before travel in all paths.
8. ✅ **Meaningful error reporting** (B6) — `showError()` surfaces WebGL, fetch, config, and validation errors.
9. ✅ **Tab visibility dt spike** — `visibilitychange` resets timer; dt clamped to 100 ms.
10. ✅ **Deterministic scenery** — `mulberry32` seeded PRNG replaces `Math.random()` for all procedural objects.

### Phase 2 — Performance (Mobile First) ✅ COMPLETE

> Goal: smooth 60 fps on a mid-range phone.

1. ✅ **`InstancedMesh` for pine trees** — 1,280 individual meshes replaced with 5 InstancedMesh (1 trunk + 4 foliage-colour batches). Single shared `CylinderGeometry` for all trunks; single shared unit `ConeGeometry` for all foliage tiers. Non-uniform per-instance matrix scaling reproduces all size variation.
2. ✅ **`InstancedMesh` for mountains** — 360 individual cone meshes replaced with 6 InstancedMesh (one per colour range). Single shared unit `ConeGeometry(1,1,6)`. Non-uniform per-instance matrix scale reproduces original height × radius variation.
3. ✅ **Seeded PRNG** — `mulberry32(0xBEEFCAFE)` replaces `Math.random()` for all scenery. Identical RNG call sequence preserved so all world positions are unchanged.
4. ✅ **Shared matrix helpers** — `_m4`, `_qY`, `_axY`, `_ipos`, `_iscl` reused across all ~1,640 instance compositions to avoid per-iteration allocation.
5. **Reduce dash count** — deferred; 260 dashes is moderate cost compared to instanced scenery.
6. **`backdrop-filter` on low-end** — deferred to post-Phase 3.

**Phase 2 results:**

| Metric | Before | After | Notes |
|---|---|---|---|
| Scenery draw calls (estimated) | ~1,640 | ~11 | 6 mountain + 5 tree InstancedMesh |
| Total draw calls (estimated) | ~1,650 | ~21 | Including ground, road, dashes, sun, rings |
| Unique geometry objects | ~1,650 | ~12 | 3 shared scenery + 9 other |
| Unique material objects | ~390 | ~11 | 6 mountain + 1 trunk + 4 foliage |
| Instancing batches | 0 | 11 | All scenery instanced |
| Browser measurement | ⚠️ Not performed | — | Check DevTools console for `renderer.info` log |

*Estimated draw-call reduction: ~98.7% for scenery geometry. Actual values logged to browser console after first frame.*

### Phase 3 — Road and Camera Quality ⏪ REVERTED (intentional)

> **Status: Implemented in commit `02ac0bc`, then intentionally reverted in commit `f26f494`.**
>
> **Reason for revert:** After reviewing the elevated mountain road and terrain changes live, the original flat winding road, scenery, and lighting were preferred. The visual experience was better with the flat road. The revert was performed via `git revert 02ac0bc` (no history was rewritten; both commits are preserved in the log on `feature/stability-navigation`).

#### What Phase 3 had implemented (now removed)

- Piecewise-smoothstep road elevation profile (`ELEV_PTS`, `roadElevation()`)  
- Updated spline Y to climb from y=0 → peak y=21 at t=0.88 → y=18 descent  
- `terrainYAt(wx, wz)` terrain height function (sigmoid lateral drop-off + undulation)  
- Road-following terrain strip mesh (6,231 vertices / 12,000 triangles, `groundMat`)  
- Ground plane lowered to y=−25 (valley floor)  
- Road dash Y fix: `a.y + 0.055`  
- Mountain bases anchored via `terrainYAt()`  
- Tree grounding via per-tree sigmoid base Y  
- Milestone markers: `pos.y = p.y + 1.7`

#### Current state after revert

All pre-Phase-3 values are restored exactly:
- Spline Y = 0 for all control points (flat winding road)
- Ground plane at y = −0.08 with `receiveShadow`
- Road dashes at fixed y = 0.055
- Mountain centres at `h/2 − 1` (original positioning)
- Tree trunks centred at `0.7 × scale`, foliage at `(1.2 + j × 0.62) × scale`
- Milestone markers at `pos.y = 1.7`

**Verification:** 30/30 structural assertions passed; `node --check` syntax clean.

#### If Phase 3 should be revisited

Re-implement from the Phase 3 design notes above, or cherry-pick `02ac0bc` and refine the elevation profile and terrain blending to better suit the artistic vision before re-applying.

**Original planned items (unchanged):**

1. **Add elevation to spline** — modify Y of control points to create 2–3 gentle hills and a final crest before the golden horizon stop
2. **Increase look-ahead during travel** — interpolate `lookAhead` from `0.035` (stopped) to `0.065` (full speed)
3. **Dynamic FOV** — narrow FOV slightly when stopped (`62°`), widen during travel (`70°`) for speed sense
4. **Camera roll on bends** — apply gentle bank angle derived from tangent X change

### Milestone Storytelling Panel ✅ COMPLETE

> **Committed:** `7c8d04a` on `feature/stability-navigation`  
> **Changed:** `index.html` only (+~170 / −~35 lines)  
> **Goal:** Transform memory panels into cinematic, readable storytelling experiences without touching the 3D environment.

#### What was implemented

| Area | Detail |
|------|--------|
| **Layout** | Centered modal → cinematic side panel (460 px, slides from right on desktop; bottom sheet on mobile ≤899 px) |
| **Animation** | `translateX(102%)` → `translateX(0)` on open; reverse on close via `visibility:hidden` + 0.5 s transition-delay (avoids `display:none` animation trap) |
| **Typography** | Georgia/Times serif headings; body 16 px / line-height 1.85; subtitle (was absent) added from `c.subtitle` |
| **Story paragraphs** | `rawStory.split(/\n{2,}/)` → individual `<p>` per paragraph; no more unbroken text wall |
| **Telugu** | `Noto Sans Telugu` loaded via Google Fonts; section hidden (`hidden` attr) when `teluguMessage` is empty |
| **Accessibility** | `role=dialog`, `aria-modal`, `aria-labelledby`; `aria-hidden` toggled on open/close; focus → ✕ on open, → `memoryBtn` on close; `panel.scrollTop = 0` reset |
| **Reduced motion** | `prefers-reduced-motion: reduce` disables all CSS transitions |
| **Backdrop** | Desktop: transparent (road visible); mobile: `rgba(7,16,27,.62)` dim; click closes panel |
| **Buttons** | ✕ close (`id=closeBtn`), `Continue the journey →` (`id=continueBtn`), `Return to Road` (`id=panelReturnBtn`) |
| **Photos** | Replaced by the gallery slideshow (see below) |
| **3D environment** | Zero changes — road, camera, scenery, lighting all untouched |

#### Assertions: 51 / 51 passed ✓ (1 false-fail was a PowerShell `-and` precedence bug in the test script — verified correct by direct `Contains` check)

---

### Photograph Gallery ✅ COMPLETE

> **Committed:** `8342af4` on `feature/stability-navigation`  
> **Changed:** `index.html` only (+309 / −27 lines)  
> **Goal:** Replace the horizontal scroll strip with an elegant, accessible, single-image slideshow with a full-screen lightbox viewer.

#### What was implemented

| Area | Detail |
|------|--------|
| **Slideshow** | One photo at a time in a fixed `aspect-ratio:4/3` frame — no layout shift while loading |
| **object-fit:contain** | Portraits, landscapes, and squares all fit without cropping |
| **Prev / Next** | Arrow buttons rendered only when > 1 photo; wrap-around navigation |
| **Photo counter** | `1 / 3` format with `aria-live="polite"` (hidden when ≤ 1 photo) |
| **Caption** | Below frame; `display:none` when empty |
| **Expand button** | SVG icon (bottom-right of stage); opens full-screen lightbox |
| **Broken images** | `onerror → gallery-placeholder` (camera SVG + text); no browser broken-icon; no layout shift |
| **Empty gallery** | `host.setAttribute('hidden')` — no blank area left behind |
| **Lightbox** | Full-screen `z-index:50` overlay; `object-fit:contain`; close ✕, Prev/Next, backdrop click |
| **Lightbox Escape** | Intercepted before panel Escape — Escape closes lightbox, not the panel |
| **Keyboard** | `ArrowLeft/Right` in gallery stage; `Enter/Space` on stage opens lightbox; `stopPropagation` prevents HUD milestone nav |
| **Touch swipe** | `touchstart/move/end` passive; horizontal delta > 40 px triggers prev/next |
| **Reduced motion** | Frame crossfade and lightbox opacity transitions disabled |
| **Focus management** | Lightbox opens → focus to ✕; closes → focus returned to opener element |
| **Accessibility** | `role=region` on stage; `aria-label` per photo; `aria-label`, `aria-hidden` on lightbox dialog |
| **`mkEl()` helper** | Lightweight `document.createElement` + className setter; distinct from `$()` (getElementById) |

#### Assertions: 54 / 54 real checks passed ✓  
(4 apparent failures were PowerShell test-script bugs — `-and` operator precedence and `$()` string interpolation — all verified correct via direct `Contains` checks)

#### Known limitations (browser testing not available in this environment)
- Lightbox, gallery animations, and swipe must be verified on physical devices.
- All photo paths (`assets/photos/`, `assets/future/`) are placeholder — images will show the broken-photo placeholder until real assets are added in Phase 5.
- `loading="lazy"` for non-first photos requires HTTP server (not `file://` protocol).

---

### Phase 4 — Visual Enhancement

> Goal: post-processing and atmospheric polish.

1. **Three.js `EffectComposer`** — add `RenderPass`, then:
   - `UnrealBloomPass` (threshold 0.85, strength 0.4, radius 0.6) — affects sun disc and rings
   - Optional: `FilmPass` for grain texture
2. **Sun glow sprite** — additive `Sprite` at sun position with a radial gradient canvas texture
3. **Stars geometry** — `BufferGeometry` of 2,000 random points, visible only in `renewal` and `future` season palettes
4. **Enhanced seasonal palettes** — add `ambient` intensity, `fogDensity`, and `sunY` position per season so the sun actually moves in the sky

### Phase 5 — Content and Story Polish

> Goal: the experience is complete and ready to share.

1. **Add generated future illustrations** to `assets/future/` using prompts in `content.json`
2. **Add family photos** (web-optimised, 1600 px JPEG) to `assets/photos/` subdirectories
3. **Populate stop stories** in `content.json` with real (or final placeholder) content
4. **Telugu font** — ✅ Done (Noto Sans Telugu loaded via Google Fonts in milestone panel)
5. **Audio** — if ambient track is sourced, wire `audioSrc` to a muted-by-default `<audio>` element with a toggle button
6. **Final end-to-end test** on physical iOS + Android devices before sharing

---

## 12. File Modification Map

> When implementing, touch files in this order to minimise risk of regression.

```
Phase 1–3:  index.html only
Phase 4:    index.html (add EffectComposer import + passes)
Phase 5:    content.json + assets/ (content drops only)
            Never touch original-prototype.html
```

---

## 13. Dependency Audit

| Dependency | Source | Version | Status |
|---|---|---|---|
| `three` | jsDelivr CDN | `0.161.0` | Current-ish — Three.js 0.170+ available; upgrade optional |
| `three/examples/jsm/postprocessing/*` | Same CDN | Must match core version | Not yet imported — needed for Phase 4 |

No build tool, bundler, or package manager is used. All imports are ES module CDN URLs — the approach can be maintained as-is.

---

## 14. Summary of Key Findings

**What is working well:**
- Core Three.js renderer is configured correctly (ACES, SRGB, PCF shadows)
- CatmullRom spline + custom ribbon road geometry is solid architecture
- Content-driven configuration via `content.json` is clean and editable
- Seasonal palette system is well-structured and easy to extend
- Modal milestone system (open/close/continue) is fully functional
- Raycasting on 3D rings for click/tap navigation works correctly
- Responsive CSS handles mobile viewports reasonably
- Reduced-motion preference is respected

**What needs immediate attention:**
- Tree placement math is geometrically incorrect (B1)
- Season blending never reaches its target colour (B2)
- Camera idle sway is invisible — multiplier is 4 orders of magnitude too small (B5)
- ~1,900 draw calls will cause frame drops on mobile phones (no instancing)
- iOS safe area not respected for progress dots

**Biggest single visual improvement available:**
Adding Y elevation to the spline control points (Phase 3, step 1) — this costs ~10 lines of code and transforms the experience from a flat drive into a mountain road with genuine drama.

---

*Phase 1 implemented 26 October 2026. `index.html` updated on branch `feature/stability-navigation`. `original-prototype.html` unchanged. Phase 2 (InstancedMesh performance) not yet started.*
