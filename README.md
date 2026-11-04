# Polyplates

A plate-tectonics simulator for generating Earth-like worlds, built as a single HTML page.
Open `index.html` in a browser (it loads three.js and Delaunator from public CDNs).

- Model description: [`docs/lithosphere_plate_model.tex`](docs/lithosphere_plate_model.tex)
- Roadmap and notes: [`PLAN.md`](PLAN.md)

## Current state (v0.2)

Milestones 0 and 1 of the plan are done:
- Each plate is its own triangulated mesh, moving on a prescribed rigid rotation.
- Every column carries the state of the model (crustal volume, reference densities, plate thickness L, …);
  elevation comes from local isostasy and L thickens with age.
- Plate boundaries are classified every step from the signed gap and relative velocity
  (gap, spreading, transform, convergent, overlap) and drawn in colour.
- Gaps between plates are filled with new oceanic plate, so ridges and young sea floor emerge
  wherever plates separate. Overlaps are detected but not yet resolved; subduction is milestone 2.

## Using it

Controls: drag to orbit, scroll or pinch to zoom. Time step and relief act immediately;
vertex count, continental area, plate count and seed apply on Reset.
The panel reports model time, number of columns, land area, the fraction of the sphere uncovered
or covered twice, and boundary length by type.
