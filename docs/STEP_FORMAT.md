# STEP format notes

## What is STEP?

STEP (STandard for the Exchange of Product model data) is the **ISO 10303** standard. The file we care about uses the text encoding **ISO 10303-21**, with the extensions `.step`, `.stp` and `.p21`.

## Common schemas (application protocols)

| Schema | Role |
|--------|------|
| AP203 | Geometry and configuration of parts (the oldest, very common) |
| AP214 | Adds colours and materials (automotive) |
| AP242 | Recent version, merges and extends the previous ones (PMI, GD&T tolerances) |

The schema is declared in the file header, for example `FILE_SCHEMA(('AP214_IS'));`.

## File structure

A file has two sections:

```
ISO-10303-21;
HEADER;
FILE_DESCRIPTION(...);
FILE_NAME(...);
FILE_SCHEMA(('AP214_IS'));
ENDSEC;
DATA;
#1 = CARTESIAN_POINT('', (0.0, 0.0, 0.0));
#2 = DIRECTION('', (0.0, 0.0, 1.0));
...
ENDSEC;
END-ISO-10303-21;
```

- Each line of `DATA` is an **entity**: `#id = ENTITY_NAME(parameters);`
- Parameters can reference other entities with `#id`.
- Entities therefore form a **graph** that must be followed to recover the geometry.
- Special values: `$` means not set, `*` means derived. They are different and must both be kept.
- Strings use `''` for an apostrophe, and `\X2\...\X0\` for Unicode characters.
- Real numbers keep their written form (`1.0E-3`); rounding them on read would lose information.

## Useful geometry entities

| Entity | Meaning |
|--------|---------|
| `CARTESIAN_POINT` | 3D point (2D points also exist: parametric coordinates on a surface) |
| `DIRECTION` | Direction vector |
| `VERTEX_POINT` | Topological vertex |
| `LINE`, `CIRCLE`, `ELLIPSE`, `B_SPLINE_CURVE_WITH_KNOTS` | Curves |
| `PLANE`, `CYLINDRICAL_SURFACE`, `CONICAL_SURFACE`, `SPHERICAL_SURFACE`, `B_SPLINE_SURFACE_WITH_KNOTS` | Surfaces |
| `EDGE_CURVE` | Edge (curve + 2 vertices) |
| `ORIENTED_EDGE` | Edge used in a loop, with an orientation |
| `EDGE_LOOP` | Loop of edges bounding a face |
| `FACE_BOUND`, `FACE_OUTER_BOUND` | Boundary of a face (hole or outer contour) |
| `ADVANCED_FACE` | Face (surface + boundaries) |
| `CLOSED_SHELL`, `OPEN_SHELL` | Closed or open shell (set of faces) |
| `MANIFOLD_SOLID_BREP` | **Solid** body: this is what we need to compute a volume |
| `SHAPE_REPRESENTATION` | Container of the geometric representation |
| `SI_UNIT`, `LENGTH_UNIT` | Units used |

Other entities relevant to the business data:

- `PRODUCT`, `PRODUCT_DEFINITION`: part name and reference
- `MATERIAL_DESIGNATION`, `PROPERTY_DEFINITION`: material, if present
- `COLOUR_RGB`, `STYLED_ITEM`: colours

## Tolerances and PMI (not verified on real files)

Tolerances are **mostly on the 2D drawing, not in the STEP file**. Three sources exist, and they differ in reliability:

1. **Drawing (PDF):** dimensional tolerances (for example `Ø8 H7`), general tolerance block (for example ISO 2768 class in the title block), surface finishes (`Ra`), and notes. This is the main source for most shops.
2. **STEP, semantic PMI (AP242):** GD&T tolerances, datums and some dimensional tolerances can be stored as entities (`DIMENSIONAL_CHARACTERISTIC_REPRESENTATION`, `GEOMETRIC_TOLERANCE`, `DATUM`, and so on). This exists only if the exporter wrote it and the option was enabled.
3. **STEP, graphical PMI:** the annotations are written only as drawn lines and text, with no meaning for software. This is hard to use and is not reliable for quoting.

AP203 and AP214 files usually carry no semantic tolerances. The reader must therefore report **which** tolerances were found and from which source, and must never assume that a missing tolerance means none was specified.

## Material and density (not verified on real files)

- **Material** is present in AP214 and AP242 files when the exporter fills it in. It is linked to the part through `PRODUCT_DEFINITION` and `PROPERTY_DEFINITION` entities.
- **Density** is a property, not a fixed attribute of the part. It is not always exported. A likely chain is:
  `PRODUCT_DEFINITION_SHAPE` → `PROPERTY_DEFINITION` (name `'density'`) → `PROPERTY_DEFINITION_REPRESENTATION` → `REPRESENTATION` → `MEASURE_REPRESENTATION_ITEM(MASS_DENSITY_MEASURE(value))` plus a unit.
- Exact entity and property names, and whether common CAD tools export them, have **not been verified**. The method is to inspect real files and look at the `MATERIAL_*`, `PROPERTY_*` and `*MEASURE*` entities, and to check the EXPRESS schema.
- **Mass and volume** are generally not stored as fixed values. They are computed from the geometry.

## Volume calculation

Two methods:

1. **On the B-rep** (exact faces): integrate over the surfaces. Precise, but needs a geometry kernel (OpenCASCADE).
2. **On a triangulated mesh** (divergence theorem): for each triangle, add the signed volume of the tetrahedron formed with the origin:

   `V = Σ (a · (b × c)) / 6`

   Simple to implement, accurate if the mesh is closed and consistently oriented, approximate on curved surfaces (depends on mesh density).

## Known pitfalls

- **Units:** the file may be in mm, cm or inches. Read `SI_UNIT` and `CONVERSION_BASED_UNIT`.
- **Non-closed solid:** an `OPEN_SHELL` or a badly oriented shell gives a wrong or negative volume.
- **Several solids** in one file: each must be handled separately.
- **Nested entities:** the same object can be reached through several levels of references (product → definition → representation → solid).
- **Schema versions:** AP203 and AP242 do not have exactly the same entities.
- **2D points:** `CARTESIAN_POINT` is also used for parametric coordinates on surfaces. These must be excluded from any spatial calculation (for example, the bounding box).

## References

- ISO 10303-21 (text encoding), ISO 10303-203 / 214 / 242 (schemas), ISO 10303-41 (measures)
- OpenCASCADE documentation (STEP reader and mass properties)
