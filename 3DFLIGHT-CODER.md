# Sky Flight — Implementation Prompt

You are implementing **"Sky Flight"**, a self-contained browser-based 3D flight
simulator. The entire app lives in a single `index.html` file (inline CSS + JS,
no build step, no modules). Read this spec top to bottom, then produce the file.

## Tech stack & constraints
- **Three.js r128** loaded from CDN:
  `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
- Vanilla JS, ES6. No frameworks, no bundler, no TypeScript.
- One file: `index.html`. Inline `<style>` and `<script>`.
- Must run by simply opening the file in a browser (or via any static server).
- Handle window `resize` (update camera aspect + renderer size).
- Cap the frame delta: `dt = Math.min(clock.getDelta(), 0.05)` to avoid physics
  explosions on tab-switch.

## Page layout
- `<body>` has `overflow: hidden`. `* { margin: 0; padding: 0 }`. Canvas is `display: block`.
- **HUD** (`#hud`, top-left, white monospace, text-shadow): shows
  `ALT: <span id="alt">0</span>m` and `SPD: <span id="spd">0</span>km/h`.
- **Help bar** (`#help`, bottom-left, faint white monospace):
  `W/S throttle   A/D yaw   Up/Down pitch   Left/Right roll`.
- HUD spans are updated every frame: altitude = `Math.round(plane.position.y)`,
  speed = `Math.round(speed * 3.6)` (km/h).

## Scene
- `THREE.Scene` with sky-blue background `0x87ceeb` and matching fog
  `new THREE.Fog(0x87ceeb, 300, 1200)`.
- `PerspectiveCamera(65, innerWidth/innerHeight, 0.1, 2000)`.
- `WebGLRenderer({ antialias: true })`, sized to window, `setPixelRatio(devicePixelRatio)`.
- Lights:
  - `AmbientLight(0xffffff, 0.5)`
  - `DirectionalLight(0xfff4e0, 1.0)` at `(100, 200, 100)` (the "sun").

## Environment
- **Sea**: `PlaneGeometry(4000, 4000, 64, 64)` with `MeshPhongMaterial({ color: 0x1a6b8a, shininess: 80 })`,
  rotated `-Math.PI/2` to lie flat. Animated every frame: for each vertex,
  `z = sin(x*0.02 + t)*1.5 + cos(y*0.025 + t*0.7)*1.5`, then `needsUpdate = true`
  and `computeVertexNormals()`.
- **Mountains**: 60 cones. Each: height `20 + rand*60`, radius `15 + rand*30`,
  radial segments `5 + floor(rand*3)`. Material `MeshLambertMaterial({ color: 0x5a7a5a })`.
  Positions spread over `(rand-0.5)*1800` on x and z, y = `h/2 - 2`, random yaw.
- **Clouds**: 18 puffy clouds. Each is a `THREE.Group` of 4–7 flattened white
  spheres (radius `8 + rand*10`, `scale.y = 0.6`, random offsets within ~30×8×20).
  Shared material `MeshLambertMaterial({ color: 0xffffff, transparent: true, opacity: 0.9 })`.
  Positions spread over `(rand-0.5)*1800` on x and z, altitude `60 + rand*180` (around/above
  the plane's start).

## Plane model (a `THREE.Group`)
Build a low-poly box-based plane. Materials:
- `bodyMat = MeshPhongMaterial({ color: 0xdd3333 })` (fuselage + control surfaces)
- `wingMat = MeshPhongMaterial({ color: 0xbb2222 })` (wings/tail/fin)

Static parts (all children of the `plane` group):
- **Fuselage**: `BoxGeometry(1.2, 1.2, 6)`, bodyMat.
- **Wing** — split into 3 boxes so the rear outer sections are left open for the
  ailerons (the ailerons form the trailing edge):
  - Center: `BoxGeometry(6, 0.15, 1.8)`, position `z = 0.5` (full chord).
  - Left: `BoxGeometry(2, 0.15, 1.1)`, position `(-4, 0, 0.85)` (front portion only, z 0.3→1.4).
  - Right: `BoxGeometry(2, 0.15, 1.1)`, position `(4, 0, 0.85)` (front portion only).
- **Tail**: `BoxGeometry(3, 0.1, 1)`, position `z = -2.8`, wingMat.
- **Fin**: `BoxGeometry(0.1, 1.5, 1)`, position `(0, 0.75, -2.8)`, wingMat.

Movable control surfaces — each is a `THREE.Group` pivot holding a mesh that
extends backward (negative z) from the hinge:
- **Elevator** (`elevatorPivot` at `(0, 0, -2.8)`): mesh `BoxGeometry(2.5, 0.1, 0.8)`,
  mesh `z = -0.4`. Rotates around **X** for pitch.
- **Rudder** (`rudderPivot` at `(0, 0.75, -2.8)`): mesh `BoxGeometry(0.15, 1.2, 0.8)`,
  mesh `z = -0.4`. Rotates around **Y** for yaw.
- **Ailerons** — one per wing, sitting on the wing's rear chord:
  - `leftAileronPivot` at `(-4, 0.08, 0.3)`, mesh `BoxGeometry(2, 0.1, 0.8)`, mesh `z = -0.4`.
  - `rightAileronPivot` at `(4, 0.08, 0.3)`, same mesh.
  - Both rotate around **X** for roll (deflect in opposite directions).

Plane starts at `(0, 50, 0)`.

### Chase camera
The camera is a **child of the plane group** so it follows the aircraft:
- `camera.position.set(0, 4, -14)`
- `camera.rotation.y = Math.PI` (faces the plane from behind).

## Controls — "Mode 2" (dual-stick)
Track keys via `keydown`/`keyup` into a `keys` object keyed by `e.code`.
Rotation uses **quaternion multiplication around the plane's local axes**
(`q.setFromAxisAngle(axis, angle); plane.quaternion.multiply(q)`) so the axes
don't couple. Each control surface also deflects while its key is held and
springs back (lerp toward 0) when released.

| Input | Action | Axis / surface | Rate |
|-------|--------|----------------|------|
| `KeyW` | Throttle up | speed += 30*dt, clamp [15, 100] | — |
| `KeyS` | Throttle down | speed -= 30*dt, clamp [15, 100] | — |
| `KeyA` | Yaw left | local **Y** +, rudder deflects | 1.2 * dt |
| `KeyD` | Yaw right | local **Y** -, rudder deflects | 1.2 * dt |
| `ArrowUp` | Pitch | local **X** +, elevator deflects | 0.8 * dt |
| `ArrowDown` | Pitch | local **X** -, elevator deflects | 0.8 * dt |
| `ArrowLeft` | Roll | local **Z** -, ailerons deflect | 1.0 * dt |
| `ArrowRight` | Roll | local **Z** +, ailerons deflect | 1.0 * dt |

Control-surface deflection details:
- Deflection rate `3 * dt`, clamped to `[-0.5, 0.5]` radians.
- On release, spring back: `pivot.rotation *= (1 - 5 * dt)`.
- **Ailerons deflect opposite each other**: when rolling one way, one aileron's
  rotation.x goes positive while the other goes negative.

## Physics (simplified)
- **Forward motion**: `dir = (0,0,1).applyQuaternion(plane.quaternion)`,
  then `plane.position.addScaledVector(dir, speed * dt)`.
- **Gravity**: `plane.position.y -= 5 * dt`.
- **Lift** (proportional to speed, net positive at speed 40):
  `plane.position.y += speed * 0.15 * dt`.
- **Ground collision**: if `y < 2`, clamp `y = 2` and decay speed
  (`speed = max(speed - 40*dt, 0)`).
- **World wrap**: if `|x| > 2000` → `x *= -0.9`; same for z. Ceiling: if
  `y > 500` → `y = 500`.

## Main loop
- `THREE.Clock` for timing.
- `requestAnimationFrame` loop: compute `dt`, apply controls, integrate physics,
  animate the sea, update the HUD, `renderer.render(scene, camera)`.

## Definition of done
- Opening `index.html` renders the sea, mountains, and the red plane with a
  chase camera.
- All Mode 2 controls work as specified, with visible elevator/rudder/aileron
  deflection and spring-back.
- The plane climbs/holds around speed 40, collides with the ground, and wraps
  at the world edges.
- HUD altitude and speed update live; the page is responsive to resizing.
