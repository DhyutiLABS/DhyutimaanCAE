# Worked example: 2D Poisson MMS, Gmsh → FEniCSx

A clean first-contact example. Tests the whole pipeline end-to-end on a problem with a perfect reference (manufactured solution).

## Spec summary

- **Target method**: FEM (conforming, $P_1$ triangles)
- **Target solver**: FEniCSx
- **Geometry**: unit square $[0,1]^2$
- **PDE**: $-\nabla^2 u = f$ with $u = 0$ on $\partial\Omega$
- **MMS**: $u(x,y) = \sin(\pi x)\sin(\pi y) \Rightarrow f(x,y) = 2\pi^2 \sin(\pi x)\sin(\pi y)$
- **Resolution levels**: $h \in \{1/16, 1/32, 1/64\}$ (L0, L1, L2)
- **Quality envelope**:
  - min scaled Jacobian > 0.1
  - aspect ratio < 50
- **Headline hypothesis**: observed order of convergence on L2 error in $[1.7, 2.3]$ across L0/L1/L2 for $P_1$
- **Success criterion**: hypothesis confirmed AND L2 error on L2 < $10^{-3}$

## Why this example

- No boundary layer, no overset, no cut-cell — the cleanest possible scaffold demo
- Perfect ground truth via MMS
- Exercises the Gmsh → meshio → FEniCSx path that is the workhorse for FEM-target work
- Runs comfortably on a single workstation; full pipeline in <60s on M-series

## Files produced by mesh-scaffold

```
mesh_run/
├── geometry.py    # Gmsh OCC unit square + boundary physical group
├── mesh.py        # Three runs at h = 1/16, 1/32, 1/64
├── quality.py     # SICN, aspect ratio per cell
├── verify.py      # FEniCSx solve, compute L2 error vs sin*sin, fit observed order
├── export.py      # meshio .msh → .xdmf
├── run.py
└── outputs/
    ├── mesh_L0.msh, mesh_L1.msh, mesh_L2.msh
    ├── mesh_L0.xdmf, mesh_L1.xdmf, mesh_L2.xdmf
    ├── quality.csv
    ├── verification.json
    └── plots/
        ├── quality_hist_minSICN.png
        ├── solution_L2.png
        ├── error_L2.png
        └── convergence_plot.png
```

## Expected `verification.json` excerpt

```json
{
  "target_method": "FEM",
  "target_solver": "FEniCSx",
  "tool_versions": {"gmsh": "4.12.2", "meshio": "5.3.5", "fenicsx": "0.8.0"},
  "quality_envelope": {
    "min_scaled_jacobian": {"threshold": 0.1, "measured": 0.62, "passes": true},
    "max_aspect_ratio": {"threshold": 50, "measured": 4.1, "passes": true}
  },
  "grid_convergence": {
    "levels": ["L0", "L1", "L2"],
    "refinement_ratio": 2.0,
    "quantity": "L2_error",
    "values": [3.9e-3, 9.8e-4, 2.5e-4],
    "observed_order": 1.99,
    "expected_order": 2.0,
    "asymptotic": true
  },
  "hypotheses": [
    {"id": 1, "statement": "observed order in [1.7, 2.3]", "result": "confirmed", "evidence": "1.99"}
  ],
  "success_criteria_met": true
}
```

## Common ways this example fails

- Forgetting to assign a Gmsh physical group on the boundary → FEniCSx can't apply Dirichlet BC → MMS error doesn't converge.
- Using the built-in Gmsh kernel instead of OCC → boundary recovery on the square is fine but breaks for curved geometry; using OCC here keeps the recipe portable to harder problems.
- Choosing $P_2$ elements but reporting $P_1$ expected order → observed order looks anomalously high.
