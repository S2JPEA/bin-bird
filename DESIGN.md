# Bins Chickens — Design & Build Plan

A planning document for a single-file, no-build 3D browser game. You play an
Australian white ibis ("bin chicken") scavenging junk food from bins around a
stylised Brisbane CBD, levelling up until you can fly across the river to South
Bank. This document covers the design and the build approach only — it is not
the implementation.

---

## 1. Guiding constraints

These shape every decision below and should not be violated:

- **One file.** A single `index.html`, opened directly from disk, no server, no
  build step, no bundler. All CSS, JS, and markup inline.
- **Three.js from a CDN**, pinned to an exact version (e.g. a specific
  `three@0.16x.x` build) via a `<script type="importmap">` or a plain module
  script tag. No other runtime dependency.
- **No external assets.** Every mesh is procedural low-poly geometry; every
  sound is synthesised with the Web Audio API; all text is inline.
- **Runs on an average laptop.** Integrated-GPU friendly: low triangle counts,
  a tight draw-call budget, a modest world size.
- **No accounts, no network at runtime.** Persistence is cookies only, with a
  visible reset.
- **Workable on first run.** No placeholder functions, no "TODO: fill in"
  stubs. Every system listed here must function end to end.

---

## 2. Single-file architecture

Organise the one file into clearly commented sections, roughly in this order:

1. `<head>` — meta, inline `<style>` (screens, HUD, buttons, crosshair, fonts
   via system font stack only).
2. DOM for the overlays: start screen, HUD, win screen, fail screen, pause.
3. Module script:
   - **Config block** — all tunable numbers in one object (see §9). Single
     source of truth so tuning is one place.
   - **Save/load** — cookie read/write/reset.
   - **Audio engine** — lazy-initialised `AudioContext`, SFX functions.
   - **World builder** — city, river, landmarks, bins.
   - **Entities** — ibis, junkies, police (factory functions returning a mesh +
     state).
   - **Systems** — movement, camera, collision, spawning, progression, combat.
   - **Game state machine** — `START → PLAYING → (WIN | FAIL) → …`.
   - **Main loop** — `requestAnimationFrame`, fixed-ish timestep with delta
     clamping.

**Shared state object.** A single `game` object holds mode, references to
`scene`/`camera`/`renderer`, the ibis, arrays of enemies and bins, and the
player stats (`food`, `level`, `health`/dollars, `strength`, `canFly`,
`flightRange`). Every system reads and writes this object; the HUD renders from
it each frame. Keeping state centralised is what lets each system be built and
tested against a known structure.

---

## 3. Game systems

### 3.1 World generation
- **Layout.** A fixed-size ground plane split by a **river** running diagonally
  (the Brisbane River loops around the CBD). The **CBD** sits on one bank; **South
  Bank** is a thin strip of green/landmark geometry on the far bank — the win
  target, not an explorable area.
- **Street grid.** A regular grid of roads (flat dark boxes/decals) with
  rectangular **city blocks** between them. Buildings are extruded boxes of
  varied height/colour placed on blocks. **Queen Street Mall** is a distinct
  pedestrian strip (no road, paved, lined with shopfronts) — a recognisable,
  named corridor.
- **Landmarks.** A handful of stylised recognisable shapes: the Mall, a clock
  tower / City Hall-ish block, the river bridge line of sight to South Bank, and
  the **Hungry Jack's** safe zone in the Mall (a flagged building with a glowing
  marker).
- **Bins.** Scattered along streets and the Mall; each bin is a simple mesh with
  a state (full/empty) and a food value. Placement is procedural but seeded so
  the layout is stable between runs. Emptied bins **refill on a timer (~25 s)** —
  this is the anti-soft-lock mechanism given that the world persists across
  respawn (§7 decision 9): without refill, repeated deaths could leave nothing to
  scavenge.
- **Interactions out:** feeds collision bounds (buildings block movement), the
  water boundary (river), spawn points (bins, enemies), and the safe zone test.

### 3.2 Ibis movement and flight
- **Third-person, keyboard + mouse.** WASD / arrows for move, mouse (or A/D) to
  turn/orbit, space to flap/ascend, shift maybe to sprint-waddle. The ibis is a
  low-poly model (body, long neck, beak, legs, wings) assembled from primitives.
- **Two modes:**
  - **Walking** — grounded, gravity pins it to the terrain height, collides with
    buildings and bins.
  - **Flying** — **unlocks at level 2**; space gains altitude, forward momentum
    carries it. Flight is limited by a **stamina bar** (§7 decision 8) that drains
    airborne and refills on the ground; its max is set by **`strength`, which is
    the flight-range stat** (§7 decision 3) and grows each level. This is the
    mechanism that makes "fly across the river" a progression gate: the river is
    only clearable at the **win-ready level 5**, not from minute one.
- **Animation.** Cheap procedural: bob the body while walking, flap wings via
  rotation while flying, waggle the neck on squawk. No skeletal rigs.
- **Interactions out:** position drives camera, collision, bin pickup proximity,
  enemy proximity, safe-zone test, and the water/win check.

### 3.3 Camera
- **Follow camera** behind and slightly above the ibis, smoothed (lerp toward a
  target offset) so it trails rather than snaps. Mouse adjusts yaw/pitch within
  clamps. Pulls in when near buildings to avoid clipping (simple raycast or
  distance clamp). Switches to a slightly wider framing in flight so the river
  crossing is readable.

### 3.4 Food and level progression
- **Collect food** by moving over / into a bin that still has food; the bin
  empties (and may refill on a timer). Each pickup adds to a `food` counter and
  plays a squawk.
- **Levelling.** Crossing a food threshold raises `level`. Each level:
  - raises **max health** (more dollars of buffer; base $5, +$2/level),
  - raises **strength**, which **governs flight range** (§7 decision 3) by
    extending the max stamina bar,
  - at **level 2, unlocks flight**; each further level extends range toward the
    ~48-unit river crossing, clearable at the **win-ready level 5**.
- **Interactions out:** writes stats to `game`, triggers HUD updates, level-up
  SFX + playful text, and gates flight. This is the spine the win condition
  hangs off.

### 3.5 Enemy behaviour
- **Junkies.** Wander the streets; when the ibis is within a detect radius they
  **approach and "ask for $1 or a ciggie"** (speech-bubble text + audio). On
  contact they **drain $1 of health** per hit (with a short cooldown so contact
  isn't an instant wipe). Simple steering: move toward ibis when in range, idle
  wander otherwise.
- **Police.** Patrol near key areas; when the ibis gets close they **shoo it
  away** — a repulsion impulse that pushes the ibis back / interrupts
  scavenging, plus a whistle SFX and text. They damage nothing; they are a
  movement/area denial nuisance.
- **Shared:** both are capped in count and only updated when reasonably near the
  player (distance cull) to protect the frame budget. Neither can act inside the
  safe zone.
- **Interactions out:** junkies modify health; police modify position; both read
  the safe-zone flag to stand down.

### 3.6 Health, failure, safe zone, respawn
- **Health is measured in dollars.** Starts at a base value; max rises with
  level. Junkie hits subtract dollars.
- **Safe zone — Hungry Jack's (Queen St Mall).** While inside the zone radius:
  enemies cannot damage the ibis and **health recharges** over time up to the
  current max. A clear visual/audio cue on entry.
- **Failure states:**
  - **Health hits $0** → fail screen → respawn at the Hungry Jack's with health
    restored (stats/level/food persist per the save rules).
  - **Falling in the water** → fail. If this happens mid-win-attempt (came up
    short on the river crossing), it's the "didn't make it across" loss.
- **Win state:** reach South Bank's landing zone on the far bank via flight →
  win screen.
- **Interactions out:** the failure/win checks read position (water, South Bank
  zone) and health each frame and drive the state machine.

### 3.7 Save and reset (cookies)
- **What the cookie stores:** just `level`, `food`, and `health` — a 3-number
  JSON blob. `strength`, `canFly`, and max stamina/flight range are all **derived
  from `level`** via `CONFIG`, so they need not be stored.
- **World state is NOT cookied.** Per §7 decision 9, the world (emptied bins,
  enemy positions) persists **in memory across a respawn within a session**, but a
  full page reload regenerates it from seed and restores only the 3 saved stats
  (player resumes at the safe zone). This keeps the save tiny while honouring
  "world persists on respawn".
- **How:** one cookie, JSON-stringified, URL-encoded, written on meaningful
  change (level-up, pickup, death) and/or on a throttle; read on load to resume.
- **Reset:** a button on the start screen (and/or pause) that expires the cookie
  and restores defaults. Confirm before wiping.
- **Interactions out:** load feeds initial `game` stats; save is called by the
  progression and failure systems.

### 3.8 Audio (Web Audio API)
- **One `AudioContext`**, created/resumed on first user gesture (start button) to
  satisfy autoplay policy.
- **Synthesised SFX only:** ibis **squawk** (noise burst + pitch-swept
  oscillator, slight randomisation so repeats aren't identical), **food pickup**
  blip, **level-up** fanfare, **junkie mumble**, **police whistle**, **splash**
  on water fail, **win** sting. A master gain + mute toggle.
- **Interactions out:** called by movement (occasional idle squawk), pickups,
  level-ups, combat, and state transitions. Purely a feedback layer — no system
  depends on audio succeeding.

### 3.9 UI screens and text
- **Start screen** titled **"Bins Chickens"** with a one-line premise, the
  controls, a **Play** button, a **Reset save** button, and a mute toggle.
- **HUD** (in-world overlay): **health (in $)**, **food collected**, **level**,
  and a flight/stamina indicator; a subtle objective nudge ("Level up to fly to
  South Bank").
- **Win screen** and **Fail screen**, each with flavour text and a Play-again /
  Continue button. Fail distinguishes "mugged to $0" from "went for a swim".
- **Floating text / toasts** for pickups, level-ups, junkie lines, police shoos
  — the main vehicle for the **Australian humour** (dry, scruffy, self-deprecating
  ibis voice).
- **Interactions out:** reads `game` state every frame; its buttons drive the
  state machine and save/reset.

### 3.10 How the systems fit together (data flow)
```
input ─▶ movement ─▶ position ─▶ { collision, camera, bin proximity,
                                     enemy proximity, safe-zone test,
                                     water/South-Bank test }
bins ─▶ food ─▶ progression ─▶ { health max, strength, flight } ─▶ HUD + save
enemies ─▶ health ─▶ failure check ─▶ state machine ─▶ { respawn, fail screen }
state machine ◀─ win check (South Bank) ─── position
every change ─▶ HUD render + audio cue (+ throttled cookie save)
```
The centralised `game` object is the hub; systems never call each other
directly, they read/write shared state and the loop orders them.

---

## 4. Build and verification sequence

Each milestone leaves the file runnable and independently testable. Build in
this order so a failure is always localised to the stage just added.

1. **Boot + render loop.** Three.js from CDN, scene, camera, renderer, a ground
   plane, a resize handler, an rAF loop. *Verify:* page opens, you see the
   ground, no console errors, resizing works.
2. **Ibis + walking + follow camera.** Procedural ibis mesh, WASD movement over
   the ground, smoothed follow camera, mouse look. *Verify:* you can waddle
   around and the camera trails smoothly.
3. **Static city + collision.** Street grid, blocks, buildings, the Mall,
   landmarks, river plane, South Bank strip. Building collision + world bounds.
   *Verify:* recognisable CBD, can't walk through buildings, can reach the
   river's edge.
4. **Bins + food + HUD.** Place bins, pickup on proximity, `food` counter, HUD
   showing food/health/level. *Verify:* walking over full bins increments food
   and empties the bin; HUD updates.
5. **Progression + flight.** Food thresholds raise level → health max, strength,
   and (at threshold) unlock/extend flight with a range budget. Flying controls.
   *Verify:* enough food levels you up; flight unlocks and range grows with
   level.
6. **Water + South Bank = win/fail boundaries.** Falling in water → fail;
   reaching South Bank in flight → win. *Verify:* a short flight falls in and
   fails; a long-enough flight (after levelling) lands and wins.
7. **Safe zone + respawn + health.** Hungry Jack's zone recharges health and
   blocks damage; $0 health → fail → respawn at HJ's. *Verify:* health refills in
   the zone; forcing health to 0 respawns you there.
8. **Enemies.** Junkies (approach, ask, drain $1 on contact w/ cooldown) and
   police (shoo/repel near key spots); both stand down in the safe zone; distance
   culling. *Verify:* junkies cost you dollars, police push you off, neither acts
   inside HJ's.
9. **Audio.** Wire squawks and all SFX to their triggers; mute toggle; context
   resumed on Play. *Verify:* each event makes its sound; mute silences all.
10. **Save/reset + screens + polish.** Cookie save/load/reset; start/win/fail
    screens; floating humour text; final tuning pass against §9. *Verify:*
    refresh resumes progress; reset wipes it; all screens and transitions read
    correctly.

A quick **performance check** (frame time, draw calls via the renderer's `info`)
should be repeated after stages 3, 8, and 10 — the three points where object
counts jump.

---

## 5. Key technical decisions and risks

### 5.1 Draw-call budget
- **Target:** keep steady-state draw calls low (order of a few dozen to low
  hundreds), comfortably within integrated-GPU headroom.
- **Tactics:**
  - **Instanced meshes** (`InstancedMesh`) for repeated objects — buildings,
    bins, street segments, and enemies that share geometry/material.
  - **Merge static geometry** (roads, footpaths, ground details) into a few
    merged buffers rather than many small meshes.
  - **Share materials**; a small fixed palette of flat/low-cost materials
    (`MeshLambert`/`MeshStandard` with no textures) lit by one directional +
    one ambient/hemisphere light.
  - **Distance culling / simple LOD** for enemies and far bins; the renderer's
    frustum culling handles off-screen geometry.
- **Risk:** naively one-mesh-per-building blows the budget. Decide the
  instancing approach *before* stage 3, because retrofitting it is costly.

### 5.2 City layout approach
- **Grid-driven procedural with fixed landmarks.** A deterministic seed places
  blocks/buildings on a grid; the Mall, Hungry Jack's, the river, and South Bank
  are hand-placed fixed features so the city is recognisable and the win route is
  stable. This balances "stylised Brisbane" against "no external assets".
- **Risk:** over-detailing. Keep it readable and cheap — silhouette and layout
  sell "Brisbane CBD" more than facade detail. Cap world size (see §9) so the
  player can't wander into empty, expensive sprawl.

### 5.3 Collision and movement
- **Cheap, axis-aligned.** Buildings and bins as AABBs; ibis as a point/sphere.
  Ground follow via terrain height lookup (flat or simple heightfield). Resolve
  by push-out, not physics engine. No external physics library.
- **Risk:** tunnelling at speed/flight — clamp delta time and cap speed so a
  single frame can't skip through a wall.

### 5.4 Water and boundaries
- **River as a zone test, not physics.** Define the river region
  (polygon/half-plane in XZ) and a shoreline height. "In the water" = ibis
  position over the river region **and** below the shoreline height (i.e. not
  flying above it). South Bank landing = position inside the far-bank zone above
  a minimum altitude/landed.
- **World edge:** invisible walls or a soft push-back so the player can't leave
  the map; falling off the map is not a lose condition — only the river is.
- **Risk:** ambiguous edges (is skimming the bank a fail?). Define the river
  region, shoreline height, and South Bank zone as explicit numbers in config and
  tune them together with flight range.

### 5.5 Cookie size and persistence
- **Well within limits.** A cookie caps around ~4 KB; the save is only **three
  integers** as JSON (level, food, health — §3.7) — kilobytes to spare. **Do not**
  serialise world state or positions; the world persists in memory across respawn
  and regenerates from seed on reload (§7 decision 9).
- **Risks:** `file://` cookie quirks in some browsers, and users with cookies
  disabled. Mitigate: wrap read/write in try/catch, degrade gracefully to an
  in-memory session (game still playable, just won't persist), and consider
  `localStorage` as the mechanism only if cookies prove unreliable from
  `file://` — but the brief calls for `document.cookie`, so cookies are primary.
- **Reset must truly expire** the cookie (past date, same path), not just blank
  it.

### 5.6 Audio autoplay
- **`AudioContext` must be created/resumed inside a user gesture** (the Play
  button). Build the engine to no-op safely if audio is unavailable so gameplay
  never blocks on it.

### 5.7 Frame-rate independence
- All movement, drains, recharges, and flight budgets scale by **delta time**
  with a clamp, so behaviour is consistent across laptop refresh rates and
  survives tab-switch stalls.

---

## 6. Scope guardrails

To stay true to the brief, the plan deliberately **excludes**: multiplayer,
networking/leaderboards, external art/audio, a map/minimap beyond the HUD nudge,
inventory/shops, more enemy types than junkies and police, and any build
tooling. If a feature isn't in §3, it isn't in v1.

---

## 7. Design decisions (resolved)

The knobs that determine whether the game is fun and fair — now decided. Numbers
live in §9; each decision notes what still needs playtest tuning in stages 5–10.

1. **Food needed to win / per level — DECIDED: short run (~5–10 min).** Win-ready
   at **level 5**; per-level food requirement rises gently (4 / 6 / 9 / 12),
   ≈31 food total. *Tune:* exact thresholds vs. bin density so pace feels right.
2. **Flight unlock & range curve — DECIDED.** Flight unlocks at **level 2**;
   range (via stamina, see decision 8) grows per level so the **river (~48 units)
   is only clearable at level 5** — L4 range falls short (splash), L5 clears. This
   is the single most important relationship. *Tune:* river width and the stamina
   curve together in stage 6.
3. **What `strength` does — DECIDED: flight range.** Strength is the stat that
   governs how far/high the ibis flies; levelling it up is the path to crossing
   the river. Not a damage-resistance stat. *Tune:* range-per-level curve.
4. **Health economy — DECIDED: cheeky but forgiving.** Start **$5**, **+$2/level**
   → **$13** at L5; junkie **$1/hit** with ~2 s cooldown; Hungry Jack's heals
   **+$1 per ~1.5 s**. Deaths should be rare. *Tune:* heal rate and cooldown.
5. **Enemy density & aggression — DECIDED: light, fixed.** ~**5 junkies** (always
   outrunnable), ~**3 police**; modest detect/shoo radii. *Tune:* counts and radii
   for comedic-nuisance feel without frustration.
6. **Bin density & refill — DECIDED.** ~**18 bins**, food value 1–2, **refill
   ~25 s** after emptying. Refill is also the anti-soft-lock mechanism given
   decision 9. *Tune:* count and refill time vs. run length.
7. **World size — DECIDED: compact CBD.** ≈**120 × 120 units**, ~5 × 5 blocks.
   *Tune:* extent vs. navigability and draw-call headroom.
8. **Flight model — DECIDED: stamina/fuel bar.** Drains airborne, refills on the
   ground, max grows with level; running dry over water = splash. *Tune:* drain/
   refill rates and max-stamina curve.
9. **World persistence on respawn — DECIDED: world persists.** On death→respawn
   the city/bins/enemies keep their state (bins stay emptied, enemies keep
   positions). Bins refilling on a timer (decision 6) prevents repeated deaths
   from soft-locking the run. **Note:** only *player stats* are cookied (see
   §3.7/§5.5); world state persists in memory across a respawn but regenerates
   from seed on a full page reload.
10. **Difficulty framing — DECIDED: fixed.** No enemy scaling with level; the
    player's own growth (health, flight range) is the power curve.

> All §9 numbers are **starting defaults**, not frozen balance — stages 5–10
> exist to tune them against real play.

---

## 8. Definition of done (v1)

- Opens from a single `index.html`, no server/build, no console errors on an
  average laptop.
- Start → play → win and start → play → fail (both $0 and water) all reachable.
- All of §3 functioning with no stubs; HUD accurate; audio and humour present.
- Progress persists across refresh and can be reset.
- Holds a smooth frame rate with draw calls within budget during normal play.

## 9. Config block — starting default profile

Every gameplay number lives in one `CONFIG` object (single source of truth; no
magic numbers buried in systems). These are the **starting defaults** derived
from the §7 decisions, to be tuned in stages 5–10 — nothing here is frozen.

**Progression / win gate**
- Win-ready level: **5**; flight unlocks at level: **2**.
- Food to reach next level: L1→2 **4**, L2→3 **6**, L3→4 **9**, L4→5 **12**
  (≈**31** cumulative to win-ready).

**Flight (strength = range, via stamina)**
- Horizontal flight speed: **~12 units/s** (constant).
- Max stamina (seconds airborne) by level: L2 **2.0**, L3 **2.8**, L4 **3.6**,
  L5 **4.4** (+0.8/level from unlock). Ground refill: **~1.5×** the drain rate.
- Range per full dash ≈ speed × stamina: L2 ≈ **24**, L4 ≈ **43**, L5 ≈ **53**.
- **River crossing width (near bank → South Bank landing): ~48 units** → L4
  falls short (splash), L5 clears with margin. Tune width + stamina together.

**Health economy (dollars)**
- Max health: base **$5** at L1, **+$2/level** → **$13** at L5; respawn restores
  to current max.
- Junkie damage: **$1/hit**, per-junkie cooldown **~2 s**.
- Hungry Jack's: heal **+$1 per ~1.5 s** up to max; **no damage** inside the zone.

**Enemies (fixed difficulty)**
- Junkies: count **~5**, detect radius **~8 units**, approach speed **below** ibis
  walk speed (always escapable); idle-wander when out of range.
- Police: count **~3**, patrol near Mall/landmarks, shoo radius **~5 units**,
  moderate repel impulse, **no damage**. Both stand down in the safe zone;
  distance-culled.

**World / bins (compact CBD)**
- World extent ≈ **120 × 120 units** (CBD + river + South Bank strip).
- Street grid ≈ **5 × 5 blocks**; buildings **~40–60** via `InstancedMesh`.
- Bins: **~18**, food value **1–2**, **refill ~25 s** after emptying.
- River region + shoreline height + South Bank landing zone as explicit numbers
  (tied to the ~48-unit crossing width above).

**Camera / audio**
- Follow offset behind + above the ibis, lerp smoothing; wider framing in flight.
- Audio master volume + mute flag.
