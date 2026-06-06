# Quality Metrics Reference

Mesh quality metrics are defined inconsistently across communities. This doc fixes the conventions DhyutimaanMesh uses, names the alternatives, and gives translation hints.

## OpenFOAM conventions

The OpenFOAM `checkMesh` utility reports:

- **Non-orthogonality (degrees)**: angle between the face-normal vector and the line connecting cell centers across that face. Threshold: typically 65–70° max. Above 75° the implicit non-orthogonal correction starts to dominate.
- **Skewness**: ratio of the offset between the face center and the line of cell centers, to the face center distance. Threshold: typically 4. Above 4 the interpolation error becomes significant.
- **Aspect ratio**: ratio of the longest to the shortest edge in a cell, or longest to shortest of the surrounding face vectors. Threshold: 1000 for FVM (generous because BL prisms can legitimately reach 10⁴ aspect ratio).

DhyutimaanMesh uses these conventions for FVM target methods. The scaffold parses `checkMesh` output directly.

## FEM Jacobian conventions

For FEM solvers, the canonical quality metric is the element Jacobian:

- **Min scaled Jacobian**: minimum over Gauss points of the Jacobian determinant, normalized by element volume. Range: $-\infty$ to 1. Threshold: 0.1 for $P_1$; 0.3 for $P_2$ and higher.
- **Aspect ratio (FEM convention)**: longest-edge-to-altitude ratio for simplex elements. Stricter than the FVM definition; threshold typically 100.
- **Condition number of the element stiffness matrix**: rarely used directly but predictive of solver convergence; computable from element node coordinates.

DhyutimaanMesh uses min scaled Jacobian as the primary FEM gate.

## Gmsh quality fields

Gmsh exposes several quality measures through its API:

- **SICN** (signed inverse condition number): in [0, 1]; 1 is best. The signed variant flags inverted elements (negative values).
- **gamma**: another tet quality metric in [0, 1]; closely related to SICN.
- **eta**: aspect-ratio-style metric in [0, 1].
- **min edge length / max edge length**: raw geometric metrics, useful for diagnostics.

DhyutimaanMesh uses Gmsh's `minSICN` as a backup FEM-side gate when the solver is not in the loop.

## SU2 / dual-mesh conventions

SU2 reports a dual-mesh skewness based on the centroid-to-edge-midpoint offset. Less standardized than OpenFOAM's metric but follows the same intuition. DhyutimaanMesh translates SU2 reports into OpenFOAM-equivalent skewness for cross-tool comparison; document the translation in `quality.csv` headers.

## Boundary-layer-specific metrics

For wall-bounded viscous flow:

- **y+ (yplus)**: non-dimensional first-cell-from-wall distance, $y^+ = y u_\tau / \nu$. Wall-resolved targets: $y^+ < 1$ in the inner layer, $y^+ \approx 30$–300 for wall-function-equipped models.
- **BL layer count**: integer number of prism cells in the BL. Spec declares the target; analysis reports the achieved count distribution.
- **BL growth ratio**: ratio of consecutive layer thicknesses. Spec declares the target (typically 1.1–1.3); analysis reports the achieved distribution.
- **BL coverage**: fraction of wall cells where the requested layer count was achieved without collapse.

## Cross-tool translations

| OpenFOAM `checkMesh` | Gmsh quality | FEM (Jacobian) |
|---|---|---|
| max non-orthogonality | — | (no direct FEM analog; affects solver stencil orthogonality not element quality) |
| skewness | — | (related to the offset between centroid and face center; FEM uses element distortion instead) |
| aspect ratio | (computable from edge lengths) | aspect ratio (similar) |
| — | minSICN | min scaled Jacobian (related but not equivalent) |
| — | gamma | (no direct map) |

When the same mesh passes through both Gmsh's scoring and OpenFOAM's `checkMesh`, expect disagreement on absolute numbers but agreement on which cells are flagged. Cross-distribution check: re-rank the worst 1% of cells by each tool's metric; the intersection should be large.

## Reporting requirements

For every quality metric in spec section 5, the scaffold's `quality.csv` must include:

- min, max, mean, median, std
- 95th, 99th, 99.9th percentile
- ID (or 3D coordinate) of the worst 5 cells

The analysis skill computes these from the CSV and reports the 99.9th percentile in `analysis-report.md`. Reporting only max-and-mean is an anti-pattern flagged in the skill.

## Threshold defaults summary

| Metric | Target method | Default threshold | Stricter recommendation |
|---|---|---|---|
| Non-orthogonality (OpenFOAM) | FVM | < 70° | < 65° for high-order schemes |
| Skewness (OpenFOAM) | FVM | < 4 | < 3 for compressible/transient |
| Aspect ratio | FVM | < 1000 | < 100 in non-BL regions |
| Aspect ratio | FEM | < 100 | < 50 for high-order |
| Min scaled Jacobian | FEM $P_1$ | > 0.1 | > 0.2 |
| Min scaled Jacobian | FEM $P_2+$ | > 0.3 | > 0.5 |
| Min SICN (Gmsh) | FEM | > 0.1 | > 0.3 |
| Hanging-node count | Conforming FEM | 0 | 0 (hard) |
| Orphan cells (overset) | Overset | 0 | 0 (hard) |
| Min EB volume fraction (cut-cell) | Cut-cell | > 0.001 with merging | > 0.01 without merging |
| y+ band coverage | Wall-resolved | > 95% in band | > 99% |
| BL collapse fraction | Wall-bounded | < 1% | < 0.1% |

These are starting points. The spec can tighten or loosen, but every threshold must be stated explicitly — there are no implicit defaults inside the scaffold.

## Cell-count budgets (informational)

Order-of-magnitude bounds for single-workstation runs (frugal-hardware target):

- 2D FEM: up to ~5M cells
- 3D FEM tet: up to ~5M cells
- 3D FVM hex: up to ~20M cells
- IBM Cartesian: up to ~256³ ≈ 16M
- Cut-cell AMReX: up to ~10M (varies with EB density)

For commodity-cluster (8–16 nodes) targets, multiply by ~10. For HPC scales, the bundle works but capturing provenance becomes the bottleneck.
