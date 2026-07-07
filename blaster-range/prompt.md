# "Blaster Range" — Exact FPS Recreation Scenario

Recreate BLASTER RANGE in this sandbox: a fast-twitch arcade first-person blaster set in a
dusk-lit low-poly downtown block. The player sprints, slides, and jumps through a walled city
arena — hollow brick/trim/metal buildings with walkable roofs, a central concrete plaza, and a
skyline of fog-faded towers — cutting down patrolling mannequin training bots with a full-auto
sci-fi rifle for score. Everything below is a measured, verified spec of the finished game;
reproduce it exactly. Use a generic sci-fi/urban training identity only; no protected game,
brand, character, logo, weapon trademark, or level.

## Working Rules

- You are running unattended; no one can answer questions. Decide, proceed, and note assumptions
  in your final report. Never stop at uncertainty, never end with only a plan, and never end early
  over time or context concerns — the deliverable is the working game.
- Follow the local agent guide (`AGENTS.md` or `CLAUDE.md`).
- Every requirement below is hard. Numeric values in this spec are NOT suggestions — they are the
  shipped tuning; the data blocks below are normative — transcribe every row and value exactly
  into `src/constants.ts` and the modules that consume them.
- Parallelize independent reads and searches; batch related edits instead of micro-editing.
- Build exactly what this spec describes: no extra features, abstractions, or defensive code.
- Do not create git commits; leave version control untouched.

## Assets

- The seeded pack is `public/assets/quaternius-urban-v1` (Quaternius, CC0). Use only files under
  `public/assets`; do not download, generate, model, replace, or import assets.
- Sub-packs used by the game:
  - `downtown-city/…` — Downtown City MegaKit: facade panels, props, decals, buildings, streets.
  - `animation-library/…` — Universal Animation Library: `UAL1_Standard.glb` (mannequin mesh +
    43 animation clips), `Mannequin_F.glb` (female mannequin mesh). Tracks retarget across both
    by bone name (shared UE-style skeleton: `root/pelvis/spine_01/…/hand_r/Head/neck_01`).
  - `sci-fi-guns/…` — Sci-Fi Essentials Kit guns: `Gun_Rifle.glb` (player), `Gun_Pistol.glb`
    (bots). Life-size, textured PBR, grip at origin, barrel along raw −x.
- Primitives are allowed only for invisible collision, helpers, and procedural VFX/HUD.
- Calibration facts (verified at runtime; bake into `ASSET_TUNING`):
  - UAL mannequin meshes are authored facing +z (bind AND animated pose) → `yawFix: 180`.
  - Sci-fi guns point the barrel along −x → `yawFix: -90` (barrel ends up on local −z).
  - `Gun_Rifle` raw muzzle at x −0.752, bore height ~y 0.15, optic top ~0.28.
  - Megakit facade panels (`Brick_*`, `Trim_*`, `Metal_*` walls/windows) are 2×3×0.2 slabs with
    the front face at local z=0 and the body behind it.
  - `Street_Asphalt_9x9` tiles are corner-origin planes whose surface sits 0.15 below the pivot.
  - `Entrance_Concrete_2x2` is a raw 2×1.007×2 base-origin block; `Trim_Column_Center` is 0.672
    wide; `Floor_4x4`/`Roof_4x4` are 4×4 plates.
  - Ground decals have their mesh below the origin and need small positive y offsets (see table).

## Build Requirements

- Vite + TypeScript + PlayCanvas engine (template already wired; engine 2.20+).
- `vite.config.ts`: host `127.0.0.1`,
  `watch: { usePolling: true, interval: 300 }`, `resolve.conditions: ['development']`.
- `index.html`: bare page, `<canvas id="application-canvas">`, module script `/src/app.ts`,
  `html, body { margin: 0; height: 100%; overflow: hidden; }`.
- Modular TypeScript in `src/`, one file per system:

```
src/app.ts        wiring only: app boot, asset load, arena build, game start
src/constants.ts  ALL gameplay tuning (MOVE, CAM, GUN, BOT, TERRACES, PLAYER, ARENA, ASSET_TUNING)
src/assets.ts     GLB container cache, spawn() pivot factory, animTracks()
src/input.ts      pointer-lock keyboard/mouse singleton
src/collision.ts  AABB math: groundY heightfield, moveBox collide-and-slide, ray tests
src/player.ts     first-person controller (yaw body / pitch head), slide, crouch, shake
src/weapon.ts     rifle viewmodel + FP arms + hitscan firing
src/bots.ts       training-bot state machine, animation, return fire
src/arena.ts      level construction, lighting, skybox, waypoints/spawns
src/vfx.ts        procedural tracers/sparks/bursts/dust/telegraph
src/audio.ts      procedural WebAudio sfx (no audio assets exist)
src/hud.ts        DOM/CSS HUD injected at runtime
src/game.ts       state machine + per-frame orchestration
```

- Name visible gameplay entities by role (`Enemy_3`, `WeaponRifle`, `Wall_N_4`, `Cover_12_…`,
  `Tower_7`) so runtime queries stay readable.
- Expose the app as `(window as any).app` for devtools verification.

## Runtime Verification

- Test in a real browser as a player would: use the Playwright or Chrome DevTools MCP tools to
  load the dev server, drive keyboard/mouse input, read the console, and take screenshots. Claims
  about gameplay must come from observing the running game, not from reading source.
- Measure, don't guess: evaluate script in the running page to read the scene graph — world
  positions, bounding boxes, forward directions, entity counts. Use screenshots for composition
  and facing, numbers for everything else.
- Facing and grounding are measured with fresh `instantiateRenderEntity` probes, not live bots —
  the bot update loop rewrites pivot yaw every frame even at `timeScale 0`.
- Foot-slide is measurable: sample a stance foot bone's world velocity along the travel direction
  (stance = lowest 30% of foot heights); ~zero drift = no skating. `Jog_Fwd_Loop` covers ~3 m/s
  at speed 1, hence the `anim.speed = groundSpeed / 3` rule below.

---

# EXACT SPECIFICATION

## 1. Boot (`app.ts`, `assets.ts`)

- `new Application(canvas)`, `FILLMODE_FILL_WINDOW`, `RESOLUTION_AUTO`,
  `maxPixelRatio = window.devicePixelRatio`, window resize → `app.resizeCanvas()`.
- `boot()`: `await loadAssets(app, Object.keys(ASSET_TUNING))` → `buildArena(app)` →
  `startGame(app, arena, canvas)` → `app.start()`.
- `assets.ts` keeps a `Map<string, Asset>` of loaded containers.
  `loadAssets` runs `app.assets.loadFromUrl('${ASSET_ROOT}/${key}.glb', 'container', …)` for every
  key in parallel (Promise.all), rejecting with the key name on failure.
- `animTracks(key)` returns a `Record<name, AnimTrack>` built from the container's
  `resource.animations` assets (`track.name ?? asset.name`).
- `spawn(key, name)` returns a pivot `Entity` whose origin sits on the floor, with one child:
  the container's `instantiateRenderEntity()` carrying the calibrated
  `setLocalScale(scale)`, `setLocalPosition(0, yOffset, 0)`, `setLocalEulerAngles(0, yawFix, 0)`
  from `ASSET_TUNING` (default `{scale:1, yOffset:0, yawFix:0}`). All placement code positions
  the pivot; per-instance stretch is applied to `entity.children[0]`.

## 2. Constants (`constants.ts`) — normative tuning data

Reproduce every group and value below in `src/constants.ts` with these exact names and numbers.
`# …` are notes, not values; comma triples are x, y, z components.

```
ASSET_ROOT  /assets/quaternius-urban-v1

MOVE
baseSpeed        7
sprintSpeed      10
crouchSpeed      3.5
accel            60
airAccel         15
friction         10
jumpVel          6.5
gravity          20
slideBoost       13.5
slideTime        0.75
slideCooldown    1.0
slideFriction    2.5
slideMinSpeed    4
eyeHeight        1.6
crouchEyeHeight  0.95
slideEyeHeight   0.7
eyeLerp          12
radius           0.35
height           1.7
crouchHeight     1.0
landThump        8

CAM
fov           90
aimFov        62
fovLerp       12
sens          0.11
aimSensScale  0.5
pitchClamp    89

GUN
shotsPerSec     10
magSize         30
reserve         300
reloadTime      1.3
damage          12
headMult        2
range           120
spreadBase      0.4                 # degrees
spreadBloom     0.45                # per shot
spreadMax       3.5
spreadRecover   5                   # deg/s
spreadAimScale  0.3
recoilKick      0.05                # weapon back per shot
recoilPitch     0.4                 # camera degrees per shot
recoilRecover   10
swayAmt         0.0035              # gun lag per mouse px
swayMax         0.035
swayLerp        10
bobAmp          0.013               # walk bob meters
lowAmmoFrac     0.2
hipPos          0.26, -0.27, -0.34
hipEuler        0, -8, 4            # shouldered cant, lerped out while aiming
sprintPos       0.2, -0.28, -0.42
sprintEuler     55, 10, -12         # tac sprint: gun-only rotation, muzzle up
aimPos          0, -0.245, -0.28    # optic aperture centered on the crosshair
muzzleOffset    0, 0.15, -0.75      # rifle grip at origin: raw muzzle x -0.752, bore ~y 0.15

BOT
count          8
armoredCount   2
hp             60
armoredHp      130
speed          2.4
turnLerp       11
respawnDelay   2.5
telegraphTime  0.9
fireInterval   2.6
fireRange      30
damage         6
hitChanceNear  0.3                  # falls off linearly to 0 at fireRange
scoreBody      100
scoreHead      150
staggerTime    0.25
flashTime      0.12
fadeTime       1.4
headY          1.55
headRadius     0.26
bodyHalf       0.42, 0.873, 0.3
moveHalf       0.35, 0.873, 0.35
stuckTime      1.5
waypointReach  0.8
pauseChance    0.35                 # chance to stop and look around at a waypoint
faceRange      26                   # track the player inside this distance

PLAYER
maxHp      100
lowHpFrac  0.3
spawnPos   10, 0, 21
spawnYaw   30

ARENA
half          24
floorY        0
skyTop        '#1c2857'             # dusk gradient, canvas css colors
skyHorizon    '#e0885e'
sunColor      1, 0.82, 0.6
sunIntensity  2.6
fogColor      0.62, 0.47, 0.42
fogStart      35
fogEnd        110
```

`TERRACES` defines the walkable elevation as data: plateaus are solid boxes, ramps resolve as a
heightfield in `groundY()` (`h0` at the axis-min edge, `h1` at axis-max). Burg-style village:
thin h=2.4 plateaus are building walls (unjumpable, block sight); roofs are floor-thick decks at
y=2.4 so interiors stay walkable; slightly-taller 0.6×0.6 posts wrap corners and flank doors.
`facade` names the facade family. Every row:

```
plateaus: x0, x1, z0, z1, h, facade
# B4 warehouse: solid block, walkable roof via the west ramp
-17,   -8,    11,   19,   3.2, metal
-17.3, -16.7, 10.7, 11.3, 3.4, metal
-8.3,  -7.7,  10.7, 11.3, 3.4, metal
# B1 NW house (doors: S at x -16..-14, E at z -16..-14; window W)
-19,   -11,   -19,   -18.4, 2.4, brick1
-19,   -18.4, -18.4, -16,   2.4, brick1
-19,   -18.4, -14,   -11.6, 2.4, brick1
-11.6, -11,   -18.4, -16,   2.4, brick1
-11.6, -11,   -14,   -11.6, 2.4, brick1
-19,   -16,   -11.6, -11,   2.4, brick1
-14,   -11,   -11.6, -11,   2.4, brick1
-19.3, -18.7, -11.3, -10.7, 2.6, brick1
-11.3, -10.7, -11.3, -10.7, 2.6, brick1
# B2 north house (aligned N+S doors at x -3..0; window E)
-7,   -3,   -19,   -18.4, 2.4, trimA
0,    1,    -19,   -18.4, 2.4, trimA
-7,   -6.4, -18.4, -11.6, 2.4, trimA
0.4,  1,    -18.4, -16,   2.4, trimA
0.4,  1,    -14,   -11.6, 2.4, trimA
-7,   -3,   -11.6, -11,   2.4, trimA
0,    1,    -11.6, -11,   2.4, trimA
-7.3, -6.7, -11.3, -10.7, 2.6, trimA
0.7,  1.3,  -11.3, -10.7, 2.6, trimA
# B3 NE house (doors: S at x 8..10, W at z -16..-14; window E)
5,    13,   -19,   -18.4, 2.4, brickT
5,    5.6,  -18.4, -16,   2.4, brickT
5,    5.6,  -14,   -11.6, 2.4, brickT
12.4, 13,   -18.4, -16.5, 2.4, brickT
12.4, 13,   -14.5, -11.6, 2.4, brickT
5,    8,    -11.6, -11,   2.4, brickT
10,   13,   -11.6, -11,   2.4, brickT
6.7,  7.3,  -11.3, -10.7, 2.6, brickT
10.7, 11.3, -11.3, -10.7, 2.6, brickT
# B5 south house (aligned N+S doors at x -3..-1; window E)
-4,   -3,   11,   11.6, 2.4, trimB
-1,   4,    11,   11.6, 2.4, trimB
-4,   -3.4, 11.6, 18.4, 2.4, trimB
3.4,  4,    11.6, 13,   2.4, trimB
3.4,  4,    15,   18.4, 2.4, trimB
-4,   -3,   18.4, 19,   2.4, trimB
-1,   4,    18.4, 19,   2.4, trimB
-4.3, -3.7, 10.7, 11.3, 2.6, trimB
-0.7, -0.1, 10.7, 11.3, 2.6, trimB
# B6 east house (aligned W+E doors at z -5..-3; window S)
14,   21,   -8,   -7.4, 2.4, metal
14,   16,   -0.6, 0,    2.4, metal
18,   21,   -0.6, 0,    2.4, metal
14,   14.6, -7.4, -5,   2.4, metal
14,   14.6, -3,   -0.6, 2.4, metal
20.4, 21,   -7.4, -5,   2.4, metal
20.4, 21,   -3,   -0.6, 2.4, metal
13.7, 14.3, -0.3, 0.3,  2.6, metal
20.7, 21.3, -0.3, 0.3,  2.6, metal
# B7 west house (doors: E at z -3..-1, S at x -19..-17)
-21.5, -15.5, -6,   -5.4, 2.4, brickNW
-21.5, -19,   1.4,  2,    2.4, brickNW
-17,   -15.5, 1.4,  2,    2.4, brickNW
-21.5, -20.9, -5.4, 1.4,  2.4, brickNW
-16.1, -15.5, -5.4, -3,   2.4, brickNW
-16.1, -15.5, -1,   1.4,  2.4, brickNW
-15.8, -15.2, -6.3, -5.7, 2.6, brickNW
-15.8, -15.2, 1.7,  2.3,  2.6, brickNW
# B8 southeast house (doors: N at x 14..16, W at z 15..17; window E)
12,   14,   12,   12.6, 2.4, trimA
16,   20,   12,   12.6, 2.4, trimA
12,   20,   19.4, 20,   2.4, trimA
12,   12.6, 12.6, 15,   2.4, trimA
12,   12.6, 17,   19.4, 2.4, trimA
19.4, 20,   12.6, 15,   2.4, trimA
19.4, 20,   17,   19.4, 2.4, trimA
11.7, 12.3, 11.7, 12.3, 2.6, trimA
19.7, 20.3, 11.7, 12.3, 2.6, trimA

ramps: x0, x1, z0, z1, axis, h0, h1, facade
# B4 roof ramp: low edge at the west wall so the street can pass along it
-23.8, -17, 13, 17, x, 0, 3.2, metal
```

`ASSET_TUNING` maps each model key to `{scale, yOffset, yawFix}`; its key list is also the asset
load manifest. Keys without a `/` are under `downtown-city/`. Every row:

```
key, scale, yOffset, yawFix
sci-fi-guns/Gun_Rifle,            1,   0,     -90
sci-fi-guns/Gun_Pistol,           1,   0,     -90
animation-library/UAL1_Standard,  1,   0,     180
animation-library/Mannequin_F,    1,   0,     180
Prop_ACUnit,                      1.8, 0,     0
Prop_Bollard,                     1,   0,     0
Prop_Planter_Single,              1,   0,     0
Sidewalk_Planter,                 1.5, 0,     0
Entrance_Concrete_2x1,            1.2, 0,     0
Entrance_Concrete_2x2,            2,   0,     0
Stairs_Entrance_Concrete,         1.2, 0,     0
Trim_Wall_Guard,                  1,   0,     0
Metal_Column_Small_Center,        1,   0,     0
Brick_Plain_3,                    1,   0,     0
Brick_Plain_3_noWear,             1,   0,     0
Brick_TopTrim,                    1,   0,     0
Brick_Window_Square_Single,       1,   0,     0
Brick_Window_Trim_Single,         1,   0,     0
Trim_FirstFloor_Wall,             1,   0,     0
Trim_FirstFloor_Window_001,       1,   0,     0
Trim_Plain_3,                     1,   0,     0
Trim_Window,                      1,   0,     0
Metal_FirstFloor_Window,          1,   0,     0
Trim_Column_Center,               1,   0,     0
Floor_4x4,                        1,   0,     0
Roof_4x4,                         1,   0,     0
Street_Asphalt_9x9,               1,   0,     0
Metal_Plain_3,                    2,   0,     0
Building_Small_1,                 1,   0,     0
Building_Medium_2_001,            1,   0,     0
Building_Large_2,                 1,   0,     0
Decal_Crosswalk,                  1,   0.165, 0
Decal_ArrowStraight,              1,   0.148, 0
Decal_BrokenLine_Straight,        1,   0.148, 0
Decal_DoubleYellow_Straight,      1,   0.128, 0
Decal_Stop,                       1,   0.145, 0
Decal_Slow,                       1,   0.145, 0
Prop_ManholeCover,                1,   0.02,  0
Prop_Drain,                       1,   0.005, 0
Sidewalk_Straight_3m,             1,   0.16,  0
```

## 3. Input (`input.ts`)

Singleton `input` object: `keys: Set<string>` (KeyboardEvent.code), accumulated `mdx/mdy` mouse
deltas, `fire`/`aim` booleans, `locked` flag, `key(...codes)` any-of test, `consume()` zeroes the
deltas (called once per frame at the end of update). Listeners: keydown adds (and
`preventDefault()` while locked), keyup deletes, window blur clears keys+buttons, mousemove
accumulates `movementX/Y` only while locked, mousedown/up map button 0→fire and 2→aim (down only
while locked, up always), contextmenu prevented, `pointerlockchange` sets `locked` and clears all
input when lock is lost.

## 4. Collision (`collision.ts`)

- `AABB = { minX, minY, minZ, maxX, maxY, maxZ }`.
- `groundY(x, z)`: max of 0, every containing plateau's `h`, and every containing ramp's
  `h0 + (h1-h0) * t` where `t` is the normalized position along the ramp axis. Ramps are walkable
  heightfield only — they have no colliders.
- `moveBox(pos, half, delta, vel, boxes)` — per-axis collide-and-slide, axis order x, z, y:
  add `delta[ax]`, then for every overlapping box eject to the NEAREST face
  (`pos[ax] = face ∓ half[ax] ∓ EPS`, `EPS = 0.001`) and zero `vel[ax]`. Overlap test uses `+EPS`
  tolerance so a box resting flush on a collider top never reads as penetrating. Ejecting upward
  on the y axis sets `grounded = true` (returned). A box that starts a frame slightly penetrated
  must never pop out the far side — always nearest face, never movement direction.
- `boxesOverlap(pos, half, boxes)` boolean; `rayAABB` slab test returning entry distance or null
  (handles parallel rays); `rayBoxes` nearest hit over a list; `raySphere` standard tca/d² form
  returning entry distance or null (null if behind origin).

## 5. Player (`player.ts`)

- Entity hierarchy: `Player` body entity (world position + yaw) → `PlayerHead` child (pitch +
  eye height + shake offsets). `pos` is FEET position. Camera and weapon pivot ride the head.
- Look: `yaw -= mdx * CAM.sens * aimScale`, `pitch -= mdy * CAM.sens * aimScale`, pitch clamped
  ±89. `aimScale` is written by the weapon (lerped 1 → 0.5 while aiming).
- Wish dir from yaw + WASD, normalized. Grounded: accelerate `accel*dt` toward wish, clamp
  horizontal speed to target (crouch 3.5 / sprint 10 / base 7); friction `1 - friction*dt` only
  when no input. Airborne: `airAccel*dt`, horizontal speed capped at
  `max(sprintSpeed, slideBoost) = 13.5`.
- Slide: crouch PRESSED (edge) while grounded, not sliding, cooldown expired, sprint held, and
  `hspeed > baseSpeed*0.8` → set horizontal vel to `slideBoost` along wish (or facing), 0.75 s
  timer, `crouched = true`, emit `onSlide`. During slide: friction `1 - slideFriction*dt`, spawn
  slide dust every 0.05 s while grounded; end when timer or speed < `slideMinSpeed`, then 1.0 s
  cooldown. Jump cancels a slide but keeps momentum.
- Crouch (held Ctrl/C): stand up only with headroom (test a standing-height box against
  colliders). Height 1.7 standing / 1.0 crouched; collision box is `radius 0.35` half-width,
  centered at `pos + height/2`.
- Jump: Space while grounded → `vel.y = 6.5`. Gravity 20.
- Integration: `moveBox` with the current box, then the ramp heightfield on top: if falling into
  `gy = groundY(x,z)`, snap up and ground — unless already grounded and the step is > 0.5 (ramp
  side reads as a wall: revert x/z). If grounded, moving down-slope with `pos.y - gy < 0.35`,
  stick to the slope. Landing after a fall faster than `landThump 8` adds shake 0.045 and fires
  `onLand`.
- `sync()`: eye height lerps (12/s) between 1.6 / 0.95 (crouch) / 0.7 (slide); shake decays
  `0.35/s`, capped 0.12, applied as random ±shake/2 offsets on head x/y; recoil pitch decays
  `recoilPitch -= (10 * recoilPitch + 2) * dt` (exponential + linear) and adds to head pitch
  before clamping.

## 6. Weapon & viewmodel (`weapon.ts`)

Entity `WeaponPivot` parented to the player head at `hipPos`/`hipEuler`. Children:

- `WeaponRifle` = `spawn('sci-fi-guns/Gun_Rifle')`.
- `RedDot`: 0.005-scale sphere, emissive red `MAT.red`, `castShadows: false`, child of the GUN at
  local `(0, 0.245, -0.34)` — on the optical axis so it centers in ADS at any depth, z matched to
  the sight housing so it reads inside the glass from hip view too. This is the ADS aim reference;
  the HUD crosshair fades out while aiming.
- `FPBody`: full-body first-person arms — `spawn('animation-library/UAL1_Standard')` riding the
  pivot so the hands stay on the rifle through sway/bob/ADS. Render child gets an `anim`
  component (`activate: true`) with two states from the UAL1 container: `'Aim'` =
  `Pistol_Aim_Neutral` (loop) and `'Reload'` = `Pistol_Reload` (no loop; record its clip
  duration). Body local rotation `(20, 18, 0)` (hunched pitch + bladed yaw keep torso/shoulders
  behind the near plane), local position `(-0.152, -1.484, -0.013)` (numerically solved so
  `hand_r` lands on the rifle grip = pivot origin). Scale bones `Head` and `neck_01` to 0.001.
  Clone every mesh material and tint diffuse `(0.55, 0.58, 0.64)` (neutral gray gear).
  Cache `hand = findByName('hand_r')`.
- `Muzzle` locator at `muzzleOffset (0, 0.15, -0.75)`; under it `MuzzleFlash` (box, `MAT.spark`,
  disabled) and `MuzzleLight` (omni, color `(1, 0.8, 0.4)`, intensity 6, range 7, no shadows,
  disabled).

Per-frame `update(dt, firing, aiming, reloadKey, bots, colliders)`:

- `aimT` lerps toward aiming at `fovLerp 12`; camera fov = lerp(90, 62, aimT); player
  `aimScale` = lerp(1, 0.5, aimT).
- Reload: `startReload()` if key pressed (guards: not reloading, mag not full, reserve > 0).
  It sets `reloadT = 1.3`, plays the arm gesture time-fitted to the reload:
  `bodyAnim.speed = reloadClipLen / reloadTime; baseLayer.transition('Reload', 0.08)`. On
  completion, top up the mag from reserve, restore `speed = 1` and `transition('Aim', 0.15)`,
  fire `onReload(false)`. Auto-reload triggers when firing on an empty mag (with one `onEmpty`
  click per trigger hold).
- Fire cadence: `fireT` countdown at 10/s. Bloom recovers at 5°/s only between bursts
  (`fireT < -0.03`), not mid-burst. `kick` decays 0.35/s.
- Spread getter: `(spreadBase + bloom) * lerp(1, spreadAimScale 0.3, aimT)`.
- Sway: gun lags the camera — target `clamp(-mdx * swayAmt, ±swayMax)` (and `+mdy` for Y),
  scaled by `lerp(1, 0.25, aimT)`, lerped at 10/s.
- Bob: `bobA` fades in with ground speed (`min(1, hs/sprintSpeed)`, 8/s); `bobT += dt*(4+hs*0.9)`;
  figure-8: `bobX = sin(bobT)*bobAmp*bobA*sw`, `bobY = -|cos(bobT)|*bobAmp*1.3*bobA*sw`; plus idle
  breathe `sin(t*1.8)*0.0025*(1-bobA)*sw`.
- Tactical sprint: `sprinting = moving && hs > baseSpeed+1 && !aiming && !firing && !reloading`;
  `sprintT` lerps at 9/s. Landing dip `sin(landT*π)*0.05` (landT set to 1 by the game on hard
  landings, decays 3.5/s). Reload dip/roll: `e = sin((1 - reloadT/reloadTime)*π)` → dip `e*0.09`,
  roll `e*-35°`. Slide cant: roll lerps to 10° while sliding (10/s).
- Pose composition (exact): position = lerp(hip, aim, aimT), then lerp(→sprintPos, sprintT),
  then add `swayX+bobX` (x), `swayY+bobY+breathe−rDip−land` (y), `kick` (z). Pivot euler:
  `x = kick*60 + swayY*100 + rDip*150 + land*130 + hipEuler.x*(1−aimT)`,
  `y = swayX*70 + hipEuler.y*(1−aimT)`,
  `z = rRoll + swayX*−90 + slideRoll + sin(bobT)*1.5*bobA*(1+sprintT) + hipEuler.z*(1−aimT)`.
- Tac-sprint gun rotation is applied to the GUN entity only (about its grip):
  `gun.setLocalEulerAngles(sprintEuler * sprintT)` — NOT to the pivot (rotating the pivot swings
  the whole body into view). The hand follows with the same world-space delta: capture the
  `hand_r` rest local rotation once after t > 0.3 s; while `sprintT > 0.01` and not reloading set
  `handLocal = inv(parentRot) · (pivotRot · sprintQ · inv(pivotRot)) · parentRot · handRest`,
  else restore `handRest`. (Bone writes during app update land after the animator; the static aim
  pose doesn't rewrite bones each frame, so the rest pose must be restored manually.)
- `fire()`: decrement ammo; bloom `+0.45` capped 3.5; kick `+0.05` capped 0.12; camera
  `recoilPitch += 0.4`; shake `+0.008`; `onShot`. Ray from the HEAD position along head.forward
  perturbed by a uniform disk in the right/up plane: `a = rand*2π`, `r = sqrt(rand) * spread(rad)`.
  Nearest hit among world colliders (`rayBoxes`, max 120) and every living bot: head sphere
  (center `headC`, r 0.26) vs body AABB, with head bias — body wins only if
  `tBody < tHead − 0.3` so grazing the head counts as head. Tracer from the MUZZLE world position
  to the end point; on bot hit call `damage(12 * (head ? 2 : 1), head, hitPos)`; on world hit
  within range spawn sparks. Muzzle flash: enable, random scale `(0.12+rand*0.08)²` × 0.32 long,
  random z-roll, light on, both disabled via 40 ms `setTimeout`.
- `reset()` restores ammo/reserve, cancels reload (restore anim speed 1 + Aim transition),
  zeroes all transient state, disables flash/light.

## 7. Bots (`bots.ts`)

8 bots, indices 0–7; the first 2 are ARMORED. Standard = `UAL1_Standard` mannequin tinted diffuse
`(1, 0.55, 0.2)` + emissive `(0.4, 0.16, 0.03)` (hi-viz orange); armored = `Mannequin_F` tinted
`(0.45, 0.5, 0.62)` + emissive `(0.35, 0.05, 0.05)` (slate). All materials cloned per instance.
Bots share the UAL1 anim tracks (retarget by bone name). State machine per bot:

- `waiting` (timer; initial stagger `0.1 + idx*0.3`, respawn 2.5 s) → pick a spawn point (prefer
  those > 12 from the player, random among them) → `telegraph` 0.9 s (pulsing cyan ring VFX at
  the spot) → `materialize`.
- `materialize`: spawn mesh, tint, `hp = 60 / 130`, set anim component on the render child with
  states `Idle→Idle_Loop` (loop), `Run→Jog_Fwd_Loop` (loop), `Aim→Pistol_Idle_Loop` (loop),
  `Death→Death01` (one-shot). Attach a `Gun_Pistol` to the `hand_r` bone at local position
  `(0.03, 0.03, 0)` and local euler `(79.4, -2.7, -1.2)` (solved numerically against the ORGANIC
  Pistol_Idle pose — a forced transition gets overridden within a frame, so solve while the state
  machine is naturally in Aim). Bot muzzle locator at `(0, 0.12, -0.35)` under the pistol. Spawn
  pop-in: scale from 0.01 with ease-out-back over 0.35 s
  (`s = 1 + 2.7(t−1)³ + 1.7(t−1)²`). Random walk-speed multiplier `0.85 + rand*0.3` per life.
- `alive` (every frame): hit-flash restores base emissive when `flashT` crosses 0. Patrol toward
  the current waypoint; within `waypointReach 0.8` pick a new one (> 6 away, random), with 35%
  chance to pause 0.6–1.8 s and look around (yaw drifts `sin(animT*1.4)*12°`).
  - Stop-to-shoot: `tracking = playerDist < 26`; `engaging = tracking && fireT < 0.9`;
    `halted = staggered || paused || engaging` → target speed 0, else `2.4 * speedMul`.
  - Velocity lerps toward target at 8/s (smooth accel/decel). Move via `moveBox` with
    `moveHalf (0.35, 0.873, 0.35)`; then follow the terrain by sampling `groundY` at ALL FOUR box
    corners and taking the max (a bot straddling a ledge must not drop through it); reject steps
    taller than 0.5. Record the ACTUAL displacement `(mvx, mvz) = (Δx/dt, Δz/dt)` — facing and
    animation must follow where the bot really went, not intended velocity, or wall-sliding bots
    moonwalk.
  - Stuck watchdog: if waypoint distance hasn't shrunk by 0.02 for 1.5 s, reroute.
  - Facing: if `|mv| > 0.5` face travel (`atan2(-mvx, -mvz)`); else if tracking face the player;
    else if paused drift. `lerpAngle` (shortest arc) at `turnLerp 11`.
  - Animation: `aimA` lerps (6/s) toward `tracking && ms < 0.6`; desired state = `Run` if
    `ms > 0.6`, else `Aim` if `aimA > 0.5`, else `Idle`; `baseLayer.transition(state, 0.15)` on
    change. Stride pace follows real ground speed so feet don't slide:
    `anim.speed = (state === 'Run') ? max(0.6, ms / 3) : 1` (Jog_Fwd_Loop covers ~3 m/s at 1×).
  - Keep hit volumes in sync: `headC = pos + (0, 1.55, 0)`, body AABB from
    `bodyHalf (0.42, 0.873, 0.3)` with minY at feet.
  - Return fire (only while `playing` and `aimA ≥ 0.5`): every `2.6 * (0.75 + rand*0.5)` s, from
    the pistol muzzle world position toward the player eye; skip if > 30 away or LOS blocked
    (`rayBoxes` hit closer than `dist − 0.5`). Hit chance `0.3 * (1 − dist/30)`; a miss aims off
    by ±1.5 x/z and +0–1.5 y. Orange tracer + muzzle sparks + `onBotShot`; on hit
    `onPlayerHit(6)`.
- `damage(amt, head, hitPos)`: hp down; flash emissive `(2.5, 0.12, 0.08)` on headshot else
  `(1.8, 0.12, 0.08)` for 0.12 s; stagger 0.25 s; sparks at the hit point (red ×7 on head, yellow
  ×4 on body); `onDamaged(head)`. At 0 hp → `dying`: 1.4 s fade (`blendType = BLEND_NORMAL`,
  opacity = remaining fraction) on top of a `Death` transition (0.08), death burst VFX,
  `onKill(head)` → then destroy and `waiting` 2.5 s. Dead bots are removed from targeting
  immediately (`target.alive = false`).

Bots update in `ready` state too (with `playing = false` so they never shoot) — the range looks
alive behind the menu overlay.

## 8. Arena (`arena.ts`)

Returns `{ colliders: AABB[], waypoints: [x,z][], spawns: [x,z][] }`. Construction order:

1. **Ground**: 6×6 grid of `Street_Asphalt_9x9` — `for gx,gz in -27..18 step 9`, position
   `(gx+9, 0.15, gz+9)` (corner-origin tiles). One floor collider
   `{-60, -2, -60, 60, 0, 60}`.
2. **Interior wood floors** (`Floor_4x4` child-scaled `(w/4, 1, d/4)`, y 0.03) at
   `[cx, cz, w, d]`: `[-15,-15,6.8,6.8], [-3,-15,6.8,6.8], [9,-15,6.8,6.8], [0,15,6.8,6.8],
   [17.5,-4,5.8,6.8], [-18.5,-2,4.8,6.8], [16,16,6.8,6.8]`.
3. **Perimeter ring**: 13 `Metal_Plain_3` panels per side (scale 2 → 4 wide × 6 tall) at
   `c = i*4 − 24`, distance `RING = half + 0.6 = 24.6`, yawed to face inward (N 0°, S 180°,
   W 90°, E −90°). Four wall colliders just inside at `±(half − 0.2)`, y −1..12.
4. **COVER** placements (model, x, z, yaw, y — y defaults 0) — each spawned, positioned, yawed,
   and given a collider from its measured world AABB (union of all mesh instance AABBs). All
   models under `downtown-city/`; yaws axis-aligned for walls so colliders match visuals:

```
model, x, z, yaw, y
Entrance_Concrete_2x2,      0,     0,    0
Stairs_Entrance_Concrete,   6,    -5,    0
Stairs_Entrance_Concrete,  -6,    -3,    90
Trim_Wall_Guard,           -18.7, -15,   90
Trim_Wall_Guard,            0.7,  -15,   90
Trim_Wall_Guard,            12.7, -15.5, 90
Trim_Wall_Guard,            3.7,   14,   90
Trim_Wall_Guard,            17,   -0.3,  0
Trim_Wall_Guard,            19.7,  16,   90
Prop_ACUnit,               -16.5, -13.5, 0,  3.2
Prop_ACUnit,                11,   -13.5, 15, 3.2
Prop_ACUnit,               -10,    17.5, 30, 3.2
Prop_Planter_Single,       -17.3, -12.7, 20
Prop_ACUnit,               -12.3, -17.5, 0
Prop_Planter_Single,       -5.5,  -17.3, 30
Prop_ACUnit,                11.3, -17.3, 45
Prop_ACUnit,                6,    -12.4, 0
Prop_ACUnit,                2.5,   17.3, 0
Prop_Planter_Single,        19.3, -6.7,  30
Prop_ACUnit,               -20.3, -4.5,  0
Prop_Planter_Single,        18.3,  18.1, 0
Stairs_Entrance_Concrete,   16.5, -14,   20
Entrance_Concrete_2x1,      21,    9,    30
Prop_ACUnit,                21,    9,    70, 1.2
Entrance_Concrete_2x1,     -17,   -22.5, 10
Prop_Bollard,               5,    -23.2, 0
Prop_Bollard,              -20,   -14,   0
Entrance_Concrete_2x1,      14,    21,   20
Entrance_Concrete_2x1,      13.5, -11,   90
Sidewalk_Planter,          -20.5,  21.5, 10
Sidewalk_Planter,           21,   -18,   45
Sidewalk_Planter,          -4.5,  -23,   0
Sidewalk_Planter,           22.8,  2,    90
Prop_ACUnit,               -14,   -17.5, 20, 3.2
Prop_ACUnit,               -5.5,  -17,   40, 3.2
Metal_Column_Small_Center,  8,    -17,   0,  3.2
Prop_ACUnit,               -2.5,   17.5, 0,  3.2
Prop_ACUnit,               -15,    13,   0,  3.2
Prop_ACUnit,                19.5, -6.5,  0,  3.2
Metal_Column_Small_Center, -20.5,  0,    0,  3.2
Prop_ACUnit,                18.5,  18.5, 20, 3.2
Trim_Wall_Guard,           -17,   -18.8, 0,  3.2
Trim_Wall_Guard,           -13,   -18.8, 0,  3.2
Trim_Wall_Guard,           -18.8, -15,   90, 3.2
Trim_Wall_Guard,           -5,    -18.8, 0,  3.2
Trim_Wall_Guard,           -1,    -18.8, 0,  3.2
Trim_Wall_Guard,            7,    -18.8, 0,  3.2
Trim_Wall_Guard,            11,   -18.8, 0,  3.2
Trim_Wall_Guard,            12.8, -15,   90, 3.2
Trim_Wall_Guard,           -2,     18.8, 0,  3.2
Trim_Wall_Guard,            2,     18.8, 0,  3.2
Trim_Wall_Guard,           -14,    18.7, 0,  3.2
Trim_Wall_Guard,           -11,    18.7, 0,  3.2
Trim_Wall_Guard,            16,   -0.2,  0,  3.2
Trim_Wall_Guard,            20,   -0.2,  0,  3.2
Trim_Wall_Guard,           -21.3, -4,    90, 3.2
Trim_Wall_Guard,            14,    19.8, 0,  3.2
Trim_Wall_Guard,            18,    19.8, 0,  3.2
Stairs_Entrance_Concrete,   5.5,   4.5,  160
Stairs_Entrance_Concrete,  -5.5,  -6.8,  20
Prop_Planter_Single,        7.5,   2.5,  70
Prop_Bollard,              -3.5,   23,   0
Prop_Bollard,              -1,     23,   0
Prop_Bollard,               1.5,   23,   0
Prop_Bollard,               4,     23,   0
Prop_Bollard,               13,   -23,   0
Prop_Bollard,               15.5, -23,   0
Prop_Bollard,               18,   -23,   0
Prop_ACUnit,                22.5, -13,   0
Prop_Planter_Single,       -13.5, -23,   0
Sidewalk_Planter,          -21,    6.5,  100
Entrance_Concrete_2x1,     -19,   -11,   30
Entrance_Concrete_2x1,      10,    5,    70
Prop_ACUnit,                11.5, -12.5, 200
Prop_ACUnit,               -18,   -16.8, 30
```

5. **DECKS** (roofs, exterior stair steps, door awnings): `Entrance_Concrete_2x2` child-scaled
   `(2, 0.8/1.007, 2)` → a 4×0.8×4 slab with base origin at the given y; collider
   `x±2, y..y+0.8, z±2`. Placements, `(x, z, y)` triples:

```
(-17,-17,2.4) (-13,-17,2.4) (-17,-13,2.4) (-13,-13,2.4)
(-5,-17,2.4) (-1,-17,2.4) (-5,-13,2.4) (-1,-13,2.4)
(7,-17,2.4) (11,-17,2.4) (7,-13,2.4) (11,-13,2.4)
(-2,13,2.4) (2,13,2.4) (-2,17,2.4) (2,17,2.4)
(16,-6,2.4) (20,-6,2.4) (16,-2,2.4) (20,-2,2.4)
(-19.5,-4,2.4) (-16.5,-4,2.4) (-19.5,0,2.4) (-16.5,0,2.4)
(14,14,2.4) (18,14,2.4) (14,18,2.4) (18,18,2.4)
# grand stairs up B2 from the square (hop north onto the roof)
(-5,-6.6,0) (-5,-7.8,0.8) (-5,-9,1.6)
# stairs up B5 from the square (hop south onto the roof)
(2,6.6,0) (2,7.8,0.8) (2,9,1.6)
# door awnings over the square-facing doors
(9,-9.6,2.4) (-2.6,9.6,2.4)
```

6. **Terraces → visuals**. Facade families map to megakit panel pairs `[wall, window]`:
   `brick1: [Brick_Plain_3, Brick_Window_Square_Single]`,
   `trimA: [Trim_FirstFloor_Wall, Trim_FirstFloor_Window_001]`,
   `brickT: [Brick_TopTrim, Brick_Window_Trim_Single]`,
   `metal: [Metal_Plain_3, Metal_FirstFloor_Window]`,
   `trimB: [Trim_Plain_3, Trim_Window]`,
   `brickNW: [Brick_Plain_3_noWear, Brick_Window_Square_Single]`.
   `facadeRow(fam, x0, z0, x1, z1, h, nx, nz)` tiles `n = max(1, round(len/1.6))` panels along
   the segment, each child-scaled `(len/n/2, h/3, 0.8)` (panel front lands exactly on the segment
   line = the collider face), yaw `atan2(nx, nz)` degrees, using the window panel when
   `n ≥ 3 && i % 2 === 1`. Per plateau:
   - `w ≤ 0.75 && d ≤ 0.75` → post: `Trim_Column_Center` child-scaled `(w/0.672, h/3, d/0.672)`.
   - `w > 2 && d > 2` → solid block (the warehouse): 4 outward facade rows + `Roof_4x4`
     child-scaled `(w/4, 1, d/4)` at `h + 0.2`.
   - otherwise (thin wall) → facade rows on BOTH long faces plus both end caps.
   - Collider per plateau: `{x0, −1, z0, x1, h − 0.25, z1}` — the top stops 0.25 short of `h` so
     walkers coming up a ramp clear the lip (the walkable surface comes from `groundY`, not this
     box).
7. **Ramp visual**: a stretched pitched slab — `Entrance_Concrete_2x2` child-scaled
   `(hypot(len, rise)/2, 0.3/1.007, wide/2)` with child local position `(0, −0.15, 0)`
   (recenters the base-origin block), entity euler `(0, axis==='x' ? 0 : −90, ±atan2(rise, len))`
   and position `(centerX, rise/2 − 0.14, centerZ)`. No collider (heightfield handles walking).
8. **DECOR** (no colliders; model, x, z, yaw — all models under `downtown-city/`):

```
model, x, z, yaw
Prop_ManholeCover,           -4,    -3,    50
Decal_Slow,                   14,    4,    80
Prop_ManholeCover,           -13,    5,    140
Decal_Stop,                   7,     22,   30
Prop_Drain,                  -14,    1,    130
Decal_ArrowStraight,          7,    -7,    0
Decal_ArrowStraight,         -8,     7,    60
Prop_Drain,                  -6,    -21.5, 0
Prop_Drain,                   22.5,  18,   80
Prop_ManholeCover,            3,     3.2,  0
Decal_ArrowStraight,          8.5,  -8.5,  210
Decal_ArrowStraight,         -9.5,   9,    30
Prop_Drain,                  -22.9, -4,    40
Prop_ManholeCover,            13.5, -19.5, 120
Prop_Drain,                   22.8,  16,   200
Prop_ManholeCover,            5.2,  -19.5, 300
Prop_Drain,                  -19.5,  11.5, 250
Prop_ManholeCover,           -11,   -22.9, 60
Prop_Drain,                   1,     15,   0
Decal_Crosswalk,              0,    -9.8,  0
Decal_Crosswalk,              12,    2,    90
Decal_Crosswalk,             -11,    4,    90
Decal_Crosswalk,              6,     20,   0
Decal_Crosswalk,             -21,   -16,   90
Decal_BrokenLine_Straight,    6.5,   21.5, 0
Decal_BrokenLine_Straight,   -16.5, -21.5, 0
Decal_DoubleYellow_Straight,  5.5,  -22.6, 0
Decal_DoubleYellow_Straight, -8,     22.6, 0
Decal_BrokenLine_Straight,   -22.6,  10,   90
Sidewalk_Straight_3m,        -6,    -8.5,  0
Sidewalk_Straight_3m,        -3,    -8.5,  0
Sidewalk_Straight_3m,         3,    -8.5,  0
Sidewalk_Straight_3m,         6,    -8.5,  0
Sidewalk_Straight_3m,        -6,     8.5,  180
Sidewalk_Straight_3m,         3,     8.5,  180
Sidewalk_Straight_3m,         6,     8.5,  180
```

9. **Out-of-bounds skyline** (no colliders): prefab towers from
   `[Building_Small_1, Building_Medium_2_001, Building_Large_2]`. Deterministic hash per tower
   `h = (ti * 2654435761) % 97` (ti = spawn counter) picks the type (`big` ring uses only the two
   larger: index `1 + h%2`; near ring `h%3`), height stretch `childScaleY = 0.85 + (h%5)*0.1`,
   and yaw jitter `(h%3 − 1) * 6°` on top of the inward-facing yaw. Near ring:
   `t = −40..40 step 16` at `±37` on each side (N faces 0°, S 180°, W 90°, E 270°). Far ring
   (big towers): `t = −54..54 step 27` at `±58`.
10. **Lighting/atmosphere**:
    - Sun: directional, color `(1, 0.82, 0.6)`, intensity 2.6, `castShadows`, shadowDistance 70,
      shadowResolution 2048, shadowBias 0.2, normalOffsetBias 0.05, euler `(38, 35, 0)`.
    - Skybox: procedural 256² canvas cubemap — side faces are a vertical gradient
      `skyTop → skyHorizon` with white star specks in the top 55% (160×frac stars, alpha
      0.35–0.9, size 0.8–2 px); +y face solid-ish `skyTop` fully starred; −y face flat
      `skyHorizon`. `setSource([side, side, top, bottom, side, side])`; also feed
      `EnvLighting.generateAtlas(EnvLighting.generateLightingSource(sky))` into
      `scene.envAtlas` for sky-tinted ambient. Camera tone mapping `TONEMAP_ACES`.
    - Accent omni lights `[x, y, z, r, g, b, range, intensity]`:
      `[0,2.6,0, 1,0.6,0.25, 9,1.6]` (plaza, warm) and interior glows at
      `[-15,2.1,-15]`, `[-3,2.1,-15]`, `[9,2.1,-15]`, `[17.5,2.1,-4]`, `[16,2.1,16]` warm
      `(1,0.6,0.25)` and `[0,2.1,15]`, `[-18.5,2.1,-2]` cool `(0.5,0.8,1.3)` — all range 8,
      intensity 1.6, no shadows.
    - Linear fog color `(0.62, 0.47, 0.42)`, start 35, end 110.
11. **Waypoints** (bot patrol graph; all standable — assert none intersect a collider using a
    0.36×0.85×0.36 half-extent probe at `groundY + 0.85`):

```
waypoints, (x, z) pairs:
(-14,-21.5) (-9,-21.5) (-1.5,-21.5) (2.5,-21.5) (10,-21.5) (17,-21)
(20.5,-13) (19,6) (22,15) (21.8,21.8)
(17.5,-4) (11.5,-4) (22.5,-4) (10.5,1)
(10,21) (3,21) (-2,21) (-10,21.5) (-13,21.5)
(-21.5,-19) (-21.5,-8) (-22.7,-2) (-23.2,10) (-23.2,20)
(-18.5,-2) (-13.5,-2) (-18,5)
(-6,2) (6,-2) (0,-6.5) (-2,5)
(-15,-9.5) (-1.5,-9.5) (9,-9.5) (-9,-9) (-2,8.5)
(-8.5,-15) (2.5,-15)
(16,16) (9,16) (15,9)
(-15,-15) (-1.5,-15) (9,-15) (-2,15)

spawns, (x, z) pairs:
(-21,-21) (20,-21) (22,21.5) (-16,21)
(0,-21.5) (19,-4) (-15,-15) (9,-15)
```

## 9. VFX (`vfx.ts`)

Procedural primitives only, parented to a `VFX` root, self-updating on app `update`. Emissive
unlit materials (diffuse black, `useLighting: false`): tracer `(0.4, 1.6, 2.2)` cyan, botTracer
`(2.2, 0.5, 0.2)` orange, spark `(2.2, 1.8, 0.5)`, red `(2.2, 0.3, 0.2)`, dust
`(0.55, 0.52, 0.48)`, telegraph `(0.3, 1.8, 1.2)`.

- `tracer(from, to, mat)`: one 0.035×0.035×len box at the midpoint, `lookAt(to)`, life 0.07 s
  (skip if < 0.2 long).
- `sparks(pos, mat=spark, n=5, speed=4)`: n 0.06³ cubes, random up-biased directions,
  speed × (0.5+rand), gravity, life 0.3–0.45 s.
- `deathBurst(pos)`: 14 cubes 0.14³ (every 3rd red, rest spark) from `pos + 0.8y`, speed 3–7,
  gravity, life 0.5–0.75 s.
- `slideDust(pos)`: one 0.16³ dust cube near the feet, upward drift, life 0.4 s.
- `telegraph(pos, dur)`: emissive cyan cylinder at y+0.04, growing `0.3 → 1.7` diameter with a
  12% pulse at 25 rad/s, height 0.06, life = dur.
- Update: gravity `vel.y −= 12·dt`; moving fx translate; shrinking fx multiply xy scale by
  `0.85 + 0.15·(1 − t/life)` per frame. `clearVfx()` destroys everything (used on restart).

## 10. Audio (`audio.ts`)

No audio assets exist — synthesize everything with WebAudio (init lazily on first click; master
gain 0.22; one shared white-noise buffer). Two helpers: `tone(type, f0→f1 exp ramp, dur, vol,
delay)` and `noise(dur, vol, bandpass freq, Q, delay)`. Recipes:

```
fire()        square 850→220 0.09s 0.5  + noise 0.06s 0.35 @2600 Q0.8
botFire()     sawtooth 400→150 0.12s 0.25
empty()       square 1800→1400 0.03s 0.25
reloadStart() square 480→300 0.05s 0.3  + noise 0.08s 0.2 @1200
reloadEnd()   square 700→950 0.05s 0.35
hit()         sine 1250→1100 0.045s 0.4
headshot()    sine 1750→1500 0.07s 0.5  + sine 2350→2100 0.07s 0.3 delay 0.02
kill()        sawtooth 700→140 0.18s 0.4 + noise 0.15s 0.3 @900 Q1
headKill()    sawtooth 1000→180 0.22s 0.45 + sine 2100→1600 0.12s 0.3
hurt()        sawtooth 160→70 0.16s 0.5 + noise 0.1s 0.3 @300 Q1
death()       sawtooth 380→45 0.55s 0.5
land()        sine 130→60 0.09s 0.4
slide()       noise 0.3s 0.3 @700 Q0.6
```

## 11. HUD (`hud.ts`)

DOM injected at construction (style tag + `#hud` fixed overlay, `pointer-events: none`, font
`"SF Mono", Menlo, monospace`, color `#e8f4ff`, z-index 10). Styling language: soft cyan glow,
NO dark borders anywhere.

- **Crosshair** `#xh` at screen center: 4 bars (3×12 px, `#eafcff`, border-radius 1.5px,
  `box-shadow: 0 0 6px rgba(120,220,255,.9)`) + 4 px round center dot with the same glow.
  `setSpread(deg, fov, aim)`: project the spread angle to pixels —
  `px = tan(deg°) / tan(fov/2°) * (innerHeight/2)`, `gap = 10 + px*2.2` (amplified for
  readability); bars translate outward by gap (side bars rotated 90°). The whole crosshair fades
  while aiming: `opacity = max(0, 1 − aim*1.6)` — the rifle's red dot takes over as the ADS
  reference.
- **Hitmarker** `#hitm`: 4 diagonal ticks (2.5×11 px, rounded, translate ±(6–8, 4–14) px,
  rotate ±45°), white with white glow; `.head` variant gold `#ffd34d` with stronger gold glow.
  `hitmarker(head)` sets duration 0.3 s (head) / 0.2 s (body); per-frame animation with
  `f = remaining/duration`: `opacity = min(1, f*2.5)`, `transform = scale(1 + f²*0.6)` — pops in
  large, contracts and fades.
- **Ammo** bottom-right: mag count 34 px bold over `RESERVE n` 16 px at 75%; `.low`
  (mag ≤ 20%) turns the count `#ff5a48` with a 0.5 s opacity pulse.
- **Reload bar**: 140×5 px above the ammo block, cyan `#7fd8ff` fill 0→100% over the reload, with
  a letter-spaced `RELOADING` label; hidden otherwise.
- **Health** bottom-left: number 26 px bold + 180×8 px bar (green `#5cf28a`, width transition
  .12s); `.low` (≤ 30%) turns number and bar red with the pulse.
- **Score** top-right (24 px bold under a letter-spaced `SCORE` small label) and a gold
  score-pop line below it (`+100` / `+150 WEAK POINT`), visible 0.8 s.
- **Damage vignette**: full-screen radial gradient (transparent 42% → `rgba(255,30,10,.55)`),
  opacity = `2 × remaining` of a 0.5 s timer on hit.
- **Overlays** (flex-centered, `rgba(8,14,24,.62)` + 2px backdrop blur):
  `ready` = `BLASTER RANGE` (42 px, 6px letter-spacing) / `CLICK TO START` / key legend
  (`WASD move · SHIFT sprint · SPACE jump · CTRL / C crouch`,
  `SPRINT + CROUCH slide · HOLD LMB fire · HOLD RMB aim · R reload`);
  `paused` = `PAUSED` / `CLICK TO RESUME`; `dead` = `YOU'RE DOWN` / gold `FINAL SCORE n` /
  `CLICK TO RESTART`.

## 12. Game orchestration (`game.ts`)

- States: `ready | playing | dead | restarting | paused`. Score lives here.
- Camera: `fov 90, farClip 300, nearClip 0.08`, `TONEMAP_ACES`, child of the player head.
- Wiring: weapon `onShot→sfx.fire`, `onEmpty→sfx.empty`, `onReload→reloadStart/End`; player
  `onLand→sfx.land + weapon.landT = 1`, `onSlide→sfx.slide`; bots `onDamaged→hud.hitmarker +
  hit/headshot sfx`, `onKill→score += 150/100, hud.setScore, scorePop ('+150 WEAK POINT' for
  head), kill sfx`, `onBotShot→sfx.botFire`, `onPlayerHit(dmg)→(only while playing) hp −= dmg,
  shake 0.09, setHealth, damageFlash, sfx.hurt, die at ≤ 0`.
- Death: state `dead`, `sfx.death`, dead overlay with final score, `exitPointerLock`.
- Restart (`restarting`): zero score, `clearVfx`, reset player/weapon/bots, refresh HUD; the
  subsequent pointer-lock acquisition flips to `playing`.
- Click (when not playing): `initAudio()`; if dead → restart; then `requestPointerLock()`
  (catch and ignore rejection — the overlay stays up and the user clicks again).
- `pointerlockchange`: lock acquired in `ready/paused/restarting` → `playing` + hide overlay;
  lock lost while `playing` → `paused` + pause overlay.
- Per-frame (`app.on('update')`): `hud.update(dt)` always; in `paused`/`dead` consume input and
  return (bots freeze). Otherwise `bots.update(dt, state === 'playing')` — bots wander in `ready`
  too so the range looks alive behind the overlay. While `playing`: `player.update`, R-key edge
  detection, `weapon.update(dt, input.fire, input.aim, rEdge, bots.targets(), arena.colliders)`,
  then HUD ammo/reload-fraction/spread (`hud.setSpread(weapon.spread, camera.fov, weapon.aimT)`).
  Finally `input.consume()`. `app.maxDeltaTime = 0.05`.

---

## Acceptance

The game is done when every item below has been OBSERVED in the running game this session — via
browser input, screenshots, console, or runtime-query evidence — not inferred from source.

- [ ] Boots clean on the dev-server URL; no console/runtime errors across a full session.
- [ ] `ready` shows the dusk city range behind `BLASTER RANGE / CLICK TO START`, with bots
      already patrolling; clicking starts and acquires pointer lock; losing lock pauses with
      `CLICK TO RESUME`.
- [ ] WASD accelerates to 7 (10 sprint); mouse-look yaws body / pitches head (±89 clamp); Space
      jumps; Ctrl/C crouches; sprint+crouch slides at 13.5 with lowered camera, dust, and a 1 s
      cooldown; jump-cancel keeps slide momentum; roofs are reachable via the B4 west ramp and
      the B2/B5 stair decks.
- [ ] Holding LMB fires 10/s with cyan tracers from the muzzle, muzzle flash + light, camera
      recoil, bloom that widens the crosshair gap and recovers between bursts; R reloads in 1.3 s
      with the bar, arm gesture, and dip/roll; empty mag auto-reloads after one click.
- [ ] Holding RMB lerps FOV 90→62, halves sensitivity, tightens spread ×0.3, fades the crosshair
      out, and centers the rifle so the optic's glowing red dot sits on the aim point; releasing
      restores hip view (canted rifle, right of center).
- [ ] Sprinting swings the rifle up into tactical carry (55° muzzle-up, gun-only rotation about
      the grip with the hand following); aiming/firing/reloading cancels it.
- [ ] First-person arms: a gray full-body mannequin holds the rifle with the right hand on the
      grip, visible forearms in hip and ADS views, no head/shoulder blobs at any pitch.
- [ ] Bots (6 orange standard UAL1 + 2 slate armored Mannequin_F) patrol waypoints facing their
      travel direction with plant-footed jog (no skating — verify stance-foot drift ≈ 0), pause
      and look around, stop and raise the pistol to engage inside 26, fire orange tracers with
      line-of-sight checks, stagger/flash on hit, take 2× headshot damage (head sphere at 1.55),
      die with a burst + death animation + fade, and respawn elsewhere after a telegraph ring.
- [ ] Kills score +100 body / +150 head with gold score pops; hitmarker pops white (gold+longer
      on headshots); armored bots soak 130 hp.
- [ ] Player takes damage with red vignette + shake; at 0 hp `YOU'RE DOWN` + final score; click
      restarts with full state reset (score, hp, ammo, bots, VFX, player pose).
- [ ] HUD matches the spec styling: glow-only crosshair (no dark ring), animated hitmarker, ammo/
      health low-state pulses, reload bar, score pop, nothing covering screen center.
- [ ] Numeric grounding: a runtime query over all visible models shows every `aabb.min.y` within
      ±0.05 of its floor (street y=0, roof decks y=2.4/3.2; decals sit at their tuned offsets).
- [ ] First frame reads as one coherent dusk low-poly city: brick/trim/metal facades per
      building, asphalt streets with decals and sidewalk curbs, perimeter metal wall, skyline
      towers fading into warm fog, gradient star sky, warm sun + interior glow lights.

Finish with a short report: what works (with the evidence you observed), where the tuning
constants live, and any item you could not verify — stated honestly, not glossed.
