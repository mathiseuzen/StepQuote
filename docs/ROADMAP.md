# Roadmap

## Phase 0: scoping (in progress)

- [x] Define the idea and scope ([IDEA.md](IDEA.md))
- [x] Document the STEP format ([STEP_FORMAT.md](STEP_FORMAT.md))
- [x] Propose an architecture and the planned functions ([ARCHITECTURE.md](ARCHITECTURE.md))
- [x] Write the documentation in English
- [ ] Collect sample STEP files from real parts (CATIA, SolidWorks, FreeCAD, Creo)
- [ ] Collect the matching 2D drawings and past quotes, with their data rights checked
- [ ] Verify the density and material entities on real files

## Phase 1: reading

- [ ] Open a STEP file and detect the header and schema (AP203 / AP214 / AP242)
- [ ] List the entities present (count per type)
- [ ] Extract the metadata (part name, units)
- [ ] Handle file errors (empty file, invalid syntax)
- [ ] Check that reading reproduces the file without loss

## Phase 1b: data boundary (before any external call)

- [ ] Keep the drawing, notes, request text and quantity as optional inputs: the pipeline runs on STEP alone

- [ ] Build a synthetic test corpus with planted identifiers, to measure leaks (no real data)
- [ ] Write the local redaction rules (`redact_note`): identifiers, names, numbers, dates
- [ ] Build the local dictionary of customers, companies, staff and internal codes
- [ ] Evaluate local entity detection (spaCy, Presidio) on the corpus
- [ ] Implement the residual check (fail closed) and the preview with approval
- [ ] Define the fixed output schema for translated notes, with a validation step
- [ ] Add the audit log of every external call
- [ ] Decide whether an external LLM is allowed at all, with the company, and check the provider's terms
- [ ] Evaluate a local language model as the default option

## Phase 2: extraction of part data

- [ ] List the solids in the file
- [ ] Count faces, edges and vertices per solid
- [ ] Extract the bounding box, excluding 2D parametric points

## Phase 3: calculations

- [ ] Volume of a solid (validated on a reference box)
- [ ] Surface area
- [ ] Centre of gravity
- [ ] Mass from a density
- [ ] Unit conversion (mm, cm, inches)

## Phase 4: outputs

- [ ] Clear console output
- [ ] JSON export
- [ ] CSV export to process several files at once

## Phase 5: process inference

- [ ] Recognise basic features: holes, pockets, slots, flat faces
- [ ] Classify the part by likely process (machining, sheet metal, turning, additive, moulding)
- [ ] Write the business rules for one process first (machining of holes and pockets)
- [ ] Read tolerances and surface finishes from the drawing (main source)
- [ ] Read semantic GD&T from AP242 STEP files, only when present, and report the source of each value

## Phase 6: cost estimation

- [ ] Define the cost parameters (shop rate, material price, time per operation), editable in a configuration file
- [ ] Estimate the cost per process, with the detail of every assumption
- [ ] Compare estimates with past quotes and measure the error

## Phase 7: learning

- [ ] Build the quote dataset from past quotes (clean, with dates and quantities)
- [ ] Train a baseline (linear regression) and then gradient boosting
- [ ] Calibrate the cost parameters from the data
- [ ] Measure the error on parts never seen during training

## Phase 8: comfort (later)

- [ ] Simple graphical interface (drag and drop import)
- [ ] 3D display of the part
- [ ] Material library with densities
- [ ] Automated tests on all reference files

## Open decisions

1. Geometry library: OpenCASCADE or own calculation for geometry?
2. Interface: command line first, then graphical interface.
3. Format of the quote dataset.
4. Drawing reading: manual entry first, or automatic reading (PDF extraction)?
