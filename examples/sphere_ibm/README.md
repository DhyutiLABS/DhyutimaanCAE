# Worked example: Sphere in channel, IBM, Cartesian + STL → AMReX

The canonical IBM demo. Cartesian background mesh with a Stanford-bunny-class STL surface immersed in the flow.

## Spec summary

- **Target method**: IBM (immersed boundary, Cartesian background + STL surface)
- **Target solver**: AMReX (incompressible flow app; can substitute custom IBM driver)
- **Geometry**: sphere of radius $R=0.5$ centered in a box $[-4R, 4R]^3$ ($L=4R$ characteristic distance)
- **Flow**: uniform inflow at $u_\infty = 1$, $Re = u_\infty (2R) / \nu = 100$
- **Resolution levels**: Cartesian L0 = 64³, L1 = 128³, L2 = 256³; refinement band of 4 cells around the sphere
- **Quality envelope**:
  - signed-distance error near surface < 1% of $R$
  - refinement-band coverage: 100% of sphere surface within band
  - aspect ratio: 1 (uniform Cartesian) → trivially passes
- **Reference**: $C_d = 1.09$ at $Re=100$ from Mittal et al. (2008), J. Fluid Mech. 601, 95-127, and Johnson & Patel (1999)
- **Headline hypothesis**: $C_d$ on L2 within 5% of $1.09$; GCI on $C_d$ across L0/L1/L2 < 10% with observed order in [1.0, 2.0]
- **Success criterion**: hypothesis confirmed; sphere volume from cell-summation within 2% of analytic $\frac{4}{3}\pi R^3$

## Why this example

- The IBM canonical case; widely referenced for cross-validation
- Exercises the Cartesian + STL pipeline that is the DhyutimaanMesh IBM workhorse
- AMReX is ARM-native and frugal-hardware friendly
- Smaller cell count budget than wall-resolved FVM at the same accuracy

## Files produced by mesh-scaffold

```
mesh_run/
├── geometry.py    # build123d sphere + meshio STL export; analytic signed-distance helper
├── mesh.py        # Three Cartesian + EB-refinement-band configs; AMReX inputs files
├── quality.py     # SDF accuracy near surface, refinement-band coverage check
├── verify.py      # Run AMReX, extract Cd, compute GCI, compare Mittal/Johnson
├── export.py      # AMReX plotfile + post-processed Cd time series
├── run.py
└── outputs/
    ├── sphere.stl
    ├── L0/, L1/, L2/   # AMReX run directories with plotfiles
    ├── quality.csv
    ├── verification.json
    └── plots/
        ├── sdf_accuracy_hist.png
        ├── refinement_band_xy.png
        ├── cd_time_series_L2.png
        └── gci_plot.png
```

## Expected `verification.json` excerpt

```json
{
  "target_method": "IBM",
  "target_solver": "AMReX",
  "tool_versions": {"amrex": "24.05", "build123d": "0.7.0", "meshio": "5.3.5"},
  "geometry_checks": [
    {"name": "sphere_volume", "analytic": 0.5236, "computed": 0.5202, "rel_error": 0.0065, "passes": true}
  ],
  "quality_envelope": {
    "sdf_max_error_pct_R": {"threshold": 1.0, "measured": 0.42, "passes": true},
    "refinement_band_coverage": {"threshold": 1.0, "measured": 1.0, "passes": true}
  },
  "grid_convergence": {
    "levels": ["L0", "L1", "L2"],
    "refinement_ratio": 2.0,
    "quantity": "Cd",
    "values": [1.18, 1.13, 1.10],
    "observed_order": 1.78,
    "expected_order": 2.0,
    "gci_fine": 0.027,
    "asymptotic": true
  },
  "benchmark_comparison": {
    "reference": "Mittal et al. 2008, Cd=1.09 at Re=100",
    "quantity": "Cd",
    "computed": 1.10,
    "rel_error": 0.0092,
    "passes": true
  },
  "success_criteria_met": true
}
```

## Latent failure modes

The mesh-analysis-report skill will flag, on this example:

- BL band thickness in cells across the estimated boundary-layer $\delta$ at $Re=100$. If under 4 cells, flag for higher-Re extension.
- SDF error pattern: spikes near sharp triangulation edges of the STL → recommend remeshing the surface with finer subdivision.
- Cd time-series settling: if oscillating at the final time, run was not converged.

## Indian-context variant

For the agri-engineering trigger, swap the sphere for an agricultural-spray nozzle interior (drip emitter), keeping the IBM machinery identical. The reference becomes a flow-rate vs pressure correlation from a published nozzle experiment (e.g. work from ICAR-CIAE or IARI).

## Common ways this example fails

- AMReX EB definition not exactly matching the STL → mass-flow imbalance.
- Refinement band too thin (band width < 4 cells) → underresolved boundary layer.
- Not running to statistical stationarity → $C_d$ time-average is wrong.
- Using a Cartesian grid with $L < 8R$ → blockage effects contaminate the drag.
