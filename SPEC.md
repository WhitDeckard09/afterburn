# AFTERBURN — Revised Build Spec

Revision of the original brief. Changes are justified inline under **WHY**.

## 0. Prime directive

A jet racer where the *flying* is the game. Every other system exists to make the
flight model legible. When a feature and the frame budget conflict, the feature loses.

Target: **stable 60 fps at 1080p on integrated graphics** (Iris Xe / M1), scaling up to Ultra.

---

## 1. Technical foundation (CHANGED)

| Original | Revised | Why |
|---|---|---|
| Cannon-es or Ammo.js | **Custom 6-DOF solver, no physics engine** | Rigid-body engines model contacts and constraints, not aerodynamics. Lift/drag/moments must be authored by hand regardless; adding an engine means fighting its integrator for control of the same body, plus ~600 KB and a broadphase we never use. Collision here is analytic: segment-vs-heightfield raycast and disc tests for gates. |
| "PBR metalness/roughness maps" | **Procedurally generated maps** (canvas → texture) | No build step means no asset pipeline. Panel lines, wear, and livery are painted to an OffscreenCanvas at load and used as albedo/roughness/normal. |
| "opening the HTML file" | **Single file, served over `python3 -m http.server`** | ES module imports from a CDN fail under `file://` in Chrome (opaque origin). One command, stated in the README. |
| Multiplayer | **Cut. Replaced with physics-honest AI.** | Netcode needs a server, which contradicts the single-file constraint. AI opponents fly *the same flight model* through an autopilot — no scripted paths, no rubber-banding. Skill difference is real. |
| Career + unlockables | **Cut. Championship kept.** | Progression metagame adds no flight quality. |
| 4 biomes, volumetric clouds, replay cinematics | **3 hand-tuned tracks, billboard clouds** | Depth over breadth. |

**Kept as specified:** fixed-timestep physics with render interpolation, object pooling,
WebGL2 with WebGL1 fallback, quality presets.

---

## 2. Flight model — the deliverable (EXPANDED)

The original said "lift as a function of airspeed², wing area, and AoA." That's
underspecified — it doesn't say what *shape* the curves are, which is where all the
feel lives. Concrete model:

**Reference airframe** (F/A-18-class, racing configuration):

```
mass            13 400 kg      wing area S     38.0 m²
span b          11.4 m         mean chord c̄   3.50 m
aspect ratio    3.42           Ixx/Iyy/Izz     31 000 / 205 000 / 230 000 kg·m²
thrust (dry)    2 × 62 kN      thrust (AB)     2 × 98 kN   → T/W 1.49
fuel            4 900 kg       spool τ         1.2 s (idle→mil), 0.4 s (mil→AB)
```

**Lift.** Attached-flow `CL = CL0 + CLα·α`, `CLα = 4.2 /rad`. Blend to a separated
flat-plate model `CL = 2 sin α cos α` through a sigmoid centred at **α_crit = 17°**,
fully separated by 26°. Buffet (airframe shake + HUD AoA bracket flash) from 13°.
The blend — not a hard cutoff — is what makes the stall recoverable instead of a cliff.

**Drag.** `CD = CD0 + k·CL² + CD_wave`, `CD0 = 0.021`, `k = 1/(π·e·AR) = 0.114` with
`e = 0.82`. Wave drag: zero below M 0.80, rising to a peak near M 1.05, relaxing
above — this is what makes the transonic region *feel* like a wall you punch through.

**Moments, not rates.** Pitch/roll/yaw come from `q̄·S·(coefficients)`, so control
authority scales with dynamic pressure as a *consequence* of the model rather than a
hand-tuned curve. Includes static stability `Cm_α`, pitch damping `Cm_q`, roll damping
`Cl_p`, dihedral `Cl_β`, weathercock `Cn_β`.

**Fly-by-wire layer (NEW — not in the original, and load-bearing).** Real fighters are
not flown by direct surface deflection. Default assist mode: pitch stick commands
*normal acceleration*, roll stick commands *roll rate*, with a 9 G limiter and auto
turn-coordination. `Direct` mode exposes the raw surfaces. Without this the aircraft is
authentic and miserable; with it, it's authentic and fast. Toggle in Settings.

**G-physiology.** G-load integrates over time, not instantaneously: grey-out → tunnel
vision → G-LOC with a ~4 s onset delay and a recovery lag. Negative G gives redout.
Control authority degrades before blackout so the player gets a warning they can feel.

**Ground effect.** Induced drag factor scales by `1 − 0.62·(1 − h/b)²` below one span.

**Atmosphere.** ISA temperature lapse, exponential density, speed of sound from
temperature — so Mach, thrust, and lift all fall off with altitude coherently.

**Turbulence.** Dryden-style filtered noise, per-track intensity.

---

## 3. HUD (ADDITIONS)

Everything in the original list, plus three omissions that are exactly what make a HUD
read as real rather than as a game overlay:

- **Flight path marker (velocity vector).** The single most recognisable element of a
  real HUD. Shows where the jet is *going*, not where it points. Essential for gate
  lineups, and it makes AoA visible as the gap between the FPM and the boresight.
- **Pitch ladder locked to world space** — climb/dive angles must overlay the real
  horizon at the correct angular scale, or it's decoration.
- **AoA indexer** — the three-light approach indexer, doubling as the stall cue.

Rendered to a 2D canvas, not DOM. Real HUDs are stroked vector graphics; canvas is the
right primitive and costs ~0.4 ms.

---

## 4. Everything else — as originally specified

Controls (keyboard / gamepad / HOTAS remapping / live input visualiser), terrain LOD,
dynamic sky, camera modes with speed-based FOV, menus, loading screen, race mechanics,
damage, ghosts, post-race stats, synthesized audio.

**Audio note:** no external files means engine tone, afterburner, wind, and warning
tones are Web Audio oscillator/noise graphs. Voice callouts become HUD text + tones.

---

## 5. Definition of done

- [ ] Holds 60 fps at 1080p on integrated graphics at Medium
- [ ] Stall, spin, and recovery are all reachable and survivable
- [ ] Transonic drag rise is perceptible without looking at the Mach readout
- [ ] AI finishes a clean lap within 15% of a competent human, with zero rubber-banding
- [ ] Physics identical at 30 fps and 144 fps (fixed timestep, verified)
- [ ] Loads with no network requests other than the pinned three.js CDN import
