# Polyplates

A plate-tectonics simulator for generating Earth-like worlds, built as a single HTML page.
Open `index.html` in a browser (it loads three.js and Delaunator from public CDNs).

- Model description: [`docs/lithosphere_plate_model.tex`](docs/lithosphere_plate_model.tex)
- Roadmap and notes: [`PLAN.md`](PLAN.md)

## Current state (v0.5)

Milestones 0, 1, 1.5, 2 and 3 of the plan are done:
- Each plate is its own triangulated mesh moving as a rigid rotation. **Its angular velocity is solved every
  step from a torque balance**: slab pull, ridge push (through gravitational potential energy), basal and
  slab drag, friction on subduction interfaces and transforms, and trench suction. The first 20 Myr
  (adjustable) blend in from prescribed motions so that the first slabs can form.
- Slip at plate boundaries damages the boundary columns, which weakens them; cold lithosphere hardly heals.
- Every column carries the state of the model (crustal volume, reference densities, plate thickness L,
  hydration, damage, …). Elevation comes from local isostasy, and L thickens with age.
- Plate boundaries are classified every step from the signed gap and relative velocity
  (gap, spreading, transform, convergent, overlap, trench) and drawn in colour.
- Gaps between plates are covered by seam faces, rebuilt every step and owned by no plate. When a seam
  widens past about one vertex spacing, both plates grow to its midline as new oceanic plate, so ridges and
  young sea floor emerge wherever plates separate. Cracks within a single plate are filled outright.
- Where plates overlap, the less buoyant column subducts and is kept as slab beneath the overriding plate
  until it is 933 km from the trench. Light crust is scraped off and accreted, and subduction builds arc
  crust. Buoyant (continental) columns that collide hand their crust to the overriding plate, which thickens
  it, and plates that keep colliding weld into one. Thick crust slowly spreads out, a stand-in for
  gravitational collapse until plates can deform.
- Emergent result: plates with long attached slabs are fast (up to about 10 cm/yr), and continental plates
  without slabs are slow (under 1 cm/yr).
- Display modes: elevation, plate, plate thickness, crustal thickness, damage, and mesh (every triangle,
  seam faces in amber). Optional plate-velocity arrows.

## Using it

Controls: drag to orbit, scroll or pinch to zoom. Time step and relief act immediately;
vertex count, continental area, plate count, spin-up length and seed apply on Reset.
The panel reports model time, number of columns, land area, the fraction of the sphere uncovered,
covered twice or covered by seam, the number of crack fills, slab columns and welds, rms and fastest
plate speed, and boundary length by type. The red overlay shows retained slabs.
