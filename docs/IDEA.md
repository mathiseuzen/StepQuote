# Idea and scope

## Problem

Mechanical parts are often stored as STEP files, a neutral exchange format between CAD systems. The file contains the full geometry but cannot be read by a person. To get simple information such as volume, mass or dimensions, you currently need to:

- open a heavy CAD application (SolidWorks, FreeCAD, Fusion, and so on), or
- write a custom script for a complex format.

More important for the business: **quoting a part is manual work**. An engineer reads the 3D model and the 2D drawing, decides how the part would be made, and estimates the cost. This takes time and depends on individual experience.

## Proposed solution

A tool that:

- **imports** a STEP file and, when available, the matching 2D drawing;
- **reads** the structure and geometry of the part without losing data;
- **infers** the likely manufacturing methods from the features of the part;
- **estimates** the cost with a detailed breakdown;
- **learns** from the engineer's corrections and from past quotes.

## Final goal: automated quoting

The purpose is not only to measure a part, but to **estimate its manufacturing cost**. To do this, the tool must work out how the part would be made, from the STEP file:

- **Likely process:**
  - machining: drilling, pockets, slots, flat faces;
  - sheet metal: constant thickness, bends, cut-outs;
  - turning: parts of revolution;
  - additive manufacturing: free-form geometry, thin walls;
  - moulding or injection: draft angles, wall thickness.
- **Manufacturing characteristics:** number and type of holes, pockets, threads and fillets; tolerances and surface finishes, which come mainly from the drawing. Some GD&T tolerances may also be in an AP242 STEP file, but only if the exporter wrote them.
- **Material:** material, density, raw material quantity (bounding box or useful volume).
- **Number of operations:** faces to machine per orientation, number of setups.

Consequence for the reader: **keep everything**. Shape entities (planes, cylinders, fillets), tolerances, materials and manufacturing data must be preserved even when the calculation does not use them yet. An entity that is not understood is kept as is, never discarded.

## Optional inputs and the data boundary

The STEP file is the only required input. The 2D drawing, its written notes, and any request email or text are optional. When they are absent, the tool works from the STEP file and reports what is missing.

Drawings often carry written notes that are not in the STEP file: finishing instructions, heat treatment, deburring, notes in several languages. When they exist, the tool reads them locally. An external LLM can then translate and structure each note into known fields. This step is optional and off by default.

Because the drawings and prices are confidential, the tool keeps everything local, except for short redacted text snippets sent to the LLM when it is enabled. See [Architecture](ARCHITECTURE.md#data-boundary-local-vs-external).

## Users

- Design offices that want a fast quote or cost check.
- Estimators who need consistent quotes across the shop.
- Developers who need part data in their own tools (quoting, ERP, scripts).

## Use cases

1. Import a part and get its **volume** to estimate the material needed.
2. Get the **mass** by choosing a material (steel, aluminium, PLA, and so on).
3. Get the **overall dimensions** of the part (bounding box).
4. Process a **batch** of STEP files and export a CSV table.
5. Produce a **cost estimate** for a part from its STEP file and drawing, with the detail of each cost item.

## Scope

In scope:

- Reading standard STEP files (AP203, AP214, AP242).
- Computing geometric and mass properties of a solid.
- Inferring manufacturing processes from features, using explicit rules.
- Estimating cost from a parameterised model, and learning its parameters from past quotes.

Out of scope at first:

- Modifying or creating parts (this is not a CAD editor).
- Reading native CAD files (SLDPRT, IPT, CATPart).
- Repairing complex geometry defects.
- Full assembly analysis (positions, constraints), unless the need appears later.

## Open questions

- Interface: command line only, or a graphical interface (and which one)?
- Should the part be displayed in 3D?
- What precision is expected for the volume (tolerance)?
- Should units (mm, cm, inches) be handled and converted automatically?
- Which manufacturing processes dominate the parts to quote?
- Which data sources exist (past quotes, drawings, shop rates), and what are the data rights?
