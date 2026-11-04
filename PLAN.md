# Polyplates implementation plan

Source model: [`docs/lithosphere_plate_model.tex`](docs/lithosphere_plate_model.tex) (provisional).
Section numbers (§) below refer to that document.
Deliverable at every milestone: a single-file `index.html` (three.js and Delaunator from a CDN, everything else inline).
Bit-for-bit reproducibility is not a goal. Initial conditions are seeded, so a seed reproduces the starting world.

The order follows one rule: each milestone ends with something visible, and later milestones replace
prescribed behaviour with emergent behaviour.

| # | Milestone | Spec sections | What becomes visible | Status |
|---|-----------|---------------|----------------------|--------|
| 0 | **Kinematic scaffold** | §2, §3, §4, §6 (isostasy), §12 (L only) | Separate plate meshes on prescribed Euler-pole rotations; per-column state; isostatic elevation; thickening of L with age; gaps open, overlaps pile up | **done** (v0.1) |
| 1 | **Contact geometry and spreading** | §11.1, §11.2 gap case, §11.3, §11.8 | Spatial index over boundary segments on the sphere; signed gap, v_n, v_t per boundary vertex; new strips of young plate fill gaps; boundary edge splitting; coverage metric shown in the panel | **done** (v0.2) |
| 1.5 | **Seams** | §11.2–11.3, §11.8 | Gaps between plates are covered by seam faces (owned by no plate) rebuilt each step; plates grow to the seam midline when it widens; cracks within a plate are filled; mesh display mode | **done** (v0.3) |
| 2 | **Overlap resolution: subduction and welding (kinematic)** | §11.4 initiation and welding, §11.5 (minus forces), §11.6 termination | Columns flagged ς=1 under the overriding plate; slab kept and deleted at d_max; felsic crust scraped and accreted; arc crust and hydration; collisions weld plates. Motions still prescribed | **done** (v0.4) |
| 3 | **Rigid-plate torque balance** | §6 GPE, §11.5 forces, §11.4 friction, §13 (Eqs. bc, drag), §14 rigid limit | Plate angular velocities solved each step from slab pull, ridge push (Γ), basal drag, interface and transform friction. First milestone where motion is emergent. Mantle model stub supplies u^m and Υ | |
| 4 | **Strength profiles** | §7, §8, §9 | Per-column quadrature of F_b, F_d, H_e, T_e (GPU-friendly, one thread per column); display layers for them; closed-form checks of §9 run as a self-test | |
| 5 | **Damage, rifting, splitting** | §10, §11.7 | Damage evolution with healing and diffusion; plume/upwelling damage; plates split along damaged paths and weld at sutures. Plate count becomes emergent | |
| 6 | **Deformable plates** | §9, §13, §14 deformable | Elasto-viscoplastic thin-sheet solve on each plate mesh (P1 finite elements, Picard iteration, probably WebGPU). Distributed deformation, mountain belts thicken crust via A_v | |
| 7 | **Game-facing outputs** | — | Planet radius, ocean volume (sea level from conserved water rather than fixed z=0), erosion and sediment transfer, export of heightmaps and plate maps | |

## Notes on the spec found while building v0.1

- **Mode indicator range (Eq. chi).** With the invariant of Eq. invariant, `T_kk / (√3 |T|)` spans
  [−2/3, 2/3], not [−1, 1]: the extreme is isotropic horizontal stress, where |T| = √3·|T₁₁| and T_kk = 2T₁₁.
  So pure thrust (β_c) and pure normal faulting (β_e) are never reached. Proposed fix:
  `χ = √3 T_kk / (2|T|)`.
- **Isostatic calibration.** With the reference column at a ridge (L = L₀, e = −2.5 km), a 35 km felsic
  crust (2700 kg/m³) over 180 km of depleted mantle stands about +2.5 to +3 km. v0.1 seeds continents
  with a whole-crust mean density of 2850 kg/m³ (felsic upper over mafic lower crust), crust 18–34 km
  thick from coast to interior, and L 90–170 km, which puts interiors near +1 km and margins on shelves.
  This is an initial condition, not a model change, but it suggests the anchor-density table and the
  "crustal density = rock type" proxy will need a two-layer crust or a calibrated mean.
- **Sea level.** The spec fixes sea level at z = 0. For generated worlds, land fraction is a key output,
  so sea level should probably come from a conserved ocean volume (milestone 7, possibly earlier).

## v0.4 (milestone 2) subduction, collision and welding

What is done:
- **Overlap resolution (§11.4–11.5).** Every step, every surface column that lies inside another plate (a
  boundary column only once it is more than g_tol inside) is compared with the overriding column u at
  that point, using the column buoyancy B of Eq. buoyancy:
  - B_v < B_u and B_v < B_crit → **subduction**: ς = 1, state frozen, kept as slab under that plate;
  - B_v ≥ B_crit and B_u ≥ B_crit → **collision**: v's crust is spread over overriding columns within
    d_orog = 200 km (crustal thickening), and v is consumed;
  - otherwise u is the one to go under, which happens when its own plate is processed.
  Subduction polarity therefore follows buoyancy locally: older ocean goes under younger ocean, ocean under
  continent, and continents never subduct.
- **Scraping and accretion.** Crust with ρ_0,c < ρ_scr is scraped off a subducting column: f_acc onto the
  overriding plate and the rest onto the subducting plate's edge, each spread area-weighted over
  columns within d_arc. ρ_scr is raised from the spec's 2850 to 2900 kg/m³, because v0.1 seeds continents at
  exactly 2850 (a whole-crust mean), and "ρ < ρ_scr" would never scrape them.
- **Arc processes.** Each subducted column adds arc crust q_arc·A_v (ρ_arc) and hydration k_w·w_v·A_v to
  overriding columns within d_arc, area-weighted. This event-based form is equivalent to the spec's rates
  (thickness rate q_arc·v_c/d_arc). Hydration is currently zero in practice, because new plate has w^o = 0
  and nothing else hydrates sea floor.
- **Slabs.** Subducted triangles move from the plate's surface list to a slab list; they ride with their
  plate. Each slab column's map distance d_v from the trench is recomputed every step as its distance (in
  the plate frame) to the nearest surface boundary vertex of its plate, and the column is deleted beyond
  d_max = z_slab / sin θ_d ≈ 933 km. Slabs are drawn as a red overlay through the overriding plate.
- **Trench class.** Boundary edges shared with slab triangles, and the facing boundary of the overriding
  plate, are classed as trench.
- **Welding (§11.4, §11.6).** Collisions between a pair of plates are counted; after 15 consumed columns the
  plates weld. The merged plate keeps the larger plate's frame and turns with the area-weighted mean of
  the two angular velocities (a kinematic stand-in for the force balance of milestone 3). Slabs of either
  plate under the other break off, and boundary columns along the suture get D = 1.
- **Bookkeeping.** When triangles are removed, surviving corner columns keep their h_c; in a collision, the
  crust of their lost dual area goes to the overriding belt, while in subduction it goes down with the slab.
  Surface pieces smaller than 1.5 mean column areas, stranded by removal, are dropped and their crust is
  handed to the nearest surviving column.

Fixed in v0.4.1 (slab patches far from any visible plate edge):
- **Subduction polarity was decided column by column**, so where two oceanic plates of similar age met, the
  choice flipped back and forth along one boundary. The plates interlocked, leaving slab patches and islands
  inside each other: 80 % (seed 1) and 50 % (seed 2) of slab columns had slab of the opposite polarity within
  3 spacings. Polarity is now coherent over a radius of 3 s, as the spec intends ("along the contiguous stretch",
  "overlap with slab" persists):
  - if slab between the two plates already exists nearby, the plate already going under keeps going under;
  - otherwise the plate with lower mean buoyancy in that radius goes under;
  - near-ties (|ΔB| < 0.5×10⁶ kg/m²) are broken the same way from both sides;
  - if the plate that should go under is buoyant and the other side can sink, the other side goes under
    (subduction flip).
  Local mixing fell to 20 %, 9 % and 5.5 % (seeds 1–3). About 80 % of what remains is next to continental
  crust, where patchy margin buoyancy produces legitimate but patchy flips.
- **Welding left third-party slabs pointing at the absorbed plate.** They are now re-pointed to the merged plate.
- **Slab pieces cut off from any trench** are deleted (break-off).
- **The slab overlay drew the unfolded slab.** A column at down-dip distance d was drawn d from the trench,
  up to 933 km into the overriding plate. It is now drawn at d·cos θ_d from the trench point it hangs from
  (≤ 660 km), coloured red at the trench to violet at depth. After these fixes 95–99 % of slab columns hang
  from a trench within 2 spacings of the edge of the plate they are under, and 84–93 % of footprints lie
  under that plate; the rest belong to overriding plates that have since moved away.

Fixed in v0.4.2: the slab overlay is drawn without a depth test (so it shows through the overriding plate),
which also drew slabs on the far side of the globe through the planet. Those far-side strips were the
"slab patches in plate interiors" first reported. The overlay now discards fragments on the hemisphere
facing away from the camera. (The v0.4.1 polarity and slab fixes were real problems, but not this one.)

Fixed in v0.4.3: welding stacked the two meshes without stitching them, leaving holes along the suture and
overlaps where the parts had interpenetrated (up to 75 folded boundary vertices and 13 boundary loops on one
merged plate). A welded plate is now re-triangulated: columns of the two parts within 0.3 s are merged; all
surface columns get a fresh spherical Delaunay triangulation, keeping short triangles inside the old union or
in open space no other plate covers; within 2.5 s of the suture V_c is conserved (so overlapped crust stacks
into a thickened suture), elsewhere h_c is preserved and then rescaled so the plate's total crust is exactly
unchanged; columns within 1.5 s of the other part get D = 1. Every merged plate tested has one boundary loop,
no folds, no duplicate or inverted triangles.

Added outside the spec (to revisit):
- **Crustal flow as a stand-in for gravitational collapse.** Without deformable plates, collision and arc
  growth thickened single columns without limit (196 km crust and 24 km elevations in testing). Crust
  thicker than h_flow = 45 km now diffuses to neighbouring columns of the same plate, conserving volume,
  with diffusivity ∝ excess thickness and time scale τ_flow = 5 Myr. Milestone 6 should replace this with
  the physical mechanism (GPE-driven spreading of thick crust in the thin-sheet solve). Reference density
  is not yet mixed by this flow.

Results (11.5k vertices, 0.5 Myr steps, seeds 1–3, 150 Myr): overlap falls from about 27 % of the sphere
(v0.3) to 0.03–0.1 %; uncovered stays at 0.2–0.4 %; the thickest crust is 50–57 km and the highest
elevation about 3.5–4 km; no edge used by two triangles in the same direction, no inverted triangles;
total crustal volume stays within 1 %. A step costs 70–90 ms on the test container.

What was learned:
- **The initial continents force early welding.** Continents are seeded independently of plates, so every
  convergent boundary that cuts a continent is an immediate collision: 12 plates typically weld down to
  5–8 within 25–100 Myr, and with prescribed motions and no splitting (milestone 5) the count can only fall.
  Options: seed continents inside plates, or accept it until splitting exists.
- **Every transfer of crust to "the nearest column" must be spread over an area.** Tiny dual cells at
  trenches turned single-column deposits into 100 km spikes. All transfers are now area-weighted over
  a belt.
- **A bug worth recording:** iterating triangle edges with "skip if a > b" visits each interior edge once,
  but drops half of the boundary edges entirely, since those belong to a single triangle. The crustal flow
  could not drain columns that touched only such edges.

## v0.3 (milestone 1.5) seams

Motivation: in v0.2 a gap counted as real only once it was wider than Δx_acc, so cracks narrower than that
stayed open indefinitely. The fix inverts the logic: the gap is always triangulated, and Δx_acc only decides
when to add vertices.

What is done:
- **Seams.** A seam is the plate-boundary zone itself: triangles whose corners belong to two or three plates.
  They carry no state, have no strength and add no area to any column; they are drawn with colours taken
  from their corner columns, so a ridge reads as continuous young sea floor.
- **Rebuilt every step, not stored.** All boundary vertices of all plates are Delaunay-triangulated (two
  stereographic projections, each used only in the hemisphere away from its pole). A triangle is kept if
  it is short (edges ≤ 2.2 s), has non-trivial area, lies in uncovered space (centroid, and the midpoints of
  edges that are not existing boundary edges), crosses no plate boundary and contains no boundary vertex.
  Because the seam is recomputed, sliding (transform) boundaries never shear it into slivers, and triple
  junctions get three-plate faces with no special case.
- **Crack fills.** A kept triangle whose three corners are on the same plate is a crack in a rigid plate,
  which can never widen; it is added to the plate permanently as new sea floor (§3 mixing rule). A
  triangle may not reuse a directed edge its plate already has.
- **Midline split rule.** When the gap at a boundary vertex exceeds Δx_acc (≈ 1 s), the plate grows a new
  vertex at the midpoint between that vertex and its facing point on the other plate; both plates aim at
  the same midline, and the seam between the two new copies starts at near-zero width. Where no plate
  faces the vertex within reach, growth falls back to the outward normal.
- **Classification.** "Open gap" now means no facing plate within reach. Otherwise a boundary is named
  by its relative motion (spreading, transform, convergent), or overlap if it lies inside another plate.
- **Display.** A "Mesh" colour mode draws every triangle, with faces and edges in the plate's colour and
  seam faces in amber. The panel reports seam area and the count of crack fills.

Results (seed 1, 11.5k vertices, 0.5 Myr steps): the uncovered fraction stays at 0.03–0.1 % through 150 Myr
(v0.2: about 2 %), seams cover about 1.5–2 % of the sphere, with no edge used by more than two triangles,
no inverted triangles and no folds. A step costs about 50 ms on the test container (v0.2: about 30 ms);
the seam triangulation is the added cost.

What was learned:
- **The seam should belong to no plate.** Assigning each seam face to the plate owning two of its corners would
  make plate meshes non-rigid (bad for the rigid torque balance of milestone 3) and would need continuous
  crust bookkeeping as faces stretch and change owner.
- **A bug worth recording:** the point-in-plate query hashes the query point, so it must be on the unit sphere.
  Passing an unnormalised centroid (a sum of three vertices) silently made almost everything look
  uncovered, which produced duplicate triangles and, downstream, folds and inverted triangles.

## v0.2 (milestone 1) implementation notes

What is done:
- **Growable plate meshes.** Per-plate structure-of-arrays state with capacity doubling; triangles stored
  in a growable index array; directed boundary edges (a→b in triangle order, interior on the left).
- **Contact geometry (§11.1).** Every step, for every boundary vertex: outward normal (sum of adjacent edge
  normals, projected to the tangent plane); host plate, if the vertex lies inside another plate; signed gap
  g to the nearest boundary segment (of the host plate if inside, of any other plate if outside);
  v_n and v_t from the plates' angular velocities. Boundary segments go in a 3-D grid hash with cells
  of 3 s (s = mean vertex spacing). Point-in-plate queries use a flat bucket grid of triangle bounding boxes.
- **Classification (§11.2).** gap (g > g_tol), overlap (g < −g_tol), otherwise contact, split into
  spreading / transform / convergent by the larger of |v_n| and |v_t|. Boundaries are drawn coloured by
  class, and the panel shows boundary length by class.
- **Spreading (§11.3).** A boundary vertex with g > Δx_acc grows a new vertex a distance min(g/2, s)
  along its outward normal, unless that point is already covered by any plate. New vertices closer than
  0.4 s are merged; strip triangles across each boundary edge are kept only if correctly oriented and
  not slivers. Each plate grows independently toward the midline, so the two copies of the midline
  required by the spec arise without a shared insertion step.
- **New sea floor and the mixing rule.** New triangle area is credited to its three vertices as new
  oceanic floor (V_c += h_c^o ΔA, with ρ_c, ρ_m, L, w, D mixed by the §3 rule). A brand-new vertex
  therefore becomes a pure ridge column, and an old boundary column becomes a mixture (a thinned margin
  where the old column was continental).
- **Edge splitting.** Boundary edges longer than 1.6 s are split at their midpoint.
- **Coverage (§11.8).** Every 15 frames, 8 000 fixed probe points count how many plates cover them.
  The panel shows the uncovered and overlapped fractions.
- **Initial plates** now come from a weighted Voronoi partition with noisy edge costs (smooth, curved
  outlines), not from random-frontier growth. Plate speeds are 1–7 cm/yr at the Euler equator.

What was learned:
- **The spec's sign rule for g needs a point-in-plate test.** Deciding inside versus outside from the side
  of the nearest segment fails wherever three plates overlap: a vertex can be inside plate r while
  its nearest boundary belongs to plate q. In testing, this made overlapped boundaries count as gaps
  and extend without limit (12k → 200k vertices in 45 Myr). v0.2 instead finds the host plate
  by containment and measures |g| to that plate's boundary.
- **Growth must not cover existing plate, including the plate's own.** The same runaway occurred when
  strips folded over their own plate; the coverage check on each extension target fixes both cases.
- **V_c = h_c A_v interacts badly with mesh growth.** If strip triangles simply enlarged the dual cells of
  old boundary columns with V_c held fixed, every column would have its crust diluted once as it
  stopped being boundary. Crediting new area as new oceanic floor avoids this artifact and keeps V_c
  exactly conservative during spreading.
- **Correct orientation and an uncovered tip do not prevent folds** (fixed in v0.2.1). Near concave bends a
  strip triangle can be counter-clockwise and reach open space while still lying across a neighbouring
  part of the same plate. The mesh stays manifold and consistently oriented, but it folds: in testing,
  312 boundary vertices sat inside their own plate by 100 Myr, and the flap's edge drew as a boundary
  loop through the plate. Each candidate triangle is now rejected if it crosses any boundary edge of its
  plate (including triangles accepted earlier in the same event) or contains a boundary vertex.
  Fold count is now zero on every seed tested, with no change in the uncovered fraction.
- **A plate must also see gaps against itself** (v0.2.1). When strips grow around a gap and meet, they
  enclose a hole that the "other plate" gap rule never measures. Gaps are now measured against the
  plate's own boundary too, excluding the 4 edges either side of the vertex along its loop, and small
  holes (clockwise loops under 4 s² not covered by another plate) are closed by ear clipping and
  credited as new sea floor.
- **Without subduction, total plate area grows.** Overlap reaches about 20 % of the sphere by 70 Myr at
  the default settings, and the number of columns grows roughly linearly (11.5k → 26k in 160 Myr).
  Milestone 2 removes area at convergent overlaps.

Known limitations (to revisit):
- Remeshing (edge splitting) preserves h_c, not V_c, so it is not yet strictly conservative.
  Interior triangles are never improved (no edge flips or vertex smoothing yet).
- Gap slivers narrower than Δx_acc (about one spacing) stay open, so ridges show as thin orange lines.
- The triangle hash is rebuilt for all plates every step. At 11.5k vertices a step takes about 20–50 ms
  on a slow 4-core container; at 40k vertices about 100 ms. This is the first candidate for the GPU.

## v0.1 implementation notes

- Global spherical Delaunay mesh from jittered Fibonacci points (stereographic projection + Delaunator),
  then triangles are partitioned into plates by weighted random-frontier growth; vertices on shared edges
  are duplicated so each plate is its own mesh (§2).
- Continents grow on the vertex graph to the requested area fraction, independent of plate boundaries.
- Initial seafloor age is distance from the boundaries that the prescribed motions make divergent,
  at a 3 cm/yr half-rate; L = a_L √(κ t).
- Each plate stores a rotation quaternion; body-frame positions never change, so rigid motion is exact.
- Elevation is the Newton root of M(e) = M_ref (§6), warm-started each step.
