# Methods: what we can extract from a STEP file, and why

This document explains why the project starts from the STEP file rather than the drawing, which methods work on a STEP file, how reliable each one is, and what they cannot do.

## 1. Why STEP before the drawing

A STEP file is a **description of the part** in a text format. A drawing is a **picture of that description**, made for a human to read.

| | STEP file | 2D drawing (PDF) |
|---|---|---|
| What it holds | Exact geometry: points, curves, surfaces, solids | Lines, dimensions and notes, as graphics |
| Reading method | Parse text, follow references | Image processing and character recognition (OCR), layout interpretation |
| Determinism | The same file always gives the same numbers | Two readings of a scan can differ |
| Volume | Computed from the geometry | Must be reconstructed from views, which is ambiguous |
| Errors | Syntax errors are detected exactly | Misread digits (3 and 8, 1 and 7), missing views, wrong scale |
| Cost to read | Low: linear in file size, seconds for tens of megabytes | High: per-page image processing, recognition models, manual checks |
| Checkability | Can be verified against a geometry kernel | Mostly verified by a person |

The drawing is still needed for information that the STEP file often lacks: dimensional tolerances, general tolerance blocks, surface finishes, free notes, and sometimes the material and quantity.

## 2. Why reading code is cheaper than OCR

Reading a STEP file is parsing a text format with a fixed grammar. Each line is `#n = ENTITY(...)`, and references are numbers. The parser is simple, and its output can be checked exactly: re-writing the parsed entities must reproduce the original text.

OCR has to solve several problems in sequence:

1. Find the drawing area and its views on the page.
2. Recognise the text, which is small, rotated, sometimes handwritten or scanned at low resolution.
3. Link each number to the right dimension arrow or leader.
4. Interpret symbols such as diameters, tolerance frames and surface finish marks.

Each step has an error rate. Errors compound. A single wrong digit changes a cost. Results therefore need human review, which removes most of the saving. OCR is also more expensive to run and to maintain.

**Rule of thumb:** use the STEP file for anything it contains, and use the drawing only for what is missing, with human confirmation.

## 3. Methods that work on a STEP file

Each method below is listed with its input, its output, and its reliability. Reliability is a statement about the method, not a guarantee for every file.

### 3.1 Syntax reading (parsing)

- **Input:** the text file.
- **Output:** all entities, with their original text kept, so nothing is lost.
- **Check:** writing the parsed entities back must reproduce the original text, and no reference may point to a missing entity.
- **Reliability:** exact for well-formed files. Malformed files are rejected, not guessed.

### 3.2 Solid and body count

- **Input:** the `MANIFOLD_SOLID_BREP` entities.
- **Output:** the number of solid bodies.
- **Reliability:** exact. The count matches OpenCASCADE on the files tested.

### 3.3 Volume

- **Input:** the bodies, with their surfaces.
- **Method:** integration over the faces, done by a geometry kernel (OpenCASCADE). Writing this kernel ourselves would mean handling curved surfaces (B-splines, tori, cones), which is a large project.
- **Reliability:**
  - Planar parts: exact (a 10 × 20 × 30 mm box gives 6000 mm³).
  - Curved parts: the result depends on the integration settings. On one test part, two settings differed by about 0.06 %. The uncertainty must be reported with each volume.

### 3.4 Overlapping bodies

Bodies in a single file can overlap. Adding their volumes counts the overlap twice.

- **Method:** fuse the bodies, then measure the result.
- **Reliability:** correct for the material volume. The fusion itself can fail on very complex models, so it must be checked.

### 3.5 Bounding box

- **Naive method:** take the minimum and maximum of all points. **Wrong:** control points of curved surfaces and unused points can lie outside the part. On a test file, the naive box was 557 × 449 × 413 mm, and the true box was 261 × 119 × 33 mm.
- **Correct method:** compute the box from the geometry (the kernel's bounding box).

### 3.6 Face types and feature signals

Each face has a geometric type. These types are signals about how the part is made.

| Face type | Typical meaning | Manufacturing signal |
|---|---|---|
| Plane | Flat face | Milling, sheet metal, sawing |
| Cylinder | Hole, boss or shaft | Drilling, boring, turning. The radius gives the diameter |
| Torus | Fillet or round | Ball-end or radius cutter, moulding |
| Cone | Chamfer, countersink, drill point | Chamfering, drilling |
| Sphere | Ball end | Turning, free-form machining |
| B-spline | Free-form surface | 5-axis machining, casting, forging, additive |

- **Reliability:** these are signals, not proof. A cylinder can be a hole (material inside) or a boss (material outside). Telling them apart needs the face orientation, which is not yet implemented.

### 3.7 Parts and quantities from the assembly structure

A STEP file can contain one part, several bodies of one part, or an assembly of several parts.

- **Entities used:** `PRODUCT`, `PRODUCT_DEFINITION`, `NEXT_ASSEMBLY_USAGE_OCCURRENCE`.
- **Rule:** a product with no children is a physical piece. Its quantity is the number of occurrences in the assembly, multiplied through the levels.
- **Reliability:** exact for what the file declares. On a generated test assembly and on a public test file, the quantities matched the tree.

### 3.8 Bodies are not pieces

Several solids in one product are one piece. CAD systems create one body per modelling operation, such as "cut-extrude", "fillet" or "boss-extrude". These bodies overlap and are not physical parts.

- **Rule:** count pieces from the product structure, not from the solids.

### 3.9 Piece volume and standalone STEP

- **Method:** each piece's geometry is linked to its product through the assembly document (OpenCASCADE's XCAF), and measured after fusing its bodies.
- **Standalone STEP:** the piece can be written to its own file, in its own coordinate frame. The file is rewritten by OpenCASCADE, so original names, colours and annotations may be lost.
- **Known limit:** this link is only reliable when one file is read per process. Reading two files in one process crashed OpenCASCADE during testing. A batch job must therefore start one process per file.

### 3.10 Units

- The unit is declared in the file (for example `SI_UNIT(.MILLI., .METRE.)`). A file can also declare other units in other contexts, such as inches.
- **Rule:** read the declared unit, check that dimensions are plausible, and report the unit used.

### 3.11 Material and density

- Present only if the exporter wrote them. Often absent.
- The density is stored as a property, linked to the part through a chain of entities. The exact entities should be checked on real files before use.

### 3.12 Tolerances and PMI

- **Drawing:** the main source for dimensional tolerances (for example `Ø8 H7`), the general tolerance block and surface finishes.
- **STEP:** GD&T can be present only in AP242 files, written as semantic PMI, and only if the exporter enabled it. AP203 and AP214 files usually carry none.
- **Rule:** a tolerance missing from the STEP file means "not in the STEP file", never "no tolerance".

## 4. Methods that need a drawing or data

| Need | Method | Cost | Reliability |
|---|---|---|---|
| Tolerances, finishes, notes | Manual entry from the drawing | Low per part, but time-consuming | High, once checked |
| Same, at scale | OCR or vision model on the drawing | High | Lower; needs human review |
| Process choice | Rules on face types and features | Low | Explainable; accuracy must be measured |
| Price | Cost model with parameters from past quotes | Low to run, needs quotes | Only as good as the quotes and the parameters |
| Price from many features | Machine learning on features from the STEP file | Medium; needs labelled data | Only inside the family of parts it was trained on |

## 5. Pitfalls found so far

- Overlapping bodies counted twice when volumes are summed.
- Non-solid items (wires, reference geometry) that carry a volume value with no meaning.
- Two-dimensional points (parametric coordinates on surfaces) used in a spatial calculation.
- Instances of the same piece counted from the geometry instead of the occurrences.
- Internal references (for example `=>[0:1:1:3]`) mistaken for names.
- Mixed units in one file.
- One OpenCASCADE document per process.

## 6. What this means for the project

1. The STEP file gives exact, checkable data for volume, bodies, pieces, quantities and face types. Start there.
2. The drawing supplies what the STEP file lacks. Enter it manually first, and automate it only if the volume of work justifies it.
3. Feature recognition and cost estimation are rules and models built on top of the geometry, each with a measured error.
4. Machine learning helps only when there are enough labelled examples. Without them, rules and a simple cost model are the honest choice.
