# Worked example: Lid-driven cavity, FVM, blockMesh → OpenFOAM

The canonical FVM benchmark. Structured hex grid, no STL ingestion, no BL — exercises the OpenFOAM `checkMesh` envelope and the Ghia et al. (1982) reference.

## Spec summary

- **Target method**: FVM hex (structured)
- **Target solver**: OpenFOAM (`icoFoam` for incompressible Newtonian, $Re=100$)
- **Geometry**: unit square cavity, lid moves at $u=1$, all other walls no-slip
- **Resolution levels**: $N \in \{32^2, 64^2, 128^2\}$ uniform hex grids
- **Quality envelope** (OpenFOAM `checkMesh`):
  - max non-orthogonality < 65°
  - max skewness < 3
  - max aspect ratio < 100
- **Headline hypothesis**: centerline u-velocity at $y \in \{0.0625, 0.0703, ..., 0.9609\}$ (16 Ghia probe points) matches Ghia et al. (1982) within 2% L2 on the L2 grid; GCI on the centerline L2 error across L0/L1/L2 < 5% with observed order in [1.5, 2.5]
- **Success criterion**: hypothesis confirmed; all quality metrics inside envelope

## Why this example

- The textbook FVM benchmark; reference data published in Ghia, Ghia, Shin (1982), J. Comp. Phys. 48, 387-411
- Structured hex via `blockMesh` is the cleanest possible OpenFOAM mesh — exercises the pipeline without `snappyHexMesh` complexity
- Three uniform refinements give a clean GCI computation
- Runs comfortably on a single workstation in ~5 min total

## Files produced by mesh-scaffold

```
mesh_run/
├── geometry.py     # No-op for blockMesh path; documents the cavity geometry
├── mesh.py         # Generates three blockMesh dicts at N=32/64/128 and runs blockMesh
├── quality.py      # Parses checkMesh stdout for the three levels
├── verify.py       # Runs icoFoam on each level, extracts centerline u, computes GCI, compares Ghia
├── export.py       # No-op (polyMesh is already OpenFOAM-native)
├── run.py
└── outputs/
    ├── L0/, L1/, L2/        # Each contains 0/, constant/, system/ OpenFOAM case structure
    ├── quality.csv
    ├── verification.json
    ├── ghia_reference.csv   # Embedded reference data
    └── plots/
        ├── centerline_u_comparison.png
        ├── checkMesh_summary.png
        └── gci_plot.png
```

## Expected `verification.json` excerpt

```json
{
  "target_method": "FVM",
  "target_solver": "OpenFOAM:icoFoam",
  "tool_versions": {"openfoam": "v2406", "blockMesh": "v2406"},
  "quality_envelope": {
    "max_non_orthogonality_deg": {"threshold": 65, "measured": 0.0, "passes": true},
    "max_skewness": {"threshold": 3, "measured": 0.0, "passes": true},
    "max_aspect_ratio": {"threshold": 100, "measured": 1.0, "passes": true}
  },
  "grid_convergence": {
    "levels": ["L0", "L1", "L2"],
    "refinement_ratio": 2.0,
    "quantity": "centerline_u_L2_error_vs_ghia",
    "values": [0.041, 0.012, 0.0036],
    "observed_order": 1.81,
    "expected_order": 2.0,
    "gci_fine": 0.018,
    "asymptotic": true
  },
  "benchmark_comparison": {
    "reference": "Ghia et al. 1982, Re=100",
    "quantity": "centerline_u_L2",
    "computed": 0.0036,
    "rel_error": 0.0036,
    "passes": true
  },
  "success_criteria_met": true
}
```

(A pure uniform structured cavity grid has skewness and non-orthogonality of 0; the envelope is trivially passed. The real check is the GCI and the Ghia comparison.)

## Apple Silicon / ARM notes

- OpenFOAM on Apple Silicon: v2306+ has official ARM support; the OpenFOAM-for-arm fork also works.
- The scaffold records the OpenFOAM build path in `verification.json` provenance.

## Common ways this example fails

- Wrong OpenFOAM solver (laminar `icoFoam` vs turbulent `pisoFoam`) — Re=100 is laminar; using a turbulence model produces wrong answers.
- Not running to steady state — Ghia compares steady-state centerline; transient snapshots will not match.
- Forgetting to sample at exactly Ghia's $y$ values — interpolation introduces extra error and contaminates the GCI.
- Reporting the wrong reference — Ghia tabulates non-dimensional $u/U_\text{lid}$; the OpenFOAM result is already non-dimensional if you set $U_\text{lid}=1$, but the comparison must be consistent.
