# AFTERBURN

A browser jet racer built on a custom six-degree-of-freedom aerodynamic solver.
One HTML file, no build step, no external assets.

## Play it

**<https://whitdeckard09.github.io/afterburn/>**

Nothing to install. Desktop browser, keyboard or gamepad.

## Run it locally

```bash
python3 -m http.server 8123
```

Then open <http://localhost:8123/>.

A server is required even locally. The page pulls three.js as an ES module from
a pinned CDN, and module imports fail under `file://` (opaque origin). Any
static server works — which is also why it drops straight onto GitHub Pages.

## Controls

| | Keyboard | Gamepad |
|---|---|---|
| Pitch / roll | `W` `A` `S` `D` or arrows | Left stick |
| Yaw | `Q` `E` | Bumpers |
| Throttle | `Shift` / `Ctrl` | Right stick / triggers |
| Afterburner | `Space` | A |
| Airbrake | `X` | B |
| Camera | `C` | Y |
| Look back | `V` | — |
| HUD on/off | `H` | — |
| Pause | `Esc` | — |

Keyboard, gamepad and HOTAS are live simultaneously — whichever moved last owns
the axis. Settings → Controls has per-axis sensitivity, deadzone, expo, invert,
key rebinding and a live input visualiser that shows raw device axes, which is
the quickest way to confirm a stick is mapped correctly.

## What is actually simulated

Nothing here is a handling curve. The numbers in the `JETS` table feed the
solver directly, so changing `S` (wing area) genuinely changes the stall speed.

- **Lift** blends an attached-flow coefficient (`CL = CL0 + CLα·α`) into a
  separated flat-plate model through a sigmoid centred on the critical angle.
  Measured: CL peaks at **1.37 at 18° AoA**, then breaks — 1.19 by 22°, 0.98 by
  24°. The blend is what makes the stall recoverable instead of a cliff.
- **Drag** is parasitic + induced (`k = 1/πeAR`) + transonic wave drag. Thrust
  required at 6 km rises from **21 kN at Mach 0.7 to 88 kN at Mach 1.0** — you
  can feel the wall without looking at the Mach readout.
- **Moments** come from dynamic pressure and a full coefficient set, so control
  authority falling off at low speed is a consequence of the model, not a tuned
  curve. Rotation uses Euler's equations with a real inertia tensor.
- **G** is measured along the seat axis. The fly-by-wire limiter anticipates
  pitch rate, so full back stick settles on the airframe limit instead of
  sailing past it: **9.19 G against 9.0** on the Vesper, **9.50 against 9.50**
  on the Kestrel, **7.75 against 8.0** on the Titan — none of them take over-G
  damage, while a gentle 3 G pull still tracks to 2.93 G. The turn bleeds
  **31 knots in 8 seconds** and greys the pilot out after about six. Control
  authority degrades before vision does.
- **Ground effect** follows the Wieselsberger relation. Measured **20.6 % less
  drag at 170 knots** inside one wingspan of the surface. Low is fast.
- **Atmosphere** is ISA — density, temperature and speed of sound all vary with
  altitude, so Mach, thrust and lift fall off together. Top speed is Mach 0.96
  at sea level and Mach 1.32 at 9 km, and full afterburner empties the tanks in
  about six minutes.
- **Fly-by-wire** commands normal acceleration and roll rate with a hard
  structural limiter, as a modern fighter does. Settings → Flight switches to
  Direct law, which puts the stick straight on the control surfaces.

Physics runs at a fixed 120 Hz with interpolated rendering, and is deterministic:
the same inputs from the same state produce bit-identical results, and the
accumulator emits the same number of steps at 30, 60 and 144 fps.

## Everything is procedural

There are no asset files. Airframes are lofted at load time from superelliptical
fuselage cross-sections and NACA aerofoil profiles, so the wings have real
thickness, taper and washout. Paint, panel lines, rivets and normal maps are
drawn to canvas and differentiated with a Sobel pass. Terrain is fractal noise
baked to a heightfield that the GPU displaces and the CPU collides against — the
same array, the same bilinear filter, so what you see is exactly what you hit.
All audio is synthesized in Web Audio.

## Lighting

Each course carries its own `light:{}` block — exposure, ambient and direct
intensity, and optional sun/ambient colour overrides — layered over the shared
time-of-day palette. That is necessary because a desert at midday, a sunrise
over reflective water and an arctic dusk need very different amounts of light
to read at the same perceived brightness; a single palette had Cerulean blown
out and Boreal nearly black. Tuned values:

| Course | Exposure | Sun elevation | Note |
|---|---|---|---|
| Vermilion Canyon | 0.33 | 38° | Midday desert, low Rayleigh so the sky stays deep |
| Cerulean Coast | 0.40 | 31° | Sun lifted off the horizon; sky brightness cut at source rather than by exposure |
| Boreal Pass | 0.48 | 16° | Pale gold key with a cool blue ambient, so snow reads as snow rather than desert |

## Performance

Rendering is fill-rate bound, so device pixel ratio is the dominant lever and is
capped per quality preset (Low 1.0 → Ultra 2.0). Physics for five aircraft costs
about **0.02 ms per 120 Hz tick** — roughly 0.25 % of one core — so the frame
budget is essentially all rendering. Quality presets are in Settings → Graphics.

## Known limitations

- **AI gate accuracy.** The AI opponents fly the same airframe through the same
  control inputs a human uses — pursuit guidance, bank-to-turn, corner-speed
  braking, terrain avoidance. There is no rubber-banding and no path scripting.
  They complete every lap, crash rarely (about 7 incidents across four aircraft
  over a nine-minute race), and post consistent, correctly-ordered lap times
  (the quickest is repeatable to a tenth of a second). But they still fly wide
  of roughly a third to two thirds of the gates and take the time penalty. Their
  tracking error is ~150 m laterally against a 200 m gate radius, so they are
  marginal rather than reliable. This is the weakest system in the build.
- **Cut from the original brief:** multiplayer (needs a server, which
  contradicts the single-file constraint), career progression and unlockables,
  replay cinematics. See `SPEC.md` for the reasoning.
- Altitude caps are opt-in (the HARDCORE toggle on the course screen) rather
  than on by default — on a canyon course the aircraft legitimately has to climb
  to clear a wall, and penalising that made the modifier punishing rather than
  hard.

## Development hooks

`window.AFTERBURN` exposes the simulation internals so the flight model can be
validated from the console without driving the UI:

```js
AFTERBURN.launch('race', 0)   // load a course directly
AFTERBURN.fastForward(300)    // advance 300 s at the fixed timestep
AFTERBURN.JETS, .TRACKS, .Aircraft, .Track, .ATM
```

Runtime errors are surfaced on screen rather than failing silently.
