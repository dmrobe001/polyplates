# Polyplates implementation plan

Source model: [`docs/lithosphere_plate_model.tex`](docs/lithosphere_plate_model.tex) (provisional).
Section numbers (§) below refer to that document.
Deliverable at every milestone: a single-file `index.html` (three.js and Delaunator from a CDN, everything else inline).
Bit-for-bit reproducibility is not a goal. Initial conditions are seeded, so a seed reproduces the starting world.

The order follows one rule: each milestone ends with something visible, and later milestones replace
prescribed behaviour with emergent behaviour.

| # | Milestone | Spec sections | What becomes visible | Status |
|---|-----------|---------------|----------------------|--------|
| 0 | **Kinematic scaffold** | §2, §3, §4, §6 (isostasy), §12 (L only) | Separate plate meshes on prescribed Euler-pole rotations; per-column state; isostatic elevation; thickening of L with age; gaps open, overlaps pile up | done (v0.1) |
| 1 | **Contact geometry and spreading** | §11.1, §11.2 gap case, §11.3, §11.8 | Spatial index over boundary segments on the sphere; signed gap, v_n, v_t per boundary vertex; new strips of young plate fill gaps; boundary edge splitting; coverage metric shown in the panel | |
| 2 | **Overlap resolution: subduction and welding (kinematic)** | §11.4 initiation and welding, §11.5 (minus forces), §11.6 termination | Columns flagged ς=1 under the overriding plate; slab kept and deleted at d_max; felsic crust scraped and accreted; arc crust and hydration; collisions weld plates. Motions still prescribed | |
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

## v0.1 implementation notes

- Global spherical Delaunay mesh from jittered Fibonacci points (stereographic projection + Delaunator),
  then triangles are partitioned into plates by weighted random-frontier growth; vertices on shared edges
  are duplicated so each plate is its own mesh (§2).
- Continents grow on the vertex graph to the requested area fraction, independent of plate boundaries.
- Initial seafloor age is distance from the boundaries that the prescribed motions make divergent,
  at a 3 cm/yr half-rate; L = a_L √(κ t).
- Each plate stores a rotation quaternion; body-frame positions never change, so rigid motion is exact.
- Elevation is the Newton root of M(e) = M_ref (§6), warm-started each step.
