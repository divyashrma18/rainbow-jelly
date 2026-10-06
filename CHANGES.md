# Rainbow Jelly — changes versus `melon-original.html`

All edits are in `index.html`. `melon-original.html` is untouched. The XPBD solver, cutting pipeline, render passes and UI structure are unchanged except where listed below.

## Shape (Step 1)
- `SHAPE` now describes a rounded bar: 4 × 1.4 in the outline plane (`hx 2.0`, `hz 0.7`), corner radius `rho 0.15`, height `T 1.0`, edge bevel `0.13`.
- `initialPiece()` returns the inset rectangle. The cutter's existing "convex polygon grown by `RC`" logic gives the rounded corners, so cut pieces get rounded corners the same way.
- Sim mesh spacing `simH 0.26` with `simLayers 4`: 570 particles / 2,184 tets (original 608 / 2,277).
- Removed the wedge-only builders that nothing called (`buildSimMesh`, `buildRenderMesh`, `outlineSamples`, `hexLattice`, `sdShape`).
- Camera target, slice-follow, framing, light/shadow and height-map extents, and the knife's home position are re-centred on the bar.

## Material (Step 2)
- `fsJelly`: the flesh / pale / skin regions are replaced by 6 colour bands chosen by **rest-space height**. Band boundaries are soft and slightly hand-poured (fbm wobble). Because the bands come from rest coordinates, they deform with the body and show on every cut face.
- Milky panna-cotta seams at 2/6 and 4/6 of the height reuse the melon's pale-layer scattering (`sigP`, `scatK 4.5`, `u.pale`).
- Beer–Lambert absorption, albedo and the thin-edge glow now use each band's own colour. Skin-only terms (stripes, specks, skin Fresnel/roughness) are removed.
- `struct U` gains `layers: array<vec4f, 6>`. `UBO_SIZE` goes 544 → 640 (and → 656 in Step 3). `shape` uniform is now `T, seam half-width, seam 1 y, seam 2 y`.
- Per-layer stiffness: `SoftBody` sets `tetFirm` from `layerCompliance(restY)` instead of the rind/flesh region sum. The bottom is firmest, the top softest, and tets near the milky seams are slightly stiffer.

## Bubbles and handle (Step 3)
- Black seeds removed. 16 tiny air bubbles are scattered through all six layers, with the same barycentric binding as before.
- `@_thecodingdiva_` is softly embossed on the top face. `embossCanvas()` draws a blurred height map laid out in rest-space (x, z). `fsJelly` turns it into a surface-gradient bump on the top face only (new binding 4 `embossTex`, `u.emboss`). It is visual only: it stretches with the jelly, and each piece after a cut keeps exactly the part of the handle it contains.

## Varieties (Step 4)
- `PALETTES` holds Rainbow (spec colours), Neon (electric) and Pastel (macaron). They are authored as sRGB hex and stored linear by `variety()`, which still fills the older uniform slots so the uniform layout stays the same.
- Swatches show a six-stripe preview. Default variety is Rainbow.

## Sound (Step 5)
- New `Sfx` object, synthesised with Web Audio (no files). The `AudioContext` is created inside the first pointer/key/touch gesture.
- Squish on grab, and on release scaled by pull distance.
- Wet "thwack" when the knife breaks through (at the world swap in the knife animation).
- Bloop when a piece lands (peak fall speed) or is bumped sideways (`listenForImpacts`). It is rate-limited globally (70 ms) and per piece (220 ms), and muted while grabbing or cutting.
- ¼ speed lowers the pitch. A **Sound** checkbox is added to the panel (`.checks` now wraps).
- `window.__melon.sfx` is exposed for automated checks.

## Copy and credit (Step 6)
- Tab title "Rainbow Jelly", with an inline rainbow SVG favicon (also removes the original's favicon 404).
- Masthead: "MATERIAL STUDIES / NO. 010", "Rainbow / Jelly.", subtitle "Six flavours. / One wobble. / Zero regrets."
- Panel: "THE SPECIMEN — fig. 10".
- "Inside the experiment" rewritten (layers, stiffness tiers, stripes surviving cuts, bubbles, embossed handle, light, sound).
- Footer credit: "Remixed from Melon Jelly by Abraham John."
- Hint, toast and canvas aria-label wording changed from slice/flesh/rind to bar/stripe.

## Verification (Step 7, Chrome + WebGPU via Playwright)
- No console errors or failed requests on load.
- 9 pieces after repeated cuts: 60 fps. ¼ speed: 60 fps, slowed motion and lowered pitch.
- Reset restores the full bar (1 piece, 570 particles, volume 99.8%).
- Firmness / damping at 0/0, 0/1, 1/0, 1/1 with nudge + hard pull: positions stay finite and volume stays within 0.3%.
- 390 × 844 layout has no horizontal scroll, and a CDP touch drag grabs the jelly.
