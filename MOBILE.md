# Mobile

Before this pass the simulator had no mobile story at all: **no media queries and no touch
handlers**. On a phone the 320px sidebar left a strip of galaxy beside it, and the canvas was
inert — pan, zoom and orbit were bound to mouse drag and wheel, none of which a finger emits.

## Touch

The canvas now follows map-app conventions, because that is what a thumb already expects:

| gesture | action |
|---|---|
| one finger drag | pan |
| two finger pinch | zoom about the midpoint |
| two finger drag | orbit (what right-drag does with a mouse) |
| tap | select a star |

`touch-action: none` on the canvas is what stops the browser scrolling the page instead of
handing the gesture to the simulation.

Pinch and orbit are read from the *same* two-finger contact and applied together: the change
in finger separation drives zoom, the movement of their midpoint drives orbit. They do not
need separating, because a pinch that also slides is a legitimate request for both.

A tap is a contact that ends within 500 ms having moved under 12 px. Its pick radius is
**2.4x** the mouse one — a fingertip is ~8 mm across and cannot be aimed like a cursor.
Releasing one finger of a pinch does not become a tap.

Selection logic is shared with the mouse path (`selectAtPoint`) rather than duplicated, so
the two cannot drift apart.

## Layout

Below 900px (or any coarse pointer under 1100px):

- The sidebar becomes an **off-canvas drawer** over a full-bleed canvas, with a scrim.
  It is promoted to the compositor with `will-change: transform` — the render loop owns the
  main thread on a phone, and without it the drawer visibly steps instead of sliding.
- Because the drawer covers the bar that opened it, it carries **its own close button**;
  otherwise the only way out is a ~55px strip of scrim.
- A **thumb bar** pins the four things people reach for constantly — Controls, Pause,
  Restart, Data — to the bottom, clear of the safe-area inset. Its buttons mirror the real
  controls rather than reimplementing them, so the two cannot disagree.
- **Telemetry starts hidden.** At phone width the HUD covers roughly a third of the screen;
  the Data button brings it back.
- Touch targets go to 44px minimum, and every input is 16px — below that iOS zooms the whole
  page when a control takes focus.
- `100dvh` rather than `100vh`, so the bar is not stranded behind a browser's URL chrome.
- Floating panels become full-width sheets above the bar; the snapshot rail is hidden.

## Performance

A phone GPU is perhaps a twentieth of a desktop one, and every default in this file was
chosen for a desktop. Rather than guess per device, one conservative profile is applied
whenever the touch layout is active — and applied *before* the first `initSimulation`, so a
phone never builds the 40k-star default and then throws it away.

| setting | desktop | mobile | why |
|---|---|---|---|
| Star count | 40,000 | 15,000 | dominates both physics and draw cost |
| Raymarch quality | 44 | 20 | the single most expensive knob in the renderer |
| Dust detail | 8,000 | 4,000 | sprite count |
| HDR pipeline | on | off | extra full-screen passes |
| GPU particle layer | on | off | a second particle system |
| Background universe | on | off | a whole extra canvas |
| Render quality | auto | 1x | never supersample; also disables the idle 2x boost |

Everything stays user-adjustable afterwards — the profile sets the controls, it does not lock
them. A phone that can afford more can simply be turned up.

Note the canvas already backed at CSS pixels rather than device pixels, so a 3x-DPR phone was
never rendering at 3x resolution. That was luck rather than design, but it is the right
behaviour and this pass preserves it.
