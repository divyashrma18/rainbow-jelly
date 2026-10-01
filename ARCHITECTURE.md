# melon-original.html — architecture map

Single file: CSS + HTML (lines 1–260), then one `<script type="module">` made of concatenated "files" (`// ===== name.js =====`).

Coordinates: the outline lives in the rest plane **(u, w) = world (x, z)**. Bodies are extruded **upward along y** from 0 to `SHAPE.T`, so "height" means rest-space `y`.

Baseline (1440×900, Chrome): 608 particles, 2,277 tets, 12,794 render vertices, 60 fps.

| Topic | Where | Notes |
|---|---|---|
| Rest shape constants | `SHAPE` (~L270) | wedge `alpha`, `Ri`, corner `rho`, thickness `T`, `bevel`, `skin`, `pale` depths |
| Outline of the live pieces | `initialPiece()` (~L831) | every piece is a convex **inner polygon `I`** grown by `RC = SHAPE.rho`, so every corner is rounded, including knife cuts |
| Outline helpers | `pieceSamples`, `sdPiece`, `bboxLattice`, `clipHalf`, `splitPiece`, `lineCrossesPiece` (L839–925) | the cutter splits `I` along a rest-plane line |
| Tet mesh generation | `buildPieceSim(I, h=0.15, layers=3)` (~L928) | 2D Delaunay of outline samples + hex lattice, extruded into `layers` prisms, 3 tets per prism |
| Legacy (unused) | `buildSimMesh`, `buildRenderMesh`, `outlineSamples`, `hexLattice`, `sdShape` | only defined, never called |
| Per-region stiffness | `buildPieceSim` sets `region[]` per particle (0 flesh, 1 pale, 2 rind, from radial depth `Ro − r`); `SoftBody` constructor (~L1232) maps the 4-vertex region sum to `tetFirm` (1.0 / 0.75 / 0.5 / 0.3) | `tetFirm` **multiplies compliance**, so lower = stiffer |
| Solver | `class SoftBody` (L1157–1701) | XPBD, 10 substeps @60 Hz, co-rotational shape matching (`solveTets`), volume (`solveVolumes`), grab, floor friction, edge damping, piece collision, blade press/wedge (`solveBlade`) |
| Seeds & bubbles | `globalProps()` (~L975) | built once in the shared rest frame; `buildPieceRender` keeps the props that lie wholly inside a piece; `mat` 1 = seed, 2 = bubble |
| Render mesh per piece | `buildPieceRender(I)` (~L1043) | rounded walls (bevel rings) + top/bottom cap Delaunay + props |
| Barycentric binding | `embed()` (~L633), used by `ensureCache` and `installWorld` | every render vertex → tet + 4 weights; `skin()` in main applies them each frame |
| World assembly | `buildWorld(pieces)` (~L1109) | all pieces concatenated into one sim + one render mesh (`comp` / `vComp` = piece id) |
| Surface colour / material regions | WGSL `fsJelly` (~L2016) | flesh / pale / skin weights computed **in the shader** from rest coords (`r = length(Q.xz)`, depth `Ro − r`), with rind stripes; absorption `sigma`, scattering `scatK`, albedo, glow per region |
| Props shading | WGSL `fsProps` (~L1911) | seeds, knife steel, walnut handle, bubbles |
| Uniform block | WGSL `struct U` + `Renderer.setUniforms` (~L2458), `UBO_SIZE = 544` | palette fields flesh / fleshDeep / pale / skin / stripe / seed / shadowTint |
| Variety palettes | `PALETTES` (~L2597) | crimson / golden / rosé, linear RGB; swatch chips built in `main()` (3 `<i>` stripes, CSS `.swatch .chip i:nth-child`) |
| Camera framing | `updateCamera`, `target`, `followSlice` (~L2749) | tuned to the wedge footprint (`target z ≈ 1.0`, `Rh = 1.32`) |
| Light / shadow maps | `updateLight` | ortho extents ±2.4 / ±2.6 around the piece centroid |
| Knife | `buildKnifeMesh`, `planCut`, `installWorld`, `runKnifeAnim` | the plan is built on release; the world is swapped at break-through (`a.committed`) |
| UI text | HTML L177–260 (masthead, panel, readouts, notes), `setTool()` hint strings, toast strings in `planCut` | |
