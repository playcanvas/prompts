# "FREERUN 08" — Exact Third-Person Parkour Recreation Scenario

Recreate FREERUN 08 in this sandbox: a polished third-person parkour time trial across a long,
authored training course suspended over reflective blue water. A detailed runner sprints,
slides, jumps, vaults, rolls, wall-runs, and wall-jumps through 8 ordered checkpoint gates on
clean concrete-and-metal decks. The presentation is bright, hazy, and cinematic: an ocean to
the horizon, a procedural blue sky with soft clouds and sun glare, white course edges, yellow
route marks, glowing numbered gates, and a dark navy/gold record-board HUD. Everything below is
a measured, verified specification of the finished demo; reproduce it almost 1:1. Use a generic
training-simulation identity only; no protected game, brand, character, logo, or level.

## Working Rules

- You are running unattended; no one can answer questions. Decide, proceed, and note assumptions
  in your final report. Never stop at uncertainty, never end with only a plan, and never end early
  over time or context concerns — the deliverable is the working game.
- Follow the local agent guide (`AGENTS.md` or `CLAUDE.md`).
- Every requirement below is hard. Numeric values are the shipped tuning, not suggestions.
  Transcribe every normative row and value exactly into `src/constants.ts` and its consumers.
- Parallelize independent reads and searches; batch related edits instead of micro-editing.
- Build exactly this demo. Do not add alternate routes, procedural generation, combat, enemies,
  pickups, progression, inventory, multiplayer, a minimap, mobile controls, or extra modes.
- Do not create git commits; leave version control untouched.

## Assets

- The locked seeded pack is `public/assets/quaternius-urban-v1` (Quaternius, CC0). Use only files
  under `public/assets`; do not download, generate, model, replace, or import assets.
- Use these exact seeded resources:
  - `base-characters/Male_Ranger.glb` for the player — the exact hooded ranger the reference
    uses, seeded into the pack with embedded textures. Preserve scale, grounding, skeleton,
    framing, and animation exactly.
  - `animation-library/UAL1_Standard.glb` and `UAL2_Standard.glb` for animation. Their tracks
    bind to the hero's 65-joint skeleton by matching bone names — assign them directly to the
    hero's animation state graph; no manual retargeting step is needed or wanted.
  - `downtown-city/T_Concrete_{BaseColor,Normal,ORM}.png`,
    `T_MetalConcrete_{BaseColor,Normal,ORM}.png`, and
    `T_Trim_{BaseColor,Normal,ORM}.png` for all course surfaces.
- Visible engine box primitives are required for the authored course, gates, route arrows, dust,
  and bursts. Apply the seeded PBR maps to course boxes. The procedural sky cubemap, reflective
  water mesh, and screen-space speed effect are required.
- Do not use downtown building GLBs, props, weapons, or decorative scenery. The sparse training
  geometry, water, sky, markers, and animated hero are the complete visual identity.
- Hero and UAL rigs face raw +z, matching the game's +z forward: use `yawFix: 0`. Spawn the hero by render-AABB minimum so its
  feet sit exactly on the deck; the spawn helper also recenters x/z on the render-AABB center.
  Do not rescale course units.

## Build Requirements

- Keep Vite, TypeScript, and the rendering engine already wired by the template. Add no dependency.
- The template's engine is a current v2 build (local `file:` dependency). Use its current APIs —
  the per-camera frame pipeline (`CameraFrame`), the `scene.fog.*` params object, direct
  `scene.skybox` texture assignment, `ShaderChunks` — not legacy v1 patterns; read the installed
  package in `node_modules` when unsure.
- `index.html` title is exactly `Freerun 08`. Keep the full-window
  `<canvas id="application-canvas">` and module entry `/src/app.ts`.
- Preserve host, port, `strictPort`, and resolve settings. Add
  `watch: { usePolling: true, interval: 300 }` to `vite.config.ts`.
- Use this compact module split:

```
src/app.ts        engine/environment boot, loading, state, orchestration, verification API
src/constants.ts  MOVE, CAM, ANIM, TRIAL, colors, and every authored course transform
src/assets.ts     container cache, AABB-grounded spawn(), fit(), bounds()
src/input.ts      pointer-lock keyboard/mouse singleton
src/course.ts     materials, course rendering, collision, gates, arrows, particles
src/player.ts     movement, jump, slide, vault, roll, wall-run, wall-jump
src/hero.ts       animation retargeting/state, facing lean, foot-drift probe
src/camera.ts     chase camera, collision clamp, FOV, dip, wall roll
src/effects.ts    procedural audio and screen-space speed shader
src/hud.ts        DOM HUD, overlays, split/result rendering
src/style.css     full-window and navy/gold HUD presentation
```

- Keep orchestration in `app.ts`. Do not add `game.ts`, `audio.ts`, `collision.ts`, or `vfx.ts`.
- Name visible entities by authored role (`Hero`, `Launch_1_Top`, `TutorialWall_2R`,
  `Checkpoint_5`, `Arrow_Finish_8`, `GateBurst_7_12`).
- Expose `window.app` and
  `window.rush = { snapshot: () => game.snapshot(), game }` for runtime verification.

## Runtime Verification

- Test in a real browser: load the dev server, acquire pointer lock, drive input, read the
  console, query `window.rush.snapshot()`, and take screenshots. Gameplay claims must come from
  the running game, not source alone.
- Evidence must come from real synthesized keyboard/mouse input. Never mutate game state, call
  game methods, or force counters, positions, or time to fabricate evidence. A teleport-style
  debug helper may be used only to reach a later course section for spot checks and must be
  disclosed in the report; traversal, checkpoint, and result-frame claims require gates crossed
  by real movement.
- Observe one ready frame, one at-speed slide or wall-run, one fault recovery, one `F` respawn,
  one `R` restart, and one result frame.
- Snapshot must expose hero position/velocity/grounding/collider/state/wall side/clip/animation
  speed/foot drift; camera position/yaw/pitch/FOV/inside; trial
  time/checkpoint/splits/faults/retries/best/respawn; and course
  segments/obstacles/walls/checkpoints/grounding error/camera collider count.
- Clamp every update delta to 0.05 seconds. Stopwatch and movement use that same clamped delta.
- Ready reference at 1440x858: hero feet `(0,18,2)`, idle, collider 1.7; camera approximately
  `(0.39,20.1864,-2.3335)`, yaw 0, pitch -9, FOV 65; 8 segments, 8 obstacles, 5 walls,
  8 checkpoints, 40 collision boxes; no application-generated console or runtime errors
  (browser/GPU-driver noise such as WebGL fallback notices is out of scope).

---

# EXACT SPECIFICATION

## 1. Engine, boot, and environment (`app.ts`)

- Fill the window, use automatic canvas resolution, resize on window resize, and cap pixel ratio
  to `min(devicePixelRatio,1.5)`.

```
ambientLight      0.44, 0.52, 0.62
exposure          1.12
fog               linear
fogColor          0.62, 0.82, 0.95
fogStart          135
fogEnd            430
cameraClear       0.33, 0.70, 0.96
cameraFarClip     700
toneMapping       ACES2
bloomIntensity    0.025
bloomBlurLevel    16
```

- Tone mapping and bloom only exist through the engine's per-camera frame pipeline: create a
  `CameraFrame` for the camera, set `rendering.toneMapping` to ACES2 and the `bloom` values
  above, then call its `update()`. Scene-level settings will not apply them, and the §9 speed
  effect hooks this frame's compose pass.
- Directional `Sun`:

```
color             1, 0.94, 0.82
intensity         1.75
euler             48, -32, 0
castShadows       true
shadowDistance    85
shadowResolution  2048
shadowBias        0.18
normalOffsetBias  0.04
```

- Generate a 256x256 six-face RGB sky cubemap texture — sRGB, no mipmaps — and assign it
  directly as the scene skybox. Face basis/u/v vectors:

```
+X  1,0,0    0,0,-1    0,-1,0
-X -1,0,0    0,0, 1    0,-1,0
+Y  0,1,0    1,0, 0    0, 0,1
-Y  0,-1,0   1,0, 0    0, 0,-1
+Z  0,0,1    1,0, 0    0,-1,0
-Z  0,0,-1  -1,0, 0    0,-1,0
```

  Normalize each pixel direction `(x,y,z)` and calculate:

```
high   = max(0,y)^0.55
low    = max(0,-y)
haze   = exp(-abs(y)*9)
lon    = atan2(z,x)
cloud  = max(0, sin(lon*5+y*19)*0.55 + sin(lon*11-y*31)*0.30
                + sin(lon*23+y*47)*0.15 - 0.15) * exp(-((y-0.20)/0.18)^2)
wisps  = max(0, sin(lon*3-y*12) + sin(lon*17+y*25)*0.35 - 0.60)
         * exp(-((y-0.55)/0.22)^2)
dot    = max(0,x*0.55+y*0.18+z*0.815)
glow   = dot^24
sun    = dot^900
R = clamp(72  - high*32 - low*24 + haze*115 + cloud*150 + wisps*80 + glow*95  + sun*210)
G = clamp(154 - high*52 - low*36 + haze*75  + cloud*105 + wisps*65 + glow*105 + sun*200)
B = clamp(232 - high*28 - low*45 + haze*18  + cloud*40  + wisps*30 + glow*70  + sun*150)
```

- Add circular reflective `Water` at `(8,-5,190)`: a mesh from `CircleGeometry` radius 600,
  96 sectors, 80 rings, ring exponent 2.4. Do not write a water shader — the engine package
  bundles a planar-reflection water script (`scripts/esm/water.mjs`, export `Water`); attach it
  with a script component, passing `cameraEntity` (the chase camera), `lightEntity` (the sun),
  and the values below. Create a new `Water` layer inserted immediately after World's
  transparent index, put the mesh on it, and add the layer id to the camera's layer list or the
  water will not render:

```
waves true                 reflectionSource planar
refraction false           depthEffects false          foam false
textureScale 0.5           reflectionStrength 0.72      opacity 0.78
shallowColor 0.18,0.42,0.30
deepColor 0.025,0.14,0.20
waveAmplitude 0.08         waveLength 18               waveSpeed 0.65
waveSteepness 0.28         swellAmplitude 0.14         swellLength 46
swellSpeed 0.45            swellDirection 35
```

- Inject a centered dark loading tag reading `ASSEMBLING FREERUN COURSE  /  08`. Load three
  texture triplets and three GLB containers in parallel where independent, build the course,
  spawn the hero, then remove the tag. Texture anisotropy is 8; base color is sRGB, normal/ORM
  are not.

## 2. Constants (`constants.ts`) — normative tuning

```
ASSET_ROOT  /assets/quaternius-urban-v1

ASSET_TUNING
hero   base-characters/Male_Ranger.glb              scale 1  yOffset 0  yawFix 0
anim1  animation-library/UAL1_Standard.glb          scale 1  yOffset 0  yawFix 0
anim2  animation-library/UAL2_Standard.glb          scale 1  yOffset 0  yawFix 0

MOVE
baseSpeed 7             sprintSpeed 8.5       crouchSpeed 3.5
accel 60                airAccel 15            friction 10
jumpVel 6.5             gravity 20             coyoteTime 0.10
jumpBuffer 0.12         slideBoost 13.5        slideTime 0.75
slideCooldown 0.65      slideFriction 2.5      slideMinSpeed 4
vaultReach 0.85         vaultMinHeight 0.45    vaultMaxHeight 1.10
vaultTime 0.38          wallReach 0.22         wallSpeed 9.5
wallTime 3              wallGravity 0.2        wallJumpOut 7.5
wallJumpUp 6.5          rollFallSpeed 10       rollMinSpeed 4
rollTime 0.55           turnLerp 12            radius 0.35
height 1.7              slideHeight 0.9

CAM
dist 4.5                minDist 0.8            pivotHeight 1.5
shoulder 0.4            slideDrop 0.85         wallRoll 8
fov 65                  sprintFov 74           fovLerp 8
followLerp 12           sens 0.11              pitchMin -55
pitchMax 70             collideRadius 0.25

ANIM
idleBelow 0.5           walkBelow 3.5          sprintAbove 8
crossfade 0.15          jogSpeedAt1x 4.3       walkSpeedAt1x 1
sprintSpeedAt1x 7.2     animSpeedMin 0.6       animSpeedMax 2.5

TRIAL
checkpointCount 8       fallY 0.5              respawnTime 0.55
splitPopTime 2.5

WORLD
gold '#f6df24'          white '#f4f7ff'        deckDepth 0.75
```

## 3. Authored course data (`constants.ts`)

The route runs mostly along +z. Store every row as data. Positions are `x,y,z`; deck sizes are
`width,depth`; wall sizes are `width,height,depth`.

### Decks — exactly 19

```
name                 segment   position         size
Launch_1             1          0,18,8           10,20
JumpPad_1A           1          0,18,27           8,14
JumpPad_1B           1          0,18,43           8,14
WallLanding_2        2          0,18,81          10,14
FlowPad_3A           3          4,19,96           8,10
FlowPad_3B           3          9,20,109          8,10
FlowDeck_3           3         12,20,125         10,14
Precision_4A         4          6,21,141          8,10
Precision_4B         4          0,22,154          8,10
TransferLaunch_4     4          0,22,168         10,14
TransferLanding_5    5          0,22,210         10,14
Rise_6A              6          0,23,225          8,10
Rise_6B              6          8,24,237          8,10
Rise_6C              6         16,25,249          8,10
DropSlide_6          6         16,20,266         10,20
EnduranceLanding_7   7         16,20,303         10,14
Gauntlet_8           8         16,20,321         10,18
FinalLanding_8       8         14,20,357         10,14
Finish_8             8        7.5,20,372         15,14
```

Player start is `(0,18,2)` facing +z.

### Wall-run panels — exactly 5

```
name                segment   position          size
TutorialWall_2R     2          2.7,21,62         0.4,6,24
TransferWall_5R     5          2.7,25,183.5      0.4,6,17
TransferWall_5L     5         -2.7,25,194        0.4,6,18
EnduranceWall_7R    7         18.7,23,286        0.4,6,20
FinalWall_8R        8         18.7,23,340        0.4,6,20
```

### Obstacles — exactly 8

Positions are `x,z`; sizes `width,height,depth`:

```
name          type         segment   position   size
SlideGate_1   slide gate   1          0,10       9.8,0.65,0.55
VaultBox_1    vault box    1          0,43       7.8,0.75,1.4
SlideGate_3   slide gate   3          4,96       7.8,0.65,0.55
VaultBox_3    vault box    3          9,109      7.8,0.8,1.4
VaultBox_6    vault box    6          8,237      7.8,0.8,1.4
SlideGate_6   slide gate   6         16,267      9.8,0.65,0.55
SlideGate_8   slide gate   8         16,317      9.8,0.65,0.55
VaultBox_8    vault box    8         16,326      9.8,0.8,1.4
```

### Ordered checkpoints — exactly 8, yaw 0

```
name          gate position   reset position
Checkpoint_1   0,18,47         0,18,43
Checkpoint_2   0,18,84         0,18,80
Checkpoint_3  12,20,129       12,20,125
Checkpoint_4   0,22,171        0,22,167
Checkpoint_5   0,22,213        0,22,209
Checkpoint_6  16,20,272       16,20,268
Checkpoint_7  16,20,306       16,20,302
Checkpoint_8  7.5,20,374      7.5,20,370
```

### Route arrows — exactly 8; positions x,z

```
Arrow_Start          0,5       yaw 0
Arrow_Wall_2       1.4,46      yaw 0
Arrow_Flow_3         0,84      yaw 0
Arrow_Transfer_5   1.4,170     yaw 0
Arrow_Rise_6         0,214     yaw 0
Arrow_Endurance_7 17.4,272     yaw 0
Arrow_Gauntlet_8  17.4,326     yaw 0
Arrow_Finish_8      12,360     yaw -25
```

## 4. Course rendering and collision (`course.ts`)

- PBR materials assign base color, normal, and ORM. Read ORM as AO=r, gloss=g, metalness=b;
  invert gloss, enable metalness, set bumpiness 0.65, and tile all maps by supplied u,v.

```
white  #f4f7ff  opacity 0.92  emissive 2.4
faint  #71879d  opacity 0.20  emissive 0.45
gold   #f6df24  opacity 0.96  emissive 3.2
```

- For each deck `(x,y,z,w,d)`:
  - shell center `(x,y-0.41,z)`, size `(w,0.68,d)`, MetalConcrete tint `#73808f`,
    tiling `max(1,w/4),max(1,d/4)`
  - top center `(x,y-0.035,z)`, size `(w-0.16,0.07,d-0.16)`, Concrete tint
    `#9aa7b3`, tiling `max(1,w/3),max(1,d/3)`
  - four white strips at y+0.015: left/right `0.09,0.06,d` at x±w/2; front/back
    `w,0.06,0.09` at z±d/2
  - deck collision min `(x-w/2,y-0.75,z-d/2)`, max `(x+w/2,y,z+d/2)`
- For each wall: main MetalConcrete panel tint `#6f7d90`, tiling
  `max(1,depth/3),max(1,height/2)`; white top/bottom caps
  `width+0.12,0.12,depth+0.12`; white start/gold end caps
  `width+0.12,height-0.12,0.12`; gold guide at `y-height*0.18` sized
  `width+0.15,0.10,depth*0.72`; one solid collision flagged `wall`.
- Slide gate: bar center `(x,floor+1.08+h/2,z)`, render width w-0.3 but collide width w.
  Posts center at `x±(w/2-0.15),floor+1.3,z`, render
  `0.3,2.6,d+0.08`, collide `0.3,2.6,d`. Use Trim tint `#8895a4`, tiling `max(1,w/2),1`.
  Gold strip at
  `floor+1.06,z-d/2-0.02` sized `w*0.82,0.07,0.04`.
- Vault box: center `(x,floor+h/2,z)`, Concrete tint `#87929d`, tiling
  `max(1,w/2),max(1,d/2)`, size w,h,d, gold top
  guide at floor+h+0.025 sized `w*0.72,0.05,d*0.72`; collision flagged vault and top.
- Arrow roots sit at floor+0.04. Two gold boxes:
  `(-0.34,0.008,0)` yaw 38 and `(0.34,0,0)` yaw -38, each `0.13,0.04,1.15`.
- Gate is 8.08 wide, 4.2 high: vertical lights `0.08,4.2,0.08` at x±4 and top light
  `8.08,0.08,0.08`. Draw numbers from seven-segment masks:

```
0 1111110   1 0110000   2 1101101   3 1111001   4 0110011
5 1011011   6 1011111   7 1110000   8 1111111
```

  Segment centers in gate-local space, mask order: `(0,3.88) (-0.22,3.68) (-0.22,3.28) (0,3.08)
  (0.22,3.28) (0.22,3.68) (0,3.48)`. Horizontal segments (mask indices 0, 3, 6) are
  `0.38,0.07,0.08` at z 0.02; verticals are `0.07,0.38,0.08` at z 0.
- Only next gate is bright: gates 1-7 white, gate 8 gold, later gates faint. Pulse enabled gates
  by scaling the whole gate root to `1+sin(time*3+index)*0.035`. Crossing disables the gate,
  emits 16 cubes (0.09 each; gold for gate 8, else white) on a radius-2 ring at gate height
  +1.8 with velocity `(sin a*4, (i%3)+1, cos a*2)` for ring angle a, destroyed after
  0.65 seconds, then activates the next.
- Dust uses 5 cubes normally, 9 for hard land/roll, 3 gold for wall actions: `0.08,0.04,0.08`
  boxes scattered ±0.35 in x/z at +0.08 height, velocity ±0.9 in x/z and 0..0.8 up.
- `ground(x,z,feet)` returns highest deck/flagged top below feet+0.35 containing x/z, else
  -Infinity. Callers offset `feet` upward: player grounding passes feet `y+0.75` (so tops up to
  1.1 above the feet are candidates for §6's 0.55-below..1.05-above snap), vault landing passes
  `box top+0.5`, course builders pass a large value. Player collision is AABB overlap with
  radius 0.35 and active height.
- Exactly 40 collision boxes: 19 decks + 5 walls + 12 slide pieces + 4 vault boxes.
- Camera collision is segment versus every AABB expanded by 0.25, returning nearest fraction.
- Vault requires a flagged box within reach and in front, top 0.45-1.1 above feet, headroom,
  and no standing collision. The landing point is exact: march the movement direction to its
  exit of the box footprint, continue 0.55 further, and ground() there with feet `box top+0.5`;
  the result must be finite and free of standing collision.
- Wall query considers flagged panels overlapping player height with face gap -0.12..0.22 and
  tangent input magnitude at least 0.3. Reject a candidate only when the wish points away from
  the wall by more than 0.55 (wish · outward normal > 0.55); wall-run entry additionally
  requires wish · normal < 0.

## 5. Input (`input.ts`)

- Click starts/resumes and requests pointer lock. Mouse orbits only while locked.
- WASD moves relative to camera; Shift sprints; Space jumps/vaults/wall-jumps; Ctrl/C
  crouches/slides; R restarts; F faults/respawns.
- Track held and edge-triggered keys separately. Clear keys/presses/mouse delta on blur or
  pointer-lock loss. Prevent Space/Ctrl/C/R/F defaults and context menu only while running.

## 6. Player movement and traversal (`player.ts`)

- `Player` stores feet position. Spawn `(0,18,2)`; visible hero faces the facing yaw directly.
- Forward is `(sin(yaw),0,cos(yaw))`; right `(-cos(yaw),0,sin(yaw))`. Normalize diagonals.
- Approach desired velocity by accel 60 on ground, 15 in air; no-input ground friction 10.
  Crouch/run/sprint targets are 3.5/7/8.5. Facing follows velocity by shortest arc at rate 12.
- Jump buffer 0.12, coyote 0.10, y velocity 6.5, gravity 20; preserve horizontal momentum.
- Resolve x then z; collision restores that axis, zeros that component, emits `bonk`.
- Leave ground if missing or >0.55 below. Snap to valid ground from 0.55 below to 1.05 above.

### Slide

- New Ctrl/C press starts only grounded, Shift held, speed >=4, cooldown zero. Lower collider to
  0.9 for 0.75 seconds, preserve direction, raise speed to at least 13.5, apply friction 2.5.
- End only with standing headroom, restore 1.7, cooldown 0.65. Holding crouch remains 0.9/3.5.
  Space during slide jumps without reducing horizontal speed.

### Vault

- Sprinting into a valid vault box at speed >=`baseSpeed*0.72` auto-vaults. Buffered Space may
  also vault while grounded.
- Over 0.38 seconds use smoothstep `t*t*(3-2*t)` from stored start to supported landing plus
  `sin(pi*t)*max(0.35,top-startY+0.2)` vertical arc. Finish grounded and preserve flow.

### Wall-run / wall-jump

- Enter only airborne, Shift held, speed >=`baseSpeed*0.72`, wishing into an authored wall not
  locked from the previous jump.
- Align at radius+0.035, move along wall tangent at `max(9.5,current speed)`, gravity 0.2, maximum
  3 seconds. Keep hero yaw aligned with travel along the wall, never toward its normal. Bank the
  hero 12 degrees away from the wall, keep the camera roll specified in §8, and emit gold dust
  every 0.10 seconds.
- Space jumps with tangent speed + 7.5 outward + 6.5 upward. Lock same wall and reduce air
  steering to 12% for 0.45 seconds; clear on landing.
- Releasing sprint/input, losing wall, or timeout returns to fall; touching ground returns run.

### Landing and faults

- Track peak fall speed. At >=10: speed >=4 rolls for 0.55 preserving momentum; slower hard
  landing lasts 0.32 with dip. Soft land never stuns or clears horizontal speed.
- Falling below y=0.5 starts 0.55-second recovery, increments faults, clears velocity, flashes
  red, keeps time running, then restores last checkpoint reset or start.

## 7. Hero and animation (`hero.ts`)

- Spawn `Hero` AABB-grounded, parent under `Player` at local origin, cast shadows. If `root` is
  not under `Armature`, add that wrapper before animation.
- Retarget exact clips:

```
UAL1: Idle_Loop, Walk_Loop, Jog_Fwd_Loop, Sprint_Loop,
      Crouch_Idle_Loop, Crouch_Fwd_Loop, Jump_Start, Jump_Loop, Jump_Land, Roll
UAL2: NinjaJump_Start, NinjaJump_Idle_Loop, NinjaJump_Land,
      Slide_Start, Slide_Loop, Slide_Exit, ClimbUp_1m
```

```
idle grounded <0.5           Idle_Loop
locomotion                   blend Walk@0.5, Jog@3.5, Sprint@8
crouch stopped/moving        Crouch_Idle_Loop / Crouch_Fwd_Loop
jump first 0.16 speed <8     Jump_Start at 4x
jump loop speed <8           Jump_Loop
jump first 0.16 speed >=8    NinjaJump_Start at 4x
jump loop speed >=8          NinjaJump_Idle_Loop
slide first 0.18/remainder   Slide_Start / Slide_Loop
slide exit                   Slide_Exit for 0.20
vault                        ClimbUp_1m at 0.666/0.38
roll                         Roll at 1.466/0.55
hard slow land               Jump_Land at 2.8x
wall-run                     locomotion at blend point 8, rate speed/7.2
airborne without jump        Jump_Loop / NinjaJump_Idle_Loop by the same speed >=8 split
```

- Crossfade 0.15; locomotion-to-idle fade is `speed/friction`. Clamp animation rate 0.6..2.5.
  Locomotion rate divides actual speed by interpolated natural stride: 1..4.3 below speed 3.5,
  then 4.3..7.2 through speed 8.
- `wallSide` is +1 when the wall is on the runner's right and -1 when it is on the left. Smooth
  the hero's local-z lean to `wallSide*12` at rate 9, banking the torso away from the wall.
- Sample `foot_l`/`foot_r` each frame and expose stance-foot drift along travel only when
  grounded and the low foot is within 0.18 of player feet.

## 8. Chase camera (`camera.ts`)

- Initial yaw 0, pitch -9. Mouse sensitivity 0.11 degrees/pixel; pitch clamp -55..70.
- Smooth pivot toward feet +(0,1.5,0) at rate 12. Lower 0.85 during slide. Dips: hard land 0.42,
  roll 0.28, wall-jump 0.18; recover at rate 8.
- Desired camera is 4.5 behind forward and 0.4 right; remove shoulder during wall-run.
- Use collision hit fraction-0.025, minimum 0.8. Pull in immediately, recover out at rate 7.
  Then iterate at most 20 times while inside expanded boxes, subtracting 0.15 distance each.
- Look at pivot; smooth roll to `wallSide*8` at rate 9.
- Lerp FOV 65..74 from `clamp((speed-7)/(8.5-7),0,1)` at rate 8. `camera.inside` stays false.
- Reset recenters the pivot on the player, restores yaw/pitch/distance/roll, then runs one
  clamped 0.05-second update to settle — this is what produces the ready camera at approximately
  `(0.39,20.1864,-2.3335)`.

## 9. Effects and audio (`effects.ts`)

- The screen-space speed effect overrides the CameraFrame compose pass: get the shader chunk
  maps via `ShaderChunks` for both GLSL and WGSL, set `composeDeclarationsPS` to the effect
  uniforms/functions, and set `composeMainEndPS` to `result = rushResult(result, uv);`.
- Set uniforms every frame: `rushCenter` (hero position +0.9 up, projected to screen, v
  flipped), `rushSize` (viewport pixel size), `rushTime` (seconds), `rushAmount` (1 only during
  slide/wall-run while running and not respawning, else 0).
- rushResult, gold radial edge streaks: `p = uv - center` with p.x scaled by aspect;
  `r = length(p)`; `spoke = (atan2(p.y,p.x) + PI) * 12/(2*PI)`;
  `ray = 1 - smoothstep(0.035, 0.075, abs(fract(spoke) - 0.5))`;
  `pulse = fract(r*1.05 - time + hash(floor(spoke)))` with
  `hash(n) = fract(sin(n*12.9898)*43758.5453)`;
  `streak = smoothstep(0, 0.16, pulse) * (1 - smoothstep(0.72, 0.96, pulse))`;
  `edge = max(abs(uv.x-0.5), abs(uv.y-0.5)) * 2`; add to the composed color
  `(1,0.78,0.18) * ray * streak * smoothstep(0.72,1,edge) * smoothstep(0.35,0.7,r) * amount * 0.08`.
- Initialize WebAudio on first click. Master gain 0.13. Continuous 48Hz sawtooth wind gain
  targets `max(0,speed-7)*0.012` with 0.08 smoothing and mutes paused/respawning.
- One-shots fall exponentially to `max(30,Hz*0.55)` and fade to 0.001. Footstep gain 0.12;
  all others 0.28:

```
footstep 150 0.035 square    jump 430 0.11 sine          land 95 0.12 triangle
landHard 62 0.24 square     roll 78 0.20 sawtooth       slide 120 0.34 sawtooth
vault 520 0.16 sine         checkpoint 760 0.18 triangle
wallrun 260 0.16 sawtooth   walljump 560 0.14 triangle  fault 54 0.32 sawtooth
restart 240 0.16 square     finish 620 0.50 triangle     record 920 0.65 sine
bonk 82 0.08 square
```

- Run/crouch footsteps interval is `0.52/max(0.7,speed/5)` above speed 1. Slide dust every 0.12.

## 10. HUD (`hud.ts`, `style.css`)

- DOM/CSS over canvas. Font `"Arial Narrow","Roboto Condensed",system-ui,sans-serif`. Base
  `#f4f7ff`, HUD gold `#f5bf42`, navy `#101a26`. Page background `#58aceb` with a fixed
  full-screen overlay: subtle radial white sun glow at 72% 18% plus a faint bottom `#bdeaff18`
  gradient. HUD text carries a `0 1px 2px #07111c` shadow; plates cast soft drop shadows.
  "Angular plate" means a linear-gradient background with transparent corner cuts
  (e.g. `linear-gradient(112deg, transparent 18px, #101a26e8 18px calc(100% - 18px),
  transparent calc(100% - 18px))`).
- The loading tag is gold 12px 900-weight text, letter-spacing .14em, on `#101a26dc`.
- Desktop layout:
  - top-left 28px: 184px record panel; gold `BEST TIME` header on dark text, clipped by
    `polygon(0 0, 96% 0, 100% 100%, 0 100%)`; then a 20px white best row and 10px
    `FAULTS`/`RETRIES` rows, each with a 34px-wide gold digit cell on `#070e17d9`
  - top-center y25: 300px angular plate (18px cuts), 8px `◢ FREERUN TRIAL ◣` label over a 35px
    italic gold timer; dashed gold wings either side (48x3px, repeating 6px on / 4px off,
    40px outside the plate at y30)
  - split pop centered y96: 11px 900-weight text on a 12px-cut angular `#121c28d9` plate
  - bottom-left 28px: `R RESTART COURSE` / `F RESPAWN AT CHECKPOINT`, 9px rows with 23x19px
    gold key caps
  - bottom-right 30/27px: 26px italic gold three-digit speed + 8px `KM/H` on an angular plate
- Floor milliseconds and format `M:SS.mmm`. Initial best `--:--.---`. Speed is
  `round(worldSpeed*3.6)` padded to 3 digits.
- Ready overlay centered near 53%, max width 560px, angular translucent navy (`#101a26e6`,
  20px cuts): gold 10px kicker, italic `FREERUN` at `clamp(38px, 6vw, 66px)`, gold 13px
  `CLICK TO START`, then 9px control hints at 1.8 line-height. Exact text:

```
SIMULATION COURSE 08
FREERUN
CLICK TO START
WASD MOVE · MOUSE ORBIT · SHIFT SPRINT / WALL RUN
SPACE JUMP / VAULT / WALL JUMP · CTRL / C SLIDE
```

- Pause: `PAUSED / CLICK TO RESUME`.
- Split: `CHECKPOINT n/8  M:SS.mmm` plus signed difference against best formatted `(±S.mmm)`
  with three decimals; ahead gold, else white; visible 2.5 seconds.
- Result: optional 210px gold `COURSE RECORD` banner, `TRAINING COMPLETE`, `COURSE COMPLETE`,
  final time at 43px italic gold, 8 splits as 9px two-column rows on `#08111bcc` (per-row diff
  vs best formatted `±SS.mmm`, ahead gold), faults/retries, `CLICK TO RUN AGAIN`.
- Edge flash lasts 360ms as inset box-shadow glows: fault red `#e63b31cc`, checkpoints 1-7
  white `#e9f8ffbb`, checkpoint 8 gold `#f5bf42bb`.

## 11. Trial orchestration (`app.ts`)

- States: `ready`, `running`, `paused`, `finished`, `restarting`.
- Ready idles/pulses with time 0. Click ready/paused initializes audio, hides overlay, runs, and
  locks pointer. Pointer-lock loss pauses time/movement/effects and shows pause.
- Stopwatch starts only on first WASD hold or Space edge. It counts clamped delta while running,
  including fault recovery, and stops at finish.
- R increments retries and resets time/checkpoint/splits/current faults/start flag/respawn,
  gates/bursts/player/camera/HUD, then continues. F uses the fault path.
- Only current gate can cross. In gate-local space accept half-width 4.2, vertical difference 5,
  half-depth 1.35. Store absolute split/difference, collapse/pop/flash/play, advance.
- Checkpoint 8 finishes: zero player velocity/speed, idle animation, stop time, show result, play
  finish/record, store faster session best/splits, exit pointer lock.
- Clicking finished increments retries, fully resets, locks pointer, immediately starts again.
  Best and total retries persist only in the page session.

## Acceptance

Every item must be observed in the running browser:

- [ ] Clean boot, title `Freerun 08`, no application-generated console/runtime errors, no
      dependency added.
- [ ] Ready view matches: hooded full-body ranger lower center on glossy gray deck; reflective blue
      water to horizon; hazy blue sky and upper-right sun; sparse long course; navy/gold HUD and
      `SIMULATION COURSE 08 / FREERUN / CLICK TO START` overlay.
- [ ] Ready snapshot reports hero `(0,18,2)`, camera approximately
      `(0.39,20.1864,-2.3335)`, FOV 65, 8 segments, 8 obstacles, 5 walls, 8 checkpoints,
      40 colliders, zero grounding error.
- [ ] All 19 deck, 5 wall, 8 obstacle, 8 checkpoint, and 8 arrow rows match exactly by runtime
      names/transforms.
- [ ] Decks are thick sealed metal shells with inset glossy concrete and four white edge strips.
      No wooden rooftop, city skyline, building kit, cloud deck, or asset-gallery treatment.
- [ ] Water, procedural cubemap, haze, sunlight, ACES2, bloom, fog, and shadows match the bright
      reflective reference.
- [ ] Pointer lock/pause, camera-relative WASD/orbit, sprint FOV, crouch, jump, slide, vault,
      roll, wall-run, and wall-jump work with exact tuning.
- [ ] First obstacle forces a slide under `SlideGate_1`; the following gap forces a
      momentum-preserving jump; a clean traversal reaches checkpoint 1 at z=47.
- [ ] All wall panels support gold-dust runs at speed 9.5 and Space wall-jumps; hero yaw stays
      tangent to travel and the torso visibly banks 12 degrees away from the wall; camera rolls
      to 8 degrees.
- [ ] Exact animation clips crossfade without T-pose/freeze; locomotion rate follows stride data.
- [ ] Camera stays outside all 40 boxes, pulls in/recover smoothly, drops/dips/rolls as specified.
- [ ] Checkpoints trigger only in order, pulse, show numbers, burst, pop splits, and restore exact
      reset positions.
- [ ] Fall and F increment faults, red-flash, clear velocity for 0.55 seconds, keep time running,
      and restore. R resets time/faults and increments retries.
- [ ] HUD copy, layout, colors, formatting, speed, overlays, flashes, best/fault/retry values
      match at 1440x858 and remain readable at speed.
- [ ] Checkpoint 8 shows first-run `COURSE RECORD`, result copy, final time, 8 splits,
      faults/retries, and rerun behavior; best updates only when faster.
- [ ] Ready and at-speed slide/wall-run screenshots both read as the same FREERUN 08 demo, not a
      generic rooftop parkour game.

Finish with a short report: observed traversal, screenshot summary, runtime counts, best completed
time/faults if achieved, and anything not verified.
