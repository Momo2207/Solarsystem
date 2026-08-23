# Finer-Detail Rendering — Implementation Plan

Goal: render **finer detail within the galaxy** (dust filigree, spiral feathering/spurs,
gas knots, HII complexes, sharper point sources) in `index.html`.

Scope: 8 improvements. **Barnes–Hut / tree gravity is explicitly OUT of scope and will
never be implemented.** No other structural rewrite of the physics N-body model.

Guiding constraints (this is a single 6,767-line file, no browser-based visual QA here):
- Keep every change localised and reversible.
- Preserve mean brightness/extinction (detail = higher spatial frequency, not a
  global darken/brighten). Noise modulations are **mean-preserving** (factor centred on 1.0).
- Anything with non-trivial risk is gated behind a UI control whose **default reproduces
  current behaviour**, so the baseline experience can't regress and the user can A/B it.
- Never move or alter the `//<<PHYSICS>>` / `//<<END_PHYSICS>>` sentinels — the physics
  Web Worker is extracted from them at runtime.

Validation after each step: extract the `#mainScript` body and `node --check` it (syntax),
then a final headless Chromium smoke load (console-error + init check).

---

## 1. Higher-resolution dust field + mean-preserving sub-grid detail
Dust lanes are the most detail-defining feature; the field is only 96².
- `DUST_FIELD_SIZE` 96 → **192** (line ~1673). `dustTauField/dustDepthField/dustTempField`,
  `dustTexData` (2265) and the `dustTex` allocation (2284) all size off this constant — no
  other alloc change. `uDustN` uniform already fed `DUST_FIELD_SIZE` (5628).
- `buildProjectedDustField` (1718): raise the per-cloud splat radius cap `mapR` 4 → **8** so
  physical lane coverage is preserved at 2× resolution (otherwise splats shrink and gap).
- GPU star shader `vsStarSrc` (2047): declare `uniform sampler2D uNoiseTex;` (already bound to
  unit 1 at 5627 but currently unused) and modulate the decoded `tauF` by a **world-locked**
  mean-preserving factor `≈0.72 + 0.56*fBm(aStarPos.xy)` (two octaves). World-locked so lane
  texture doesn't swim under camera motion.
- CPU `sampleProjectedDustField` (1763): rely on the resolution bump only (screen-space noise
  would swim; this path only feeds gas particles + diffuse maps, secondary).
- Cost: 3×192² Float32 (~440 KB), splat ~3×, 192² texel repack/frame. Fine.
- Risk: low–med. Falls back cleanly (dust already gated by realism/dust toggles).

## 2. Higher-resolution filament noise texture
Finest octave currently equals the lattice (`f=32=P`) → nothing finer to sample.
- In the `noiseTex` IIFE (2320): `N` 128 → **256**, `P` 32 → **128**, octaves 4 → **6**
  (`f = 4<<o` → 4,8,16,32,64,128; invariant `max f ≤ P` preserved). Gas + dust sprites gain
  ~2× finer wisps. One-time gen, one 256² texture. Risk: none.

## 3. Full gas coverage + fine sub-sprite splatting
Gas carries the visible fine structure but is CPU-strided at 30k while the cloud cap is 40k.
- `RENDER_GAS_BUDGET` 30000 → **60000** (stride 1 for all clouds; bounded by #4 frustum cull).
- New checkbox `tglFineGas` ("Fine Gas Structure", Astrophysical Detail group), global
  `fineGasDetail` (default **on**). When on, the gas loop (5530) emits one extra small
  noise-seeded sub-sprite per sufficiently large cloud → filamentary knots.
- Risk: low. Toggle off = pre-existing two-sprite look.

## 4. Frustum-aware LOD with temporal-feedback decimation ("detail where you look")
Striding is global, so zooming in only magnifies a sparse sample.
- Precompute `cos/sin(rotX,rotY)` once per frame; inline a cheap projection cull.
- Gas loop + CPU-star loop (5530 / 5547): skip off-screen particles **before** the expensive
  `getGasRenderParams`/`getStarRenderParams`; decimate the visible set to budget using the
  previous frame's visible count (`lastVisibleGasCount`/`lastVisibleStarCount`) — self-tuning,
  stable. Net effect: when zoomed in, visible count drops → stride drops → full on-screen detail.
- GPU-star visible-mode path (draws all stars) is untouched. Benefits gas everywhere and the
  CPU-star path used by IR/UV/X-ray/radio/telescope and the GPU-off fallback.
- Risk: med (must replicate projection exactly). Guarded by matching the existing
  `rawX*scale*highResMult + translateX*highResMult + centerX` formula.

## 5. HII-region substructure
Currently plain pulsing radial gradients (`drawHIIRegions`, 4750).
- Add a brighter ionisation **rim shell** (extra gradient stop) + 2–3 embedded **knot**
  sub-gradients placed by a stable per-region hash (`r.id`/position), alpha ∝ intensity.
  Keep the per-mode recolour and dust attenuation. Overlay-only 2D — zero physics cost.
- Risk: low (isolated).

## 6 + 7. Unified "Render Quality" internal supersampling + still-frame auto-boost
Delivered together as one render-scale subsystem (both are internal-resolution SSAA).
- New global `renderScale`; `<select id="renderQuality">` (Core Physics group): `auto`, `1`,
  `1.5`, `2`. Model: **`highResMult` = active render scale during live view** (reuses the
  proven screenshot supersampling path — it already scales projection, sizes and overlays).
- `onResize` (2495): backing = clientSize × activeScale; `highResMult` = activeScale;
  `pcPerPixel` from **clientWidth** (display px) so the HUD scale stays correct.
- Fix mouse→backing mapping: multiply CSS mouse coords by activeScale at the `mousemove`
  (6279) and click (6308) handlers; change the hover gate `highResMult === 1` (5712) to
  `=== activeScale`. Panning stays 1:1 (translate is multiplied by `highResMult` already).
- `auto`: base 1×; when **paused + mouse outside canvas + camera idle** (~45 frames), boost to
  2× (a crisp static frame = the #7 outcome); revert instantly on any interaction/unpause/step.
  Gating on mouse-out means picking never happens at boosted scale.
- Screenshot path: ensure it restores to `activeScale` (not hard-coded 1) after capture.
- Default `auto` ⇒ identical to today during motion/interaction; only idle stills sharpen.
- Risk: highest of the set, but blast radius contained (non-1× only on a static, non-interactive
  frame). Manual 1.5/2 available for deliberate high-detail viewing.

## 8. Finer self-gravity grid (NOT Barnes–Hut)
The 96² grid + 3×3 smooth sets the smallest self-organised structure (why arms lack feathering).
- `LG_N` 96 → **128** (line 1125, inside a `//<<PHYSICS>>` region → runs in the worker).
  Smaller cells ⇒ finer spurs/knots; `LG_CAP=0.35` still bounds the force so stability holds.
  Keep the single binomial smoothing pass and `LG_STRENGTH`. Grid 128²=16,384 cells (~1.8×).
- Risk: med (live dynamics). Mitigated by the existing force cap; value chosen moderate to
  limit shot noise at low particle counts.

---

## Execution order
2 → 1 → 5 → 3 → 4 → 8 → 6/7 (cheap/isolated first; riskiest render-scale last), `node --check`
after each, headless smoke test at the end, then commit + push to
`claude/solarsystem-repo-analysis-n58f8n`.
