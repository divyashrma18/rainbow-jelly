Build "Rainbow Jelly" — an interactive soft-body jelly experiment as a single self-contained index.html (raw WebGPU + WGSL, vanilla JS, NO libraries, no external assets). It must match the quality of a "material study" lab piece: physically convincing wobble, a knife that cuts it into pieces, and a refractive, glossy jelly look.

## Object
A rectangular jelly bar (approx 4 × 1.4 × 1.6 units, rounded edges, rounded corners radius ~0.15) made of 6 horizontal stacked colour layers, bottom to top: violet, blue, green, yellow, orange, pink-red. Thin milky/opaque "panna cotta" band between every second layer. The handle "@_thecodingdiva_" is softly embossed (raised ~0.03) on the top face — as a visual surface mesh/normal-map effect, not physics geometry.

## Physics — XPBD soft body on a tetrahedral mesh
- Build a volumetric tet mesh procedurally: target ~600 particles / ~2,200 tetrahedra. Generate it from the 2D outline (side profile) extruded in depth so cutting stays simple.
- Fixed 60 Hz step, 10 substeps per frame, accumulator-based so it's framerate-independent.
- Per tet: co-rotational elastic constraint (pull toward best-fit rotated rest shape, finite compliance; must recover inverted tets) + volume-preservation constraint (near-incompressible so stretched jelly necks in).
- Per-layer stiffness: each colour layer has its own firmness multiplier (e.g. top layers softer, bottom layer firmest) so the bar wobbles with visible layered shear.
- Damping only on relative velocity along mesh edges (wobble dies down, but free fall/tumbling is undamped). Gravity on.
- Ground plane contact: project particles out of floor, Coulomb-style friction.
- Piece–piece collision: separate pieces push against each other, never interpenetrate (particle-vs-tet or particle-vs-surface, keep it cheap).
- Global controls: Firmness (0–1, "trembling" → "set") and Internal Damping (0–1, "lively" → "syrupy") scale the compliances live.
- Run the solver in WebGPU compute shaders if feasible; otherwise JS with typed arrays — but it must hold 60 fps with 1–8 pieces on a laptop.

## Tools
1. HAND (default): grab any point (tip, corner, any layer) — ray-pick the surface, attach a soft spring to the patch of nearby particles, bounded reach. Drag to pull. Scroll wheel or a second finger while holding twists it. Full multi-touch support.
2. KNIFE: user drags a stroke across the jelly. On release:
   - The stroke, read on the surface it was drawn over, defines a vertical blade plane.
   - For each piece it crosses: fit the piece's current pose with a best rotation, map the plane back into rest space → a straight line across the piece's convex 2D rest outline. Split the outline (keep rounded corners), rebuild both halves as fresh tet meshes.
   - Animate a 3D steel cleaver knife (blade + dark handle with rivets) coming down along the stroke: its edge first presses a groove into the jelly (push particles), then breaks through — at that moment swap in the new pieces, inheriting position and velocity from the original flesh — and the blade wedges the faces apart on its way to the floor.
   - Each piece is an independent soft body: can be grabbed, stretched, and cut again, down to small sizes (set a sensible minimum).
   - The cut faces must clearly show the rainbow layers in cross-section — this is THE hero moment.

## Visual surface
- The smooth rendered surface, the embossed handle, and a few tiny trapped air bubbles are separate render meshes; each vertex is pinned to a tet via barycentric weights so they stretch and travel with the jelly.
- Layer colour is a function of rest-space height (so it stays correct after deformation and cutting).

## Rendering (WGSL)
- Pass order: floor + bubbles first; then jelly back faces into a thickness buffer (how much jelly each view ray crosses); then front faces.
- Front faces: refract the scene behind by thickness, Beer–Lambert absorption per layer colour, milky scattering in the panna-cotta bands, back-lit glow at thin edges, Fresnel reflections of a procedural soft studio environment, GGX specular highlights (glossy, wet-looking).
- The knife blade is drawn behind the jelly surface so the part buried inside is seen refracted.
- Floor: warm off-white studio paper; soft key-light shadow tinted by the jelly colours + contact occlusion from a top-down height map.
- MSAA or FXAA, devicePixelRatio aware, resize-safe.

## Sound (Web Audio, synthesized — no files)
- Squish/squelch on grab and release, intensity from pull distance.
- Wet "thwack" + slice sound when the knife breaks through.
- Soft bloop when pieces land/collide, scaled by impact speed, rate-limited.
- Mute toggle; audio starts only after first user interaction.

## UI — editorial "material study" layout
- Background: warm light grey/paper. Fonts from Google Fonts: an elegant italic serif display (e.g. Instrument Serif / Cormorant) for the title, a clean grotesk for labels, a mono for numbers.
- Top left: small caps label "MATERIAL STUDIES / NO. 010"; huge italic title "Rainbow / Jelly." on two offset lines; three-line poetic subtitle (write your own, playful, e.g. "Six flavours. / One wobble. / Zero regrets.").
- Top right: pill badge "● WEBGPU · LIVE".
- Right: "THE SPECIMEN — fig. 10" control panel: Tool toggle (Hand / Knife with icons), Variety swatches (Rainbow, Neon, Pastel — swap the layer palette live), Firmness slider, Internal Damping slider, buttons "Give it a nudge" (random impulse) and "Reset", checkboxes "¼ speed" and "Show mesh" (wireframe tets), "Pause" button, sound toggle.
- Bottom left: italic one-line instruction for the active tool, then a live stats row: MASS (≈g), VOLUME (% of rest, summed over every tet), KINETIC (µJ), PIECES — with a tiny footnote on the illustrative scale.
- Collapsible "INSIDE THE EXPERIMENT +" panel bottom-right explaining (in your own words) the tet mesh + XPBD, cutting, surface binding, and the rendering.
- Fully responsive; on mobile the panel collapses into a bottom sheet. Cursor changes per tool (grab hand / crosshair).
- If WebGPU is unavailable, show a tasteful fallback message.

## Process
Work in phases and verify each before moving on:
1. WebGPU setup + procedural tet mesh + XPBD solver + floor, with Show mesh. Confirm stable, no explosions at any slider value.
2. Hand tool (mouse + touch + twist).
3. Surface mesh binding + full jelly shading + shadows.
4. Knife: rest-space split, re-meshing, blade animation, velocity inheritance, repeated cuts, piece collisions.
5. UI, stats, varieties, sound, explainer panel.
6. Polish + performance pass: test 8+ pieces at 60 fps, ¼ speed for slow-mo recording, no console errors.
Open it in the browser after each phase and fix what you see.