# Worked example: Couette annulus, overset, two grids + TIOGA → OpenFOAM overset

The minimal overset / Chimera demo. Two grids — an outer annular background and an inner cylinder body-fitted patch that rotates relative to the background. Analytic Couette reference closes the loop.

This example is v0.1.0 and TIOGA-path is research-grade; see the caveats below.

## Spec summary

- **Target method**: Overset (Chimera, two grids + TIOGA connectivity)
- **Target solver**: OpenFOAM overset (`overPimpleDyMFoam` or equivalent)
- **Geometry**: 2D annulus, inner radius $r_i = 0.5$, outer radius $r_o = 1.0$; inner cylinder rotates at $\omega$, outer wall stationary
- **Grids**: outer background = annular sector with structured hex, inner = body-fitted ring around the rotating cylinder; overlap region 0.1 wide
- **Flow regime**: $Re = \omega r_i (r_o - r_i) / \nu = 10$ (laminar Couette)
- **Resolution levels**: cells per radial unit ∈ {32, 64, 128} (L0/L1/L2) on both grids
- **Quality envelope**:
  - max non-orthogonality < 70° on both grids (curvilinear)
  - overlap orphan cells = 0
  - overlap mass-imbalance < 0.5%
- **Reference**: analytic Couette profile $u_\theta(r) = A r + B/r$ with $A, B$ from BCs; closed-form solution available
- **Headline hypothesis**: L2 error of $u_\theta(r)$ on L2 < 1%; observed order in [1.5, 2.5]; zero orphan cells across all three levels
- **Success criterion**: hypothesis confirmed; mass-conservation across overlap < 0.5%

## Why this example

- Couette has a closed-form analytic reference; no numerical-ref uncertainty
- Minimal two-grid setup; exposes TIOGA connectivity without distracting geometry
- Tests the failure mode the overset community cares most about (orphan cells, overlap mass-conservation)
- Single workstation runtime

## Caveats — this is the riskiest v0.1.0 example

- **TIOGA is research-grade OSS**. Build path, example coverage, and documentation are thinner than Gmsh or OpenFOAM proper. The scaffold pins exact commits and documents the build steps in `mesh_run/README.md`.
- **OpenFOAM overset on ARM** is not as battle-tested as core OpenFOAM. The scaffold documents which OpenFOAM build (and version) was used.
- **An OpenFOAM-native overset path** exists (`oversetMesh` + `oversetGAMGSolver`) and is the recommended primary; TIOGA is the cross-validation alternative.

If the user wants a less risky overset entry point, the scaffold falls back to OpenFOAM-native overset only; the TIOGA path is opt-in via a spec flag.

## Files produced by mesh-scaffold

```
mesh_run/
├── geometry.py     # build123d annular sectors for the two grids
├── mesh.py         # Two Gmsh runs (one per grid) at L0/L1/L2; TIOGA connectivity call (optional)
├── quality.py      # checkMesh on each grid; TIOGA orphan-fringe report
├── verify.py       # Run overset solver, extract u_theta(r), compare analytic, compute GCI
├── export.py       # polyMesh + connectivity tables
├── run.py
└── outputs/
    ├── grid_bg_L0/, grid_bg_L1/, grid_bg_L2/   # background grid at three levels
    ├── grid_inner_L0/, grid_inner_L1/, grid_inner_L2/  # inner grid at three levels
    ├── tioga_connectivity_L*.txt
    ├── quality.csv
    ├── verification.json
    └── plots/
        ├── overset_layout.png
        ├── orphan_fringe_map.png
        ├── u_theta_profile_L2.png
        └── gci_plot.png
```

## Expected `verification.json` excerpt

```json
{
  "target_method": "Overset",
  "target_solver": "OpenFOAM:overPimpleDyMFoam",
  "tool_versions": {"openfoam": "v2406", "tioga": "git-<sha>", "gmsh": "4.12.2"},
  "quality_envelope": {
    "max_non_orthogonality_deg": {"threshold": 70, "measured": 22.4, "passes": true}
  },
  "overset_audit": {
    "orphan_cells": 0,
    "fringe_cells": 128,
    "overlap_mass_imbalance": 0.0021
  },
  "grid_convergence": {
    "levels": ["L0", "L1", "L2"],
    "refinement_ratio": 2.0,
    "quantity": "u_theta_L2_error",
    "values": [2.4e-2, 6.7e-3, 1.9e-3],
    "observed_order": 1.85,
    "expected_order": 2.0,
    "gci_fine": 0.039,
    "asymptotic": true
  },
  "benchmark_comparison": {
    "reference": "analytic Couette u_theta = A r + B/r",
    "quantity": "L2_error_u_theta",
    "computed": 1.9e-3,
    "rel_error": 1.9e-3,
    "passes": true
  },
  "success_criteria_met": true
}
```

## Common ways this example fails

- Overlap too thin → TIOGA reports orphan cells. Recommended overlap width: at least 3 cells in each grid.
- Inner-grid rotation not transferred to OpenFOAM dynamic mesh dictionary → grid moves visually but the solver sees a stationary BC.
- Reporting overall mass-conservation only — overlap-local conservation is the real diagnostic; the scaffold computes it explicitly.
- TIOGA build pointing at a different MPI than OpenFOAM → silent connectivity failures.

## v0.2.0 roadmap for this example

- A second worked overset example with relative motion (rotor-in-box) to expose dynamic-overset issues
- Cross-validation between OpenFOAM-native overset and TIOGA-driven connectivity on the same geometry
- Single-workstation runtime budget at L2 sized for the M5 Pro
