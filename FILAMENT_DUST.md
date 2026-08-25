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

## Where the large-scale layout comes from

The noise supplies *texture*, not *arms*. Two things place it:

- **Differential-rotation shear** (`uTwist`): the sample point is rotated about z by an angle
  growing with `log R`, which winds structures into trailing arcs with no seam at ±π. At the
  old default of 2.2 this dominated everything and the galaxy read as a marbled whirlpool;
  1.4 keeps the flow without taking over.
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
| Shear / Winding | `filTwist` | 0 = no winding, 6 = tight spiral |
| Volume Density | `volDensityMult` | overall opacity |
| Raymarch Quality | `volSteps` | step count; lower it on weak GPUs |

Turning **Filament Dust (3D)** off restores the sprite path unchanged.

## Not in scope

Barnes–Hut / tree gravity is explicitly out of scope for this project and is not used
anywhere in the physics.
