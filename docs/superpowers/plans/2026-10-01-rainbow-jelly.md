# Rainbow Jelly Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained `index.html` — raw WebGPU + WGSL, vanilla JS, no libraries — delivering an interactive soft-body rainbow jelly bar: XPBD tet-mesh physics, hand/knife tools, refractive layered rendering, synthesized audio, editorial UI.

**Architecture:** One HTML file. Inline `<script>` (classic, not module, so it runs via `file://` or any static server) holds: (1) a procedural tet-mesh builder, (2) an XPBD solver (CPU typed-arrays first; WebGPU compute upgrade only if Phase 1 CPU version can't hold budget), (3) a WebGPU render pipeline fed by WGSL template strings for floor/jelly/knife/bubbles, (4) input/tool handling (hand, knife), (5) Web Audio synthesis, (6) DOM UI wired to live solver/renderer state. Built and verified in the spec's own 6 phases; each phase ends with a manual browser check.

**Tech Stack:** WebGPU, WGSL, vanilla JS (ES2020+, classic script), Web Audio API, HTML/CSS (Google Fonts via `<link>`, which is not a script/library dependency — spec's "no libraries" targets JS/physics/rendering libs, not typography).

**Spec:** `docs/superpowers/specs/2026-10-01-rainbow-jelly-design.md`

## Global Constraints

- Single self-contained `index.html`, raw WebGPU + WGSL, vanilla JS, no libraries, no external assets (fonts via Google Fonts `<link>` are the one allowed external fetch).
- Jelly bar: ~4 × 1.4 × 1.6 units, rounded edges/corners radius ~0.15, 6 horizontal layers bottom→top: violet, blue, green, yellow, orange, pink-red, with thin milky "panna cotta" band between every second layer.
- Embossed handle `@_thecodingdiva_` on top face, raised ~0.03, surface/normal effect only — not physics geometry.
- Tet mesh target: ~600 particles / ~2200 tetrahedra, generated from 2D side-profile outline extruded in depth.
- Fixed 60 Hz physics step, 10 substeps/frame, accumulator-based (framerate-independent).
- Per-tet: co-rotational elastic constraint (finite compliance, must recover inverted tets) + volume-preservation constraint.
- Per-layer stiffness multiplier (top softer, bottom firmest).
- Damping only on relative velocity along mesh edges; free fall/tumbling undamped. Gravity on.
- Ground plane contact: project out of floor + Coulomb friction. Piece-piece collision: no interpenetration, cheap approach.
- Global sliders: Firmness (0-1) and Internal Damping (0-1) scale compliances live.
- Must hold 60 fps with 1-8 pieces on a laptop.
- Knife: stroke defines vertical blade plane in world space at draw time; per crossed piece, fit rotation, map plane to rest space, split rest-space 2D outline (keep rounded corners), rebuild both halves as fresh tet meshes, inherit position/velocity, animate 3D cleaver (press groove → break through → swap pieces → wedge apart).
- Minimum piece size enforced for repeated cuts.
- Cut faces must show rainbow layers in cross-section.
- Surface mesh (jelly skin, embossed handle, bubbles) bound to tets via barycentric weights; layer color = function of rest-space height.
- Render pass order: floor + bubbles → jelly back-face thickness pass → jelly front faces (refraction by thickness, Beer-Lambert absorption per layer, milky scattering in panna-cotta bands, backlit edge glow, Fresnel + GGX specular, procedural studio env). Knife blade drawn behind jelly surface (refracted when buried).
- Floor: warm off-white studio paper, soft key-light shadow tinted by jelly colors, contact occlusion from top-down height map.
- MSAA or FXAA, devicePixelRatio-aware, resize-safe.
- Audio: synthesized only (Web Audio, no files) — grab/release squish scaled by pull distance, knife break-through thwack+slice, landing/collision bloop scaled by impact speed (rate-limited), mute toggle, starts only after first user interaction.
- UI: warm paper background, Instrument Serif/Cormorant italic display title, grotesk labels, mono numbers; top-left label + title + subtitle; top-right "● WEBGPU · LIVE" pill; right control panel (tool toggle, variety swatches, firmness/damping sliders, nudge/reset buttons, ¼-speed + show-mesh checkboxes, pause, sound toggle); bottom-left tool instruction + live stats (mass, volume %, kinetic, pieces); collapsible "INSIDE THE EXPERIMENT +" explainer panel. Responsive, mobile bottom-sheet panel, per-tool cursor. WebGPU-unavailable fallback message.
- Six phases, each verified in-browser before proceeding: (1) WebGPU+mesh+solver+floor+wireframe, (2) hand tool, (3) surface binding+shading+shadows, (4) knife, (5) UI/stats/varieties/sound/explainer, (6) polish+perf (8+ pieces @ 60fps, ¼ speed, no console errors).

## Review Focus

- **Degenerate/zero-reach drag:** hand tool grabbing at the very tip/corner where few particles are in reach — spring must still attach to at least one particle, not silently no-op.
- **Knife stroke that misses the jelly or is too short:** release with a stroke that never crosses any piece, or a 1-pixel click — must not crash or produce a zero-area split; should just do nothing.
- **Cutting a piece below minimum size:** repeated cuts on an already-small piece must respect the minimum and refuse/ignore rather than producing degenerate (near-zero-volume or inverted) tets.
- **Tab backgrounded / huge frame delta:** switching tabs for a while then returning must not feed a giant `dt` into the fixed-step accumulator and explode the sim (accumulator must clamp).
- **WebGPU unavailable:** `navigator.gpu` missing or adapter/device request failing must show the fallback message, not a blank page or uncaught exception.

---

## Phase 1 — WebGPU setup, procedural tet mesh, XPBD solver, floor, wireframe

### Task 1.1: Project skeleton, WebGPU device init, resize-safe canvas, fallback message

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: global `app` object holding `{ canvas, context, device, format, presentationSize }`; function `async function initWebGPU()` that populates `app` or throws; function `showFallback(message)` that replaces canvas with a fallback DOM message; function `resizeCanvas()` that updates canvas backing size from `devicePixelRatio` and reconfigures the context.

- [ ] **Step 1: Write `index.html` shell with canvas, fallback container, and Google Fonts link**

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Rainbow Jelly</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Instrument+Serif:ital@1&family=Cormorant:ital@1&family=Space+Grotesk:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
  :root { color-scheme: light; }
  html, body { margin: 0; padding: 0; width: 100%; height: 100%; background: #f3efe9; overflow: hidden; }
  #app { position: fixed; inset: 0; }
  canvas#gpu-canvas { position: absolute; inset: 0; width: 100%; height: 100%; display: block; touch-action: none; }
  #fallback { display: none; position: fixed; inset: 0; align-items: center; justify-content: center; text-align: center; font-family: 'Space Grotesk', sans-serif; background: #f3efe9; color: #2a2a2a; padding: 2rem; }
  #fallback.visible { display: flex; }
  #fallback h1 { font-family: 'Instrument Serif', serif; font-style: italic; font-size: 2.2rem; font-weight: 400; margin-bottom: 0.5rem; }
</style>
</head>
<body>
<div id="app">
  <canvas id="gpu-canvas"></canvas>
</div>
<div id="fallback">
  <div>
    <h1>This experiment needs WebGPU.</h1>
    <p>Your browser or device doesn't support it yet. Try the latest Chrome, Edge, or Safari Technology Preview on a desktop machine.</p>
  </div>
</div>
<script>
const app = {
  canvas: null,
  context: null,
  device: null,
  format: null,
  width: 0,
  height: 0,
};

function showFallback(message) {
  const canvas = document.getElementById('gpu-canvas');
  if (canvas) canvas.style.display = 'none';
  const fb = document.getElementById('fallback');
  if (message) fb.querySelector('p').textContent = message;
  fb.classList.add('visible');
}

async function initWebGPU() {
  if (!('gpu' in navigator)) {
    throw new Error('navigator.gpu is not available');
  }
  const adapter = await navigator.gpu.requestAdapter({ powerPreference: 'high-performance' });
  if (!adapter) throw new Error('No WebGPU adapter found');
  const device = await adapter.requestDevice();
  const canvas = document.getElementById('gpu-canvas');
  const context = canvas.getContext('webgpu');
  if (!context) throw new Error('Failed to get webgpu canvas context');
  const format = navigator.gpu.getPreferredCanvasFormat();

  app.canvas = canvas;
  app.context = context;
  app.device = device;
  app.format = format;

  device.lost.then((info) => {
    console.error('WebGPU device lost:', info.message);
  });

  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);
}

function resizeCanvas() {
  if (!app.canvas || !app.device) return;
  const dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 2));
  const rect = app.canvas.getBoundingClientRect();
  const width = Math.max(1, Math.floor(rect.width * dpr));
  const height = Math.max(1, Math.floor(rect.height * dpr));
  if (width === app.width && height === app.height) return;
  app.width = width;
  app.height = height;
  app.canvas.width = width;
  app.canvas.height = height;
  app.context.configure({
    device: app.device,
    format: app.format,
    alphaMode: 'opaque',
  });
}

async function main() {
  try {
    await initWebGPU();
  } catch (err) {
    console.error('WebGPU init failed:', err);
    showFallback();
    return;
  }
  console.log('WebGPU ready', app.device, app.format, app.width, app.height);
}

main();
</script>
</body>
</html>
```

- [ ] **Step 2: Manual browser check**

Serve the directory (`npx http-server -p 8080` or any static server) and open `http://localhost:8080/index.html`. Confirm in DevTools console: `WebGPU ready` logged with a device, a format string (e.g. `"bgra8unorm"`), and nonzero width/height. Resize the window and confirm no errors; add a temporary `console.log` in `resizeCanvas` if needed to confirm it fires, then remove it.

To force-test the fallback path, temporarily rename `navigator.gpu` check (e.g. run `Object.defineProperty(navigator, 'gpu', {value: undefined})` in the console before reload is not possible since it's read at load — instead temporarily edit `initWebGPU` to `throw new Error('test')` as the first line, reload, confirm the fallback message appears and the canvas is hidden, then revert the temporary edit.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: WebGPU device init, resize-safe canvas, fallback message"
```

### Task 1.2: Procedural tet mesh builder from 2D rounded-rect outline

**Files:**
- Modify: `index.html` (add a `<script>` block before `main()`, or extend the existing script)

**Interfaces:**
- Consumes: nothing external.
- Produces: `function buildJellyMesh(opts)` returning
  `{ positions: Float32Array /* [x,y,z]*N, rest-space */, tets: Uint32Array /* [a,b,c,d]*T */, particleCount: N, tetCount: T, layerHeightMin: number, layerHeightMax: number }`.
  `opts = { width=4, height=1.4, depth=1.6, cornerRadius=0.15, targetParticles=600, targetTets=2200 }`.
  Particles are laid out on a 3D lattice: 2D rounded-rect cross-section (in the X-Y plane, the "side profile") sampled on a grid, extruded along Z. Each lattice cell of 8 corner particles is split into 5 or 6 tets (alternating pattern to keep shared faces consistent between neighbors). Particles outside the rounded-rect outline (corner cut-off regions) are dropped and remaining ones re-indexed.

- [ ] **Step 1: Write the mesh-builder test harness (plain Node, no framework) and failing assertions**

Create a throwaway `test-mesh.mjs` in the scratchpad dir (not committed) that `eval`s the `buildJellyMesh` function extracted from `index.html` is awkward; instead, write the function to be self-testable by pasting it into a quick Node check. Since there's no module boundary, verify via a temporary inline `<script>` test at the bottom of `index.html` during development:

```js
// TEMPORARY - remove after Step 4
function assert(cond, msg) { if (!cond) throw new Error('ASSERT FAILED: ' + msg); }
const testMesh = buildJellyMesh({});
assert(testMesh.particleCount > 400 && testMesh.particleCount < 900, 'particle count in range: ' + testMesh.particleCount);
assert(testMesh.tetCount > 1500 && testMesh.tetCount < 3200, 'tet count in range: ' + testMesh.tetCount);
assert(testMesh.positions.length === testMesh.particleCount * 3, 'positions length matches particle count');
assert(testMesh.tets.length === testMesh.tetCount * 4, 'tets length matches tet count * 4');
// every tet index must reference a real particle
for (let i = 0; i < testMesh.tets.length; i++) {
  assert(testMesh.tets[i] < testMesh.particleCount, 'tet index in range at ' + i);
}
// no degenerate (zero-volume) tets
function tetVolume(positions, a, b, c, d) {
  const ax = positions[a*3], ay = positions[a*3+1], az = positions[a*3+2];
  const bx = positions[b*3]-ax, by = positions[b*3+1]-ay, bz = positions[b*3+2]-az;
  const cx = positions[c*3]-ax, cy = positions[c*3+1]-ay, cz = positions[c*3+2]-az;
  const dx = positions[d*3]-ax, dy = positions[d*3+1]-ay, dz = positions[d*3+2]-az;
  const crossX = cy*dz - cz*dy, crossY = cz*dx - cx*dz, crossZ = cx*dy - cy*dx;
  return (bx*crossX + by*crossY + bz*crossZ) / 6;
}
let minVol = Infinity;
for (let t = 0; t < testMesh.tetCount; t++) {
  const v = Math.abs(tetVolume(testMesh.positions, testMesh.tets[t*4], testMesh.tets[t*4+1], testMesh.tets[t*4+2], testMesh.tets[t*4+3]));
  if (v < minVol) minVol = v;
}
assert(minVol > 1e-6, 'no degenerate tets, min volume: ' + minVol);
console.log('MESH TESTS PASSED', testMesh.particleCount, testMesh.tetCount, 'minVol', minVol);
```

- [ ] **Step 2: Run it (currently fails — `buildJellyMesh` not defined)**

Reload the page in browser, confirm console shows `ReferenceError: buildJellyMesh is not defined` or similar.

- [ ] **Step 3: Implement `buildJellyMesh`**

```js
function roundedRectSDF(x, y, hw, hh, r) {
  const qx = Math.abs(x) - (hw - r);
  const qy = Math.abs(y) - (hh - r);
  const ax = Math.max(qx, 0), ay = Math.max(qy, 0);
  return Math.sqrt(ax * ax + ay * ay) + Math.min(Math.max(qx, qy), 0) - r;
}

function buildJellyMesh(opts = {}) {
  const width = opts.width ?? 4;
  const height = opts.height ?? 1.4;
  const depth = opts.depth ?? 1.6;
  const cornerRadius = opts.cornerRadius ?? 0.15;
  const targetParticles = opts.targetParticles ?? 600;

  const hw = width / 2;
  const hh = height / 2;
  const hd = depth / 2;

  // Solve grid resolution so nx*ny*nz (before outline culling) lands near targetParticles / (fillRatio)
  // Rounded rect fill ratio vs bounding box is roughly 0.9 for r << hw,hh, so inflate target accordingly.
  const aspectXY = width / height;
  const fillRatio = 0.88;
  const depthRatio = depth / height;
  // particles ~ nx * ny * nz; nx ~ ny*aspectXY; nz ~ ny*depthRatio
  const ny = Math.max(3, Math.round(Math.cbrt(targetParticles / (fillRatio * aspectXY * depthRatio))));
  const nx = Math.max(3, Math.round(ny * aspectXY));
  const nz = Math.max(2, Math.round(ny * depthRatio));

  const positions = [];
  const grid = new Array(nx * ny * nz).fill(-1);

  const idx3 = (ix, iy, iz) => ix + nx * (iy + ny * iz);

  for (let iz = 0; iz < nz; iz++) {
    const z = -hd + (depth * iz) / (nz - 1);
    for (let iy = 0; iy < ny; iy++) {
      const y = -hh + (height * iy) / (ny - 1);
      for (let ix = 0; ix < nx; ix++) {
        const x = -hw + (width * ix) / (nx - 1);
        const d = roundedRectSDF(x, y, hw, hh, cornerRadius);
        if (d <= 1e-6) {
          grid[idx3(ix, iy, iz)] = positions.length / 3;
          positions.push(x, y, z);
        }
      }
    }
  }

  const tets = [];
  // For each lattice cube with all 8 corners present, split into 6 tets (consistent diagonal direction
  // so adjacent cubes share triangular faces exactly — using the standard "6-tet cube" decomposition
  // along the main diagonal from corner 0 to corner 6).
  const cubeTets = [
    [0, 1, 3, 7], [0, 1, 7, 5], [0, 5, 7, 4],
    [0, 3, 2, 7], [0, 2, 6, 7], [0, 6, 4, 7],
  ];

  for (let iz = 0; iz < nz - 1; iz++) {
    for (let iy = 0; iy < ny - 1; iy++) {
      for (let ix = 0; ix < nx - 1; ix++) {
        const c = [
          grid[idx3(ix, iy, iz)],       grid[idx3(ix + 1, iy, iz)],
          grid[idx3(ix, iy + 1, iz)],   grid[idx3(ix + 1, iy + 1, iz)],
          grid[idx3(ix, iy, iz + 1)],   grid[idx3(ix + 1, iy, iz + 1)],
          grid[idx3(ix, iy + 1, iz + 1)], grid[idx3(ix + 1, iy + 1, iz + 1)],
        ];
        if (c.some((v) => v < 0)) continue; // cube not fully inside outline, skip (surface stays approximate at edges)
        for (const [a, b, cc, d] of cubeTets) {
          tets.push(c[a], c[b], c[cc], c[d]);
        }
      }
    }
  }

  const positionsArr = new Float32Array(positions);
  const tetsArr = new Uint32Array(tets);

  let minY = Infinity, maxY = -Infinity;
  for (let i = 1; i < positionsArr.length; i += 3) {
    if (positionsArr[i] < minY) minY = positionsArr[i];
    if (positionsArr[i] > maxY) maxY = positionsArr[i];
  }

  return {
    positions: positionsArr,
    tets: tetsArr,
    particleCount: positionsArr.length / 3,
    tetCount: tetsArr.length / 4,
    layerHeightMin: minY,
    layerHeightMax: maxY,
  };
}
```

- [ ] **Step 4: Run test block again, confirm pass, then delete the temporary test `<script>` block from `index.html`**

Reload, confirm console prints `MESH TESTS PASSED <particleCount> <tetCount> minVol <value>` with particleCount/tetCount inside the asserted ranges. Remove the temporary test block (Step 1's code) from `index.html` entirely — it was for development verification only, not part of the shipped app.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: procedural tet mesh builder from rounded-rect outline"
```

### Task 1.3: XPBD solver — co-rotational elastic + volume constraints, gravity, floor contact, fixed-step accumulator

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `buildJellyMesh()` output shape `{ positions, tets, particleCount, tetCount }` (Task 1.2).
- Produces: `class SoftBody` with constructor `new SoftBody(meshData, opts)` where `opts = { layerStiffness: (restY) => number }`; instance fields `positions` (current, Float32Array len N*3), `restPositions`, `prevPositions`, `velocities`, `invMass` (Float32Array len N), `tets`, `tetRestVolumes`, `tetRestInvMatrices` (Float32Array len T*9, inverse of rest-shape edge matrix per tet), `tetStiffness` (Float32Array len T); method `step(dt, globalParams)` where `globalParams = { firmness, damping, gravity: [x,y,z], floorY, substeps }` — advances exactly one fixed physics tick (caller handles substep loop and accumulator); method `computeStats()` returning `{ mass, volumeRatio, kineticEnergy }`.
- Produces: top-level `function fixedStepAccumulator(maxCatchupSteps = 5)` object pattern used by the render loop — implemented as `class Clock { tick(nowMs, stepFn) }` that clamps huge deltas.

- [ ] **Step 1: Add inline dev-test block for solver stability (temporary, same pattern as Task 1.2)**

```js
// TEMPORARY - remove after Step 4
(function testSolverStability() {
  const mesh = buildJellyMesh({});
  const body = new SoftBody(mesh, { layerStiffness: () => 1.0 });
  const params = { firmness: 0.5, damping: 0.5, gravity: [0, -9.8, 0], floorY: -10, substeps: 10 };
  for (let i = 0; i < 300; i++) {
    body.step(1 / 600, params); // 10 substeps * 60Hz = 600Hz sub-steps
  }
  let maxAbs = 0;
  for (let i = 0; i < body.positions.length; i++) {
    const v = Math.abs(body.positions[i]);
    if (!isFinite(v)) throw new Error('ASSERT FAILED: non-finite position after free fall');
    if (v > maxAbs) maxAbs = v;
  }
  if (maxAbs > 1000) throw new Error('ASSERT FAILED: solver exploded, maxAbs=' + maxAbs);
  const stats = body.computeStats();
  if (!(stats.volumeRatio > 0.5 && stats.volumeRatio < 1.5)) throw new Error('ASSERT FAILED: volume ratio out of range: ' + stats.volumeRatio);
  console.log('SOLVER STABILITY TEST PASSED, maxAbs', maxAbs, 'volumeRatio', stats.volumeRatio);
})();
```

- [ ] **Step 2: Run, confirm it fails (`SoftBody is not defined`)**

- [ ] **Step 3: Implement `SoftBody` and `Clock`**

```js
function mat3Multiply(a, b, out) {
  out[0] = a[0]*b[0] + a[3]*b[1] + a[6]*b[2];
  out[1] = a[1]*b[0] + a[4]*b[1] + a[7]*b[2];
  out[2] = a[2]*b[0] + a[5]*b[1] + a[8]*b[2];
  out[3] = a[0]*b[3] + a[3]*b[4] + a[6]*b[5];
  out[4] = a[1]*b[3] + a[4]*b[4] + a[7]*b[5];
  out[5] = a[2]*b[3] + a[5]*b[4] + a[8]*b[5];
  out[6] = a[0]*b[6] + a[3]*b[7] + a[6]*b[8];
  out[7] = a[1]*b[6] + a[4]*b[7] + a[7]*b[8];
  out[8] = a[2]*b[6] + a[5]*b[7] + a[8]*b[8];
  return out;
}

function mat3Determinant(m) {
  return m[0]*(m[4]*m[8]-m[5]*m[7]) - m[3]*(m[1]*m[8]-m[2]*m[7]) + m[6]*(m[1]*m[5]-m[2]*m[4]);
}

function mat3Inverse(m, out) {
  const det = mat3Determinant(m);
  const invDet = Math.abs(det) > 1e-12 ? 1 / det : 0;
  out[0] = (m[4]*m[8]-m[5]*m[7]) * invDet;
  out[1] = (m[2]*m[7]-m[1]*m[8]) * invDet;
  out[2] = (m[1]*m[5]-m[2]*m[4]) * invDet;
  out[3] = (m[5]*m[6]-m[3]*m[8]) * invDet;
  out[4] = (m[0]*m[8]-m[2]*m[6]) * invDet;
  out[5] = (m[2]*m[3]-m[0]*m[5]) * invDet;
  out[6] = (m[3]*m[7]-m[4]*m[6]) * invDet;
  out[7] = (m[1]*m[6]-m[0]*m[7]) * invDet;
  out[8] = (m[0]*m[4]-m[1]*m[3]) * invDet;
  return out;
}

// Polar decomposition via Jacobi-free iterative method (Müller et al.) — extracts best-fit rotation R from F.
function extractRotation(F, out) {
  let q = [1, 0, 0, 0]; // identity quaternion (w,x,y,z)
  for (let iter = 0; iter < 8; iter++) {
    const qx = q[1], qy = q[2], qz = q[3], qw = q[0];
    const r00 = 1-2*(qy*qy+qz*qz), r01 = 2*(qx*qy-qz*qw), r02 = 2*(qx*qz+qy*qw);
    const r10 = 2*(qx*qy+qz*qw), r11 = 1-2*(qx*qx+qz*qz), r12 = 2*(qy*qz-qx*qw);
    const r20 = 2*(qx*qz-qy*qw), r21 = 2*(qy*qz+qx*qw), r22 = 1-2*(qx*qx+qy*qy);
    const cx = (r10*F[2]+r11*F[5]+r12*F[8]) - (r20*F[1]+r21*F[4]+r22*F[7]);
    const cy = (r20*F[0]+r21*F[3]+r22*F[6]) - (r00*F[2]+r01*F[5]+r02*F[8]);
    const cz = (r00*F[1]+r01*F[4]+r02*F[7]) - (r10*F[0]+r11*F[3]+r12*F[6]);
    const dot = r00*F[0]+r01*F[3]+r02*F[6] + r10*F[1]+r11*F[4]+r12*F[7] + r20*F[2]+r21*F[5]+r22*F[8];
    const omegaLen2 = cx*cx+cy*cy+cz*cz;
    if (omegaLen2 < 1e-9) break;
    const w = Math.abs(dot) + 1e-6;
    const omega = [cx / w, cy / w, cz / w];
    const angle = Math.sqrt(omega[0]*omega[0]+omega[1]*omega[1]+omega[2]*omega[2]);
    if (angle < 1e-9) break;
    const axis = [omega[0]/angle, omega[1]/angle, omega[2]/angle];
    const half = angle / 2;
    const s = Math.sin(half);
    const dq = [Math.cos(half), axis[0]*s, axis[1]*s, axis[2]*s];
    const nq = [
      dq[0]*q[0] - dq[1]*q[1] - dq[2]*q[2] - dq[3]*q[3],
      dq[0]*q[1] + dq[1]*q[0] + dq[2]*q[3] - dq[3]*q[2],
      dq[0]*q[2] - dq[1]*q[3] + dq[2]*q[0] + dq[3]*q[1],
      dq[0]*q[3] + dq[1]*q[2] - dq[2]*q[1] + dq[3]*q[0],
    ];
    const len = Math.hypot(nq[0], nq[1], nq[2], nq[3]) || 1;
    q = [nq[0]/len, nq[1]/len, nq[2]/len, nq[3]/len];
  }
  const qx = q[1], qy = q[2], qz = q[3], qw = q[0];
  out[0] = 1-2*(qy*qy+qz*qz); out[1] = 2*(qx*qy-qz*qw);   out[2] = 2*(qx*qz+qy*qw);
  out[3] = 2*(qx*qy+qz*qw);   out[4] = 1-2*(qx*qx+qz*qz); out[5] = 2*(qy*qz-qx*qw);
  out[6] = 2*(qx*qz-qy*qw);   out[7] = 2*(qy*qz+qx*qw);   out[8] = 1-2*(qx*qx+qy*qy);
  return out;
}

class SoftBody {
  constructor(meshData, opts = {}) {
    const n = meshData.particleCount;
    const t = meshData.tetCount;
    this.particleCount = n;
    this.tetCount = t;
    this.restPositions = new Float32Array(meshData.positions);
    this.positions = new Float32Array(meshData.positions);
    this.prevPositions = new Float32Array(meshData.positions);
    this.velocities = new Float32Array(n * 3);
    this.invMass = new Float32Array(n).fill(1);
    this.tets = new Uint32Array(meshData.tets);
    this.tetRestVolumes = new Float32Array(t);
    this.tetRestInvMatrices = new Float32Array(t * 9);
    this.tetStiffness = new Float32Array(t);

    const layerFn = opts.layerStiffness || (() => 1.0);

    for (let i = 0; i < t; i++) {
      const a = this.tets[i*4], b = this.tets[i*4+1], c = this.tets[i*4+2], d = this.tets[i*4+3];
      const ax = this.restPositions[a*3], ay = this.restPositions[a*3+1], az = this.restPositions[a*3+2];
      const e1 = [this.restPositions[b*3]-ax, this.restPositions[b*3+1]-ay, this.restPositions[b*3+2]-az];
      const e2 = [this.restPositions[c*3]-ax, this.restPositions[c*3+1]-ay, this.restPositions[c*3+2]-az];
      const e3 = [this.restPositions[d*3]-ax, this.restPositions[d*3+1]-ay, this.restPositions[d*3+2]-az];
      const m = [e1[0], e2[0], e3[0], e1[1], e2[1], e3[1], e1[2], e2[2], e3[2]];
      const vol = mat3Determinant(m) / 6;
      this.tetRestVolumes[i] = Math.abs(vol);
      const inv = new Array(9);
      mat3Inverse(m, inv);
      for (let k = 0; k < 9; k++) this.tetRestInvMatrices[i*9+k] = inv[k];
      const centroidY = (ay + this.restPositions[b*3+1] + this.restPositions[c*3+1] + this.restPositions[d*3+1]) / 4;
      this.tetStiffness[i] = layerFn(centroidY);
    }

    // mass lumping: distribute tet volume*density equally to its 4 particles
    const massAccum = new Float32Array(n);
    const density = 1.0;
    for (let i = 0; i < t; i++) {
      const share = (this.tetRestVolumes[i] * density) / 4;
      for (let k = 0; k < 4; k++) massAccum[this.tets[i*4+k]] += share;
    }
    for (let i = 0; i < n; i++) {
      this.invMass[i] = massAccum[i] > 1e-9 ? 1 / massAccum[i] : 0;
    }
    this._totalMass = 0;
    for (let i = 0; i < n; i++) this._totalMass += massAccum[i];

    this._F = new Array(9);
    this._R = new Array(9);
  }

  step(dt, globalParams) {
    const n = this.particleCount;
    const g = globalParams.gravity;
    const floorY = globalParams.floorY;
    const firmness = globalParams.firmness;
    const damping = globalParams.damping;

    for (let i = 0; i < n; i++) {
      if (this.invMass[i] === 0) continue;
      this.prevPositions[i*3] = this.positions[i*3];
      this.prevPositions[i*3+1] = this.positions[i*3+1];
      this.prevPositions[i*3+2] = this.positions[i*3+2];
      this.velocities[i*3]   += g[0] * dt;
      this.velocities[i*3+1] += g[1] * dt;
      this.velocities[i*3+2] += g[2] * dt;
      this.positions[i*3]   += this.velocities[i*3] * dt;
      this.positions[i*3+1] += this.velocities[i*3+1] * dt;
      this.positions[i*3+2] += this.velocities[i*3+2] * dt;
    }

    this._solveTetConstraints(dt, firmness);
    this._solveFloorContact(floorY);

    for (let i = 0; i < n; i++) {
      if (this.invMass[i] === 0) continue;
      this.velocities[i*3]   = (this.positions[i*3]   - this.prevPositions[i*3])   / dt;
      this.velocities[i*3+1] = (this.positions[i*3+1] - this.prevPositions[i*3+1]) / dt;
      this.velocities[i*3+2] = (this.positions[i*3+2] - this.prevPositions[i*3+2]) / dt;
    }

    this._applyEdgeDamping(damping);
  }

  _solveTetConstraints(dt, firmness) {
    const t = this.tetCount;
    const complianceScale = Math.max(1e-4, 1 - firmness); // firmness=1 -> near-zero compliance (stiff); firmness=0 -> soft
    const alphaElastic = 0.0001 * complianceScale;
    const alphaVolume = 0.00002 * complianceScale;
    const dtSq = dt * dt;

    for (let i = 0; i < t; i++) {
      const a = this.tets[i*4], b = this.tets[i*4+1], c = this.tets[i*4+2], d = this.tets[i*4+3];
      const wa = this.invMass[a], wb = this.invMass[b], wc = this.invMass[c], wd = this.invMass[d];
      if (wa + wb + wc + wd === 0) continue;

      const ax = this.positions[a*3], ay = this.positions[a*3+1], az = this.positions[a*3+2];
      const e1 = [this.positions[b*3]-ax, this.positions[b*3+1]-ay, this.positions[b*3+2]-az];
      const e2 = [this.positions[c*3]-ax, this.positions[c*3+1]-ay, this.positions[c*3+2]-az];
      const e3 = [this.positions[d*3]-ax, this.positions[d*3+1]-ay, this.positions[d*3+2]-az];
      const D = [e1[0], e2[0], e3[0], e1[1], e2[1], e3[1], e1[2], e2[2], e3[2]];

      const invRest = this.tetRestInvMatrices.subarray(i*9, i*9+9);
      mat3Multiply(D, invRest, this._F);
      extractRotation(this._F, this._R);

      const stiffness = this.tetStiffness[i];
      const localAlphaElastic = alphaElastic / stiffness;
      const vol = this.tetRestVolumes[i];

      // Co-rotational elastic: target positions are rest shape rotated by R, applied as a position correction
      // toward (R * restEdge + a) for each of b,c,d — implemented as a direct per-vertex correction (shape matching style).
      const targets = [];
      for (const e of [e1, e2, e3]) {
        targets.push(e); // placeholder, overwritten below using rest edges
      }
      const restEdges = [
        [this.restPositions[b*3]-this.restPositions[a*3], this.restPositions[b*3+1]-this.restPositions[a*3+1], this.restPositions[b*3+2]-this.restPositions[a*3+2]],
        [this.restPositions[c*3]-this.restPositions[a*3], this.restPositions[c*3+1]-this.restPositions[a*3+1], this.restPositions[c*3+2]-this.restPositions[a*3+2]],
        [this.restPositions[d*3]-this.restPositions[a*3], this.restPositions[d*3+1]-this.restPositions[a*3+1], this.restPositions[d*3+2]-this.restPositions[a*3+2]],
      ];
      const R = this._R;
      const rotatedEdges = restEdges.map(([ex, ey, ez]) => [
        R[0]*ex + R[1]*ey + R[2]*ez,
        R[3]*ex + R[4]*ey + R[5]*ez,
        R[6]*ex + R[7]*ey + R[8]*ez,
      ]);

      const verts = [a, b, c, d];
      const invMasses = [wa, wb, wc, wd];
      const compliance = localAlphaElastic / dtSq;
      const kStiff = 1 / (1 + compliance);

      // centroid-preserving correction: pull each of b,c,d toward a + rotatedEdge, and pull a oppositely
      let corrAx = 0, corrAy = 0, corrAz = 0;
      const corrections = [[0,0,0], [0,0,0], [0,0,0], [0,0,0]];
      for (let k = 0; k < 3; k++) {
        const vIdx = k + 1;
        const curX = this.positions[verts[vIdx]*3]   - ax;
        const curY = this.positions[verts[vIdx]*3+1] - ay;
        const curZ = this.positions[verts[vIdx]*3+2] - az;
        const diffX = (curX - rotatedEdges[k][0]) * kStiff;
        const diffY = (curY - rotatedEdges[k][1]) * kStiff;
        const diffZ = (curZ - rotatedEdges[k][2]) * kStiff;
        const wSum = invMasses[vIdx] + wa;
        if (wSum < 1e-9) continue;
        const moveV = invMasses[vIdx] / wSum;
        const moveA = wa / wSum;
        corrections[vIdx][0] -= diffX * moveV;
        corrections[vIdx][1] -= diffY * moveV;
        corrections[vIdx][2] -= diffZ * moveV;
        corrAx += diffX * moveA;
        corrAy += diffY * moveA;
        corrAz += diffZ * moveA;
      }
      corrections[0][0] += corrAx;
      corrections[0][1] += corrAy;
      corrections[0][2] += corrAz;

      for (let k = 0; k < 4; k++) {
        if (invMasses[k] === 0) continue;
        this.positions[verts[k]*3]   += corrections[k][0] / 3;
        this.positions[verts[k]*3+1] += corrections[k][1] / 3;
        this.positions[verts[k]*3+2] += corrections[k][2] / 3;
      }

      // Volume constraint (recompute edges after elastic correction for a stabler estimate — reuse same-step edges, acceptable for XPBD iteration)
      const bx = this.positions[b*3]-this.positions[a*3], by = this.positions[b*3+1]-this.positions[a*3+1], bz = this.positions[b*3+2]-this.positions[a*3+2];
      const cx2 = this.positions[c*3]-this.positions[a*3], cy2 = this.positions[c*3+1]-this.positions[a*3+1], cz2 = this.positions[c*3+2]-this.positions[a*3+2];
      const dx2 = this.positions[d*3]-this.positions[a*3], dy2 = this.positions[d*3+1]-this.positions[a*3+1], dz2 = this.positions[d*3+2]-this.positions[a*3+2];
      const crossCX = cy2*dz2 - cz2*dy2, crossCY = cz2*dx2 - cx2*dz2, crossCZ = cx2*dy2 - cy2*dx2;
      const currentVol = (bx*crossCX + by*crossCY + bz*crossCZ) / 6;
      const C = currentVol - vol;
      const gradB = [ (cy2*dz2 - cz2*dy2)/6, (cz2*dx2 - cx2*dz2)/6, (cx2*dy2 - cy2*dx2)/6 ];
      const gradC = [ (dy2*bz - dz2*by)/6, (dz2*bx - dx2*bz)/6, (dx2*by - dy2*bx)/6 ];
      const gradD = [ (by*cy2*0) , 0, 0 ]; // placeholder overwritten below
      const gradDx = (by*cz2 - bz*cy2)/6, gradDy = (bz*cx2 - bx*cz2)/6, gradDz = (bx*cy2 - by*cx2)/6;
      const gradAx = -(gradB[0]+gradC[0]+gradDx), gradAy = -(gradB[1]+gradC[1]+gradDy), gradAz = -(gradB[2]+gradC[2]+gradDz);

      const wSumVol = wa*(gradAx*gradAx+gradAy*gradAy+gradAz*gradAz)
                    + wb*(gradB[0]*gradB[0]+gradB[1]*gradB[1]+gradB[2]*gradB[2])
                    + wc*(gradC[0]*gradC[0]+gradC[1]*gradC[1]+gradC[2]*gradC[2])
                    + wd*(gradDx*gradDx+gradDy*gradDy+gradDz*gradDz);
      const complianceVol = alphaVolume / dtSq;
      if (wSumVol + complianceVol > 1e-9) {
        const lambda = -C / (wSumVol + complianceVol);
        this.positions[a*3]   += wa * gradAx * lambda;
        this.positions[a*3+1] += wa * gradAy * lambda;
        this.positions[a*3+2] += wa * gradAz * lambda;
        this.positions[b*3]   += wb * gradB[0] * lambda;
        this.positions[b*3+1] += wb * gradB[1] * lambda;
        this.positions[b*3+2] += wb * gradB[2] * lambda;
        this.positions[c*3]   += wc * gradC[0] * lambda;
        this.positions[c*3+1] += wc * gradC[1] * lambda;
        this.positions[c*3+2] += wc * gradC[2] * lambda;
        this.positions[d*3]   += wd * gradDx * lambda;
        this.positions[d*3+1] += wd * gradDy * lambda;
        this.positions[d*3+2] += wd * gradDz * lambda;
      }
    }
  }

  _solveFloorContact(floorY) {
    const n = this.particleCount;
    const friction = 0.4;
    for (let i = 0; i < n; i++) {
      if (this.invMass[i] === 0) continue;
      if (this.positions[i*3+1] < floorY) {
        const penetration = floorY - this.positions[i*3+1];
        this.positions[i*3+1] = floorY;
        const dx = this.positions[i*3]   - this.prevPositions[i*3];
        const dz = this.positions[i*3+2] - this.prevPositions[i*3+2];
        this.positions[i*3]   -= dx * friction;
        this.positions[i*3+2] -= dz * friction;
      }
    }
  }

  _applyEdgeDamping(damping) {
    if (damping <= 0) return;
    const t = this.tetCount;
    const k = Math.min(0.5, damping * 0.5);
    const edgePairs = [[0,1],[0,2],[0,3],[1,2],[1,3],[2,3]];
    for (let i = 0; i < t; i++) {
      const v = [this.tets[i*4], this.tets[i*4+1], this.tets[i*4+2], this.tets[i*4+3]];
      for (const [p, q] of edgePairs) {
        const a = v[p], b = v[q];
        const wa = this.invMass[a], wb = this.invMass[b];
        if (wa + wb === 0) continue;
        const rvx = this.velocities[a*3]   - this.velocities[b*3];
        const rvy = this.velocities[a*3+1] - this.velocities[b*3+1];
        const rvz = this.velocities[a*3+2] - this.velocities[b*3+2];
        const corrX = rvx * k, corrY = rvy * k, corrZ = rvz * k;
        const moveA = wa / (wa + wb), moveB = wb / (wa + wb);
        this.velocities[a*3]   -= corrX * moveA;
        this.velocities[a*3+1] -= corrY * moveA;
        this.velocities[a*3+2] -= corrZ * moveA;
        this.velocities[b*3]   += corrX * moveB;
        this.velocities[b*3+1] += corrY * moveB;
        this.velocities[b*3+2] += corrZ * moveB;
      }
    }
  }

  computeStats() {
    let volume = 0;
    let restVolume = 0;
    let kinetic = 0;
    for (let i = 0; i < this.tetCount; i++) {
      const a = this.tets[i*4], b = this.tets[i*4+1], c = this.tets[i*4+2], d = this.tets[i*4+3];
      const bx = this.positions[b*3]-this.positions[a*3], by = this.positions[b*3+1]-this.positions[a*3+1], bz = this.positions[b*3+2]-this.positions[a*3+2];
      const cx = this.positions[c*3]-this.positions[a*3], cy = this.positions[c*3+1]-this.positions[a*3+1], cz = this.positions[c*3+2]-this.positions[a*3+2];
      const dx = this.positions[d*3]-this.positions[a*3], dy = this.positions[d*3+1]-this.positions[a*3+1], dz = this.positions[d*3+2]-this.positions[a*3+2];
      const crossX = cy*dz-cz*dy, crossY = cz*dx-cx*dz, crossZ = cx*dy-cy*dx;
      volume += Math.abs(bx*crossX+by*crossY+bz*crossZ) / 6;
      restVolume += this.tetRestVolumes[i];
    }
    for (let i = 0; i < this.particleCount; i++) {
      if (this.invMass[i] === 0) continue;
      const mass = 1 / this.invMass[i];
      const vx = this.velocities[i*3], vy = this.velocities[i*3+1], vz = this.velocities[i*3+2];
      kinetic += 0.5 * mass * (vx*vx+vy*vy+vz*vz);
    }
    return {
      mass: this._totalMass,
      volumeRatio: restVolume > 0 ? volume / restVolume : 1,
      kineticEnergy: kinetic,
    };
  }
}

class Clock {
  constructor(hz = 60, maxCatchupSteps = 5) {
    this.dt = 1 / hz;
    this.accumulator = 0;
    this.lastTime = null;
    this.maxCatchupSteps = maxCatchupSteps;
  }
  tick(nowMs, stepFn) {
    if (this.lastTime === null) { this.lastTime = nowMs; return; }
    let frameTime = (nowMs - this.lastTime) / 1000;
    this.lastTime = nowMs;
    frameTime = Math.min(frameTime, this.dt * this.maxCatchupSteps);
    this.accumulator += frameTime;
    let steps = 0;
    while (this.accumulator >= this.dt && steps < this.maxCatchupSteps) {
      stepFn(this.dt);
      this.accumulator -= this.dt;
      steps++;
    }
  }
}
```

- [ ] **Step 4: Run dev-test block, confirm pass, then remove the temporary block**

Reload; console should print `SOLVER STABILITY TEST PASSED maxAbs <small-ish number> volumeRatio <near 1>`. If it throws "solver exploded" or a non-finite error, check `alphaElastic`/`alphaVolume` scaling and rotation extraction before proceeding — do not continue to Task 1.4 with an unstable solver. Once passing, delete the temporary `testSolverStability` IIFE from `index.html`.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: XPBD co-rotational + volume solver with fixed-step clock"
```

### Task 1.4: Minimal render loop — floor plane + wireframe tet mesh via WebGPU, "Show mesh" always on for this phase

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `app` (Task 1.1), `SoftBody`/`Clock` (Task 1.3).
- Produces: `function createWireframeRenderer(device, format)` returning `{ render(passEncoder, viewProj, body) }`; `function createFloorRenderer(device, format)` returning `{ render(passEncoder, viewProj) }`; global `camera` object `{ eye, target, up, fovY, near, far }` and `function getViewProjMatrix(camera, aspect)` returning a `Float32Array(16)`; the page's `requestAnimationFrame` loop wiring `Clock` → `SoftBody.step` → draw.

- [ ] **Step 1: Add a tiny mat4 helper block (no library — hand-rolled column-major 4x4)**

```js
const mat4 = {
  perspective(fovY, aspect, near, far) {
    const f = 1 / Math.tan(fovY / 2);
    const nf = 1 / (near - far);
    return new Float32Array([
      f / aspect, 0, 0, 0,
      0, f, 0, 0,
      0, 0, (far + near) * nf, -1,
      0, 0, 2 * far * near * nf, 0,
    ]);
  },
  lookAt(eye, target, up) {
    const zx = eye[0]-target[0], zy = eye[1]-target[1], zz = eye[2]-target[2];
    let zl = Math.hypot(zx, zy, zz) || 1;
    const z = [zx/zl, zy/zl, zz/zl];
    const xx = up[1]*z[2]-up[2]*z[1], xy = up[2]*z[0]-up[0]*z[2], xz = up[0]*z[1]-up[1]*z[0];
    let xl = Math.hypot(xx, xy, xz) || 1;
    const x = [xx/xl, xy/xl, xz/xl];
    const y = [z[1]*x[2]-z[2]*x[1], z[2]*x[0]-z[0]*x[2], z[0]*x[1]-z[1]*x[0]];
    return new Float32Array([
      x[0], y[0], z[0], 0,
      x[1], y[1], z[1], 0,
      x[2], y[2], z[2], 0,
      -(x[0]*eye[0]+x[1]*eye[1]+x[2]*eye[2]),
      -(y[0]*eye[0]+y[1]*eye[1]+y[2]*eye[2]),
      -(z[0]*eye[0]+z[1]*eye[1]+z[2]*eye[2]), 1,
    ]);
  },
  multiply(a, b) {
    const out = new Float32Array(16);
    for (let c = 0; c < 4; c++) {
      for (let r = 0; r < 4; r++) {
        out[c*4+r] = a[0*4+r]*b[c*4+0] + a[1*4+r]*b[c*4+1] + a[2*4+r]*b[c*4+2] + a[3*4+r]*b[c*4+3];
      }
    }
    return out;
  },
};

const camera = {
  eye: [3.5, 2.5, 4.5],
  target: [0, 0, 0],
  up: [0, 1, 0],
  fovY: Math.PI / 4,
  near: 0.1,
  far: 100,
};

function getViewProjMatrix(cam, aspect) {
  const view = mat4.lookAt(cam.eye, cam.target, cam.up);
  const proj = mat4.perspective(cam.fovY, aspect, cam.near, cam.far);
  return mat4.multiply(proj, view);
}
```

- [ ] **Step 2: Add floor renderer (simple flat quad, flat-shaded WGSL)**

```js
function createFloorRenderer(device, format) {
  const shaderCode = `
    struct Uniforms { viewProj: mat4x4<f32> };
    @group(0) @binding(0) var<uniform> uniforms: Uniforms;
    struct VSOut { @builtin(position) pos: vec4<f32>, @location(0) worldPos: vec3<f32> };
    @vertex
    fn vs_main(@location(0) position: vec3<f32>) -> VSOut {
      var out: VSOut;
      out.pos = uniforms.viewProj * vec4<f32>(position, 1.0);
      out.worldPos = position;
      return out;
    }
    @fragment
    fn fs_main(in: VSOut) -> @location(0) vec4<f32> {
      return vec4<f32>(0.953, 0.937, 0.910, 1.0);
    }
  `;
  const module = device.createShaderModule({ code: shaderCode });
  const size = 20;
  const verts = new Float32Array([
    -size, 0, -size,  size, 0, -size,  size, 0, size,
    -size, 0, -size,  size, 0, size,  -size, 0, size,
  ]);
  const vertexBuffer = device.createBuffer({ size: verts.byteLength, usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST });
  device.queue.writeBuffer(vertexBuffer, 0, verts);

  const uniformBuffer = device.createBuffer({ size: 64, usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST });
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [{ binding: 0, visibility: GPUShaderStage.VERTEX, buffer: {} }],
  });
  const bindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [{ binding: 0, resource: { buffer: uniformBuffer } }],
  });
  const pipeline = device.createRenderPipeline({
    layout: device.createPipelineLayout({ bindGroupLayouts: [bindGroupLayout] }),
    vertex: {
      module, entryPoint: 'vs_main',
      buffers: [{ arrayStride: 12, attributes: [{ shaderLocation: 0, offset: 0, format: 'float32x3' }] }],
    },
    fragment: { module, entryPoint: 'fs_main', targets: [{ format }] },
    primitive: { topology: 'triangle-list', cullMode: 'none' },
    depthStencil: { format: 'depth24plus', depthWriteEnabled: true, depthCompare: 'less' },
  });

  return {
    render(passEncoder, viewProj) {
      device.queue.writeBuffer(uniformBuffer, 0, viewProj);
      passEncoder.setPipeline(pipeline);
      passEncoder.setBindGroup(0, bindGroup);
      passEncoder.setVertexBuffer(0, vertexBuffer);
      passEncoder.draw(6);
    },
  };
}
```

- [ ] **Step 3: Add wireframe tet renderer (draws tet edges as line-list, rebuilds vertex buffer from live positions each frame)**

```js
function createWireframeRenderer(device, format) {
  const shaderCode = `
    struct Uniforms { viewProj: mat4x4<f32> };
    @group(0) @binding(0) var<uniform> uniforms: Uniforms;
    @vertex
    fn vs_main(@location(0) position: vec3<f32>) -> @builtin(position) vec4<f32> {
      return uniforms.viewProj * vec4<f32>(position, 1.0);
    }
    @fragment
    fn fs_main() -> @location(0) vec4<f32> {
      return vec4<f32>(0.1, 0.1, 0.1, 1.0);
    }
  `;
  const module = device.createShaderModule({ code: shaderCode });
  const uniformBuffer = device.createBuffer({ size: 64, usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST });
  const bindGroupLayout = device.createBindGroupLayout({
    entries: [{ binding: 0, visibility: GPUShaderStage.VERTEX, buffer: {} }],
  });
  const bindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [{ binding: 0, resource: { buffer: uniformBuffer } }],
  });
  const pipeline = device.createRenderPipeline({
    layout: device.createPipelineLayout({ bindGroupLayouts: [bindGroupLayout] }),
    vertex: { module, entryPoint: 'vs_main', buffers: [{ arrayStride: 12, attributes: [{ shaderLocation: 0, offset: 0, format: 'float32x3' }] }] },
    fragment: { module, entryPoint: 'fs_main', targets: [{ format }] },
    primitive: { topology: 'line-list' },
    depthStencil: { format: 'depth24plus', depthWriteEnabled: true, depthCompare: 'less' },
  });

  let lineBuffer = null;
  let lineBufferCapacity = 0;
  const edgePairs = [[0,1],[0,2],[0,3],[1,2],[1,3],[2,3]];

  return {
    render(passEncoder, viewProj, body) {
      const lineCount = body.tetCount * edgePairs.length;
      const needed = lineCount * 2 * 3;
      if (!lineBuffer || lineBufferCapacity < needed) {
        if (lineBuffer) lineBuffer.destroy();
        lineBuffer = device.createBuffer({ size: needed * 4, usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST });
        lineBufferCapacity = needed;
      }
      const data = new Float32Array(needed);
      let o = 0;
      for (let i = 0; i < body.tetCount; i++) {
        const v = [body.tets[i*4], body.tets[i*4+1], body.tets[i*4+2], body.tets[i*4+3]];
        for (const [p, q] of edgePairs) {
          const a = v[p], b = v[q];
          data[o++] = body.positions[a*3]; data[o++] = body.positions[a*3+1]; data[o++] = body.positions[a*3+2];
          data[o++] = body.positions[b*3]; data[o++] = body.positions[b*3+1]; data[o++] = body.positions[b*3+2];
        }
      }
      device.queue.writeBuffer(lineBuffer, 0, data);
      device.queue.writeBuffer(uniformBuffer, 0, viewProj);
      passEncoder.setPipeline(pipeline);
      passEncoder.setBindGroup(0, bindGroup);
      passEncoder.setVertexBuffer(0, lineBuffer);
      passEncoder.draw(lineCount * 2);
    },
  };
}
```

- [ ] **Step 4: Wire it all into `main()` — create one SoftBody, run the Clock-driven physics + render loop**

```js
function layerStiffnessFn(restY) {
  // bottom firmest, top softest, 6 bands across [-0.7, 0.7]
  const t = Math.min(1, Math.max(0, (restY + 0.7) / 1.4));
  return 1.6 - 1.0 * t; // 1.6 at bottom, 0.6 at top
}

let depthTexture = null;
function getDepthTexture(device, width, height) {
  if (depthTexture && depthTexture.width === width && depthTexture.height === height) return depthTexture;
  if (depthTexture) depthTexture.destroy();
  depthTexture = device.createTexture({
    size: [width, height], format: 'depth24plus', usage: GPUTextureUsage.RENDER_ATTACHMENT,
  });
  return depthTexture;
}

async function main() {
  try {
    await initWebGPU();
  } catch (err) {
    console.error('WebGPU init failed:', err);
    showFallback();
    return;
  }

  const mesh = buildJellyMesh({});
  const body = new SoftBody(mesh, { layerStiffness: layerStiffnessFn });
  console.log('Jelly mesh ready:', body.particleCount, 'particles,', body.tetCount, 'tets');

  const floorRenderer = createFloorRenderer(app.device, app.format);
  const wireRenderer = createWireframeRenderer(app.device, app.format);
  const clock = new Clock(60);

  const physicsParams = { firmness: 0.5, damping: 0.5, gravity: [0, -9.8, 0], floorY: -0.7, substeps: 10 };

  function physicsStep(dt) {
    const subDt = dt / physicsParams.substeps;
    for (let i = 0; i < physicsParams.substeps; i++) {
      body.step(subDt, physicsParams);
    }
  }

  function frame(nowMs) {
    resizeCanvas();
    clock.tick(nowMs, physicsStep);

    const aspect = app.width / app.height;
    const viewProj = getViewProjMatrix(camera, aspect);
    const depthTex = getDepthTexture(app.device, app.width, app.height);

    const encoder = app.device.createCommandEncoder();
    const pass = encoder.beginRenderPass({
      colorAttachments: [{
        view: app.context.getCurrentTexture().createView(),
        clearValue: { r: 0.953, g: 0.937, b: 0.910, a: 1 },
        loadOp: 'clear', storeOp: 'store',
      }],
      depthStencilAttachment: {
        view: depthTex.createView(), depthClearValue: 1.0, depthLoadOp: 'clear', depthStoreOp: 'store',
      },
    });
    floorRenderer.render(pass, viewProj);
    wireRenderer.render(pass, viewProj, body);
    pass.end();
    app.device.queue.submit([encoder.finish()]);

    requestAnimationFrame(frame);
  }
  requestAnimationFrame(frame);
}

main();
```

- [ ] **Step 5: Manual browser check**

Reload. Confirm: console logs particle/tet counts in range; a wireframe jelly-bar shape (rounded rectangular block) is visible above a pale floor plane, falling under gravity and settling/wobbling without flying apart or passing through the floor. Test stability across slider-equivalent values by temporarily editing `physicsParams.firmness` to `0` and to `1` in the console (`physicsParams.firmness = 0`) and confirm no explosion at either extreme (per spec requirement "Confirm stable, no explosions at any slider value"). Also try `physicsParams.damping = 0` and `1`. Revert to `0.5`/`0.5` when done.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: render loop with floor + wireframe tet mesh, physics wired to Clock"
```

---

## Phase 2 — Hand tool (mouse + touch + twist)

Written after Phase 1 shipped, against the real code. Spec: "grab any point (tip, corner, any layer) — ray-pick the surface, attach a soft spring to the patch of nearby particles, bounded reach. Drag to pull. Scroll wheel or a second finger while holding twists it. Full multi-touch support."

Verification follows Phase 1's method: temporary in-page assertion blocks run via Playwright (RED before code, GREEN after, then removed), plus Playwright-driven pointer/wheel input and screenshots.

### Task 2.1: Boundary surface + ray picking

**Interfaces:**
- Produces: `SoftBody.surfaceTris` (Uint32Array, outward-wound boundary triangles, 3 indices each) and `surfaceTriCount`. Phase 3 will reuse these as the base skin mesh.
- Produces: `mat4.invert(m) -> Float32Array|null`; `screenToRay(clientX, clientY, invViewProj) -> {origin:[3], dir:[3]}` (dir normalized).
- Produces: `pickBody(body, ray) -> {t, point:[3], tri} | null`, using Möller–Trumbore against `surfaceTris`, returning the nearest hit.

**Tests (temporary assertion block):**
- Every edge of `surfaceTris` is shared by exactly 2 surface triangles (closed manifold).
- The divergence-theorem volume of the surface (Σ dot(v0, cross(v1, v2)) / 6) is > 0 and within 1% of Σ|tetRestVolumes| (all triangles wound outward).
- A ray from the camera eye toward the body centroid hits, with hit.point on the camera-facing side.
- A ray pointing away from the body returns null.

### Task 2.2: Grab spring + pointer drag (mouse + multi-touch)

**Interfaces:**
- Consumes: `pickBody`, `screenToRay`, `camera`, `getViewProjMatrix`.
- Produces: `SoftBody.grabs` (array) and `SoftBody._solveGrabs(dt)`, called in `step()` after the tet constraints and before floor contact. Each grab is `{ indices: Uint32Array, offsets: Float32Array (rest-of-patch offsets from the anchor, in world space at grab time), weights: Float32Array (smooth falloff 1→0 over the radius), target: [3], twist: number, axis: [3], compliance: number }`. Each particle i in the patch is pulled toward `target + Rot(axis, twist) · offset_i` as a single-particle XPBD constraint scaled by weight_i.
- Produces: `createHandTool({ canvas, getBodies, getInvViewProj, floorY })` that installs pointer listeners on the canvas. Each pointer that hits jelly gets its own grab, so several fingers pull independently. The drag target moves on a camera-facing plane through the hit point, is clamped to `MAX_REACH = 2.5` from the grab start (bounded reach), and is kept at least 0.05 above the floor. Pointer up or cancel removes that pointer's grab. Uses `setPointerCapture`.
- Patch: particles within `GRAB_RADIUS = 0.35` of the hit point, at least 1 (the nearest vertex of the hit triangle as a fallback — Review Focus: a tip or corner grab must never silently no-op).
- Cursor: `grab` while idle, `grabbing` while any grab is active.

**Tests:**
- A Playwright mouse drag from the bar's top surface upward raises the grabbed region's centroid by more than 0.5, with no NaN.
- After release, `body.grabs.length === 0` and the body settles back (volume ratio within [0.9, 1.1]).
- A grab at the extreme corner (tip) attaches at least 1 particle.
- Dragging the pointer far off-screen keeps the target within MAX_REACH of the grab start.

### Task 2.3: Twist (wheel + second finger)

**Interfaces:**
- Wheel while a mouse grab is active adds `-deltaY * 0.004` rad to that grab's `twist` and calls `preventDefault`. The axis is the view direction (camera eye → grab anchor).
- Touch: a pointer that goes down on empty space while another touch grab is active becomes that grab's twist finger. Twist is driven by the change in the angle of the vector from the grab pointer to the twist pointer. Lifting the twist finger leaves the twist where it is.

**Tests:**
- With a grab held, wheel scrolling (5 × deltaY = -100) rotates the patch: the angle between a patch particle's offset before and after, about the view axis, is > 0.3 rad.
- A synthetic second touch pointer rotated 90° around the first changes `grab.twist` by about π/2 (±0.2).
