# Filament Dust Field — how the 3D dust structure is generated

The dust in this simulation used to be drawn as sprites: one soft round splat per gas
cloud. Sprites are radially symmetric, so no arrangement of them produces a *filament* —
at best you get a lumpy field, which is what "blobby" and "patchy" describe. Scattering
each cloud into many sub-motes made the lumps smaller but did not change their shape.

The replacement raymarches a procedural 3D field through the disc (`fsVolSrc`,
`drawVolumetric`). This note records why it is built the way it is, because most of the
design is reactions to specific failures.

## The line-of-sight problem (the one that mattered)

A face-on raymarch integrates ~40 samples of the field along z. If the field decorrelates
over the disc thickness, that integral is an *average of independent samples* and it
converges to a constant — the structure cancels itself out. The first version made the
noise **finer** vertically than in-plane (`filFlatten = 3.4`) on the theory that dust is a
thin layer, and it rendered as smooth brushed haze no matter how the other knobs moved.

The fix is the opposite: the field is **columnar**, coherent through the disc thickness
(`filFlatten = 0.20`, so vertical features are ~5x longer than in-plane ones). The line of
sight then crosses roughly one feature and the in-plane pattern survives intact. Edge-on
this is also right — a vertically uniform thin slab reads as a clean dark lane.

## Three decoupled scales

One noise scale cannot look like dust. A coarse field gives smooth swirls; a fine one gives
uniform brushing, like combed hair. So the field is a product:

| term | what it decides | frequency |
|---|---|---|
| **lane** | *where* dust is at all — sparse, high-contrast, most of the disc is empty | base |
| **thread** | texture within a lane, the visible filaments | `uFineScale` (6x) |
| **grain** | detail that appears as you zoom | pixel-locked (see below) |

Each is ridged noise — `1 - |2n-1|`, which turns smooth lumps into sharp creases; plain fBm
can only give round blobs — passed through `shape()`, a low cut plus a power, so the field
reads as distinct threads against clear gaps rather than continuous haze. A single domain
warp is computed once and shared by all three, which stretches and folds otherwise-round
features into elongated, branching ones.

The `lane` term returning zero short-circuits the other two, so most samples cost one fetch.

## Pixel-locked detail cascade

Noise at a fixed world frequency runs out of detail when you zoom in: the same features get
magnified into blobs. That is the "still too patchy" complaint, and no amount of retuning a
fixed-frequency field fixes it.

`uGrainScale` is recomputed each frame so the cascade's finest octave lands at about **7
screen pixels**, whatever the zoom. Zooming in resolves new threads instead of enlarging old
ones; zooming out retires them before they alias into sparkle. It self-disables when the
`thread` term is already that fine (so wide views pay nothing), is capped so extreme zoom
cannot run away, and the `thread` term drops an octave while it is active so the per-step
fetch count stays flat.

## Making it move (two rotating frames)

The first version of this field was **static**: it was sampled at fixed world coordinates, so
the galaxy turned underneath dust that never did, and gravity and stellar feedback changed
nothing. Advecting it is not a detail — a dust lane that does not move is not a dust lane.

The fix is to sample the noise in a *rotating* frame, but which frame is not obvious, because
lanes and clumps genuinely move differently:

- **Dust lanes are the compressed ridge of a density wave**, and a density wave rotates
  **rigidly** at the pattern speed. That is precisely why real spiral arms do not wind
  themselves shut over a few orbits. The `lane` term is therefore sampled in the pattern
  frame, using the simulation's own `patternAngle` — the same angle the arm modulation in the
  gas volume rides on, so the two stay locked together.
- **Individual dust clumps orbit at the local circular speed** and stream *through* the
  pattern. The `thread` and `grain` terms are sampled in the material frame, rotated by
  `-Ω(R)·t`.

Sampling everything in one frame gives you either frozen dust (what it did) or a field that
winds up into a tight spiral and never stops (what a single material frame would do). Two
frames cost one extra rotation per sample and reproduce both behaviours.

`Ω(R)` comes from the simulation's real rotation curve, but `getTheoreticalVelocities` is far
too heavy to call per fragment, so it is fitted on the CPU to
`v(R) = v_flat · R / sqrt(R² + Rc²)` — hence `Ω(R) = v_flat / sqrt(R² + Rc²)`, one `sqrt` in
the shader. `Rc` is found by a coarse scan and `v_flat` is the least-squares optimum for each
`Rc` (it is linear, so that half is exact). Measured fit error against the real curve is 2-6%
across the disc, rising to ~9% at the very outer edge. The fit is redone whenever the pattern
speed is, so it tracks mass, radius and preset changes.

## How gravity and stellar events reach the dust

The simulated gas volume is rebuilt every other frame from the actual particles, so it already
carries everything: arms sweeping past, clouds collapsing, supernovae and H II regions
clearing gas. What matters is *how* the dust reads it.

It modulates the **lane threshold**, not opacity. That distinction is the whole game: the gas
volume is a blurred mip, so scaling opacity by it stamps soft round blobs over the image —
which is exactly the artefact this renderer exists to avoid. Moving the threshold instead
means dense gas simply lets more filaments through, and every visible edge is still a filament
edge.

The threshold is centred on `uEnvPivot`, **measured from the volume itself** each frame (the
mean stored density over voxels that actually hold dust, smoothed) rather than hard-coded. It
has to be: the measured value is ~0.22, and the 0.75 originally guessed would have pushed the
threshold up by 0.4 and left the disc nearly dust-free. Measuring it also means the coupling
stays centred whatever the gas mass, galaxy size or dust bias.

## Where the large-scale layout comes from

The noise supplies *texture*, not *arms*. Two things place it:

- **Static winding** (`uTwist`): the sample point is rotated about z by an angle growing with
  `log R`, which wraps structures into trailing arcs with no seam at ±π. This sets how tightly
  wound the lanes look, independent of time — the time-dependent motion is the two frames
  above. At the old default of 2.2 this dominated everything and the galaxy read as a marbled
  whirlpool; 1.4 keeps the flow without taking over.
- **The simulated gas** (`uEnvMix`, raised to 0.55): the blurred gas density from the actual
  N-body run modulates the field, so lanes follow the arms the simulation produced rather
  than a pattern painted on top of it.

## Colour

Optical depth accumulates **per channel** (`uExtCol = (0.62, 0.82, 1.0)`, blue absorbed most)
and composites as pure multiplicative extinction, `blendFuncSeparate(ZERO, SRC_COLOR, ZERO,
ONE)`. An earlier version added single scattering; it washed the lanes out to grey, which is
why lit volumetric dust was rejected in favour of the warm brown look. Alpha is left
untouched so the WebGL canvas still composites over the background-universe canvas.

## Controls

| control | maps to | notes |
|---|---|---|
| Filament Fineness | `filFreq` -> `uFxy` | overall feature scale |
| Lane Coverage | `filCoverage` | inverted: 38% coverage is a cut of 0.62 |
| Thread Definition | `filSharp` | higher = thinner, harder threads |
| Shear / Winding | `filTwist` | static wrap; 0 = none, 6 = tight spiral |
| Volume Density | `volDensityMult` | overall opacity |
| Raymarch Quality | `volSteps` | step count; lower it on weak GPUs |

Turning **Filament Dust (3D)** off restores the sprite path unchanged.

## Not in scope

Barnes–Hut / tree gravity is explicitly out of scope for this project and is not used
anywhere in the physics.
