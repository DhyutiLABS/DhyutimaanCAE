# DhyutimaanMesh Architecture

## Design principles

1. **Method-aware before tool-aware.** Every spec declares its target discretisation method first. Tool, element type, quality envelope, and export format are downstream of that decision.
2. **Frameworks wrapped, not reimplemented.** Gmsh, cfMesh, snappyHexMesh, MMG, TIOGA, AMReX EB, OpenCASCADE — the scaffold orchestrates, captures provenance, and asserts envelopes.
3. **Falsifiable specs, adversarial analysis.** Every hypothesis carries an intervention, a metric, and a threshold. The analysis skill is suspicious by default.
4. **Cross-distribution evaluation is mandatory.** A mesh that passes one tool's quality scoring is re-checked against a second tool's scoring before being declared robust.
5. **OSS-only.** No commercial mesher in the loop. Tools accepted: see "Tool inventory" below.
6. **One pipeline per domain.** Internal subdomain routing chooses the right backend (Gmsh vs cfMesh vs MMG vs TIOGA vs AMReX EB) based on the spec; the spec → scaffold contract is single.

## Pipeline contract

```
literature-survey-mesh   →  mesh-knowledge-base.md       (markdown)
mesh-problem-spec        →  problem-spec.md              (10 numbered sections)
mesh-scaffold            →  mesh_run/                    (geometry/mesh/quality/verify/export/run.py + outputs/)
                            mesh_run/outputs/quality.csv
                            mesh_run/outputs/verification.json   (schema: docs/verification.schema.json)
mesh-analysis-report     →  analysis-report.md           (11 numbered sections)
```

Section headers and filenames are stable contracts. The downstream skills parse them.

## Tool inventory

### CAD layer

- **CadQuery** — Python-first parametric CAD; emits STEP/STL.
- **build123d** — modern CadQuery alternative; emits STEP/STL.
- **OpenCASCADE / OCCT** — kernel under FreeCAD and Gmsh's OCC kernel.
- **pythonOCC** — Python bindings to OCCT; for direct BREP manipulation.
- **FreeCAD** — GUI-in-the-loop fallback; scriptable in Python.

### Surface and volume meshing

- **Gmsh** — backbone for FEM tet/hex, IBM background grids, surface meshes. Python API mature.
- **Netgen / NGSolve** — alternative tet mesher with NGSolve FEM coupling.
- **TetGen** — Si's classic tet mesher; constrained Delaunay.
- **Triangle** — Shewchuk's 2D Delaunay mesher (for 2D problems and surface previews).
- **cfMesh** — hex-dominant FVM for OpenFOAM; Cartesian-template + boundary-fit.
- **snappyHexMesh** — OpenFOAM's native FVM hex-dominant mesher.
- **blockMesh** — OpenFOAM's structured hex tool for simple block-structured meshes.
- **CGAL** — computational-geometry kernel for robust geometric operations (surface mesh, polyhedral meshing).

### Adaptation and remeshing

- **MMG / mmg3d / mmg2d / mmgs** — anisotropic adaptive remeshing on tet / tri meshes.
- **ParMmg** — parallel MMG for large meshes.
- **libigl** — surface processing, smoothing, repair.
- **PyMesh** — surface and volume mesh utilities; Boolean operations.

### Interop and IO

- **meshio** — universal mesh format translator.
- **trimesh** — STL/OBJ surface mesh utilities, watertightness checks.

### Solver-specific paths

- **TIOGA** — overset connectivity (research-grade OSS).
- **AMReX EB** — embedded boundary / cut-cell on AMR.
- **Basilisk** — alternative AMR + cut-cell, Octree-based.

## File layout per generated run

```
mesh_run/
├── geometry.py
├── mesh.py
├── quality.py
├── verify.py
├── export.py
├── run.py
└── outputs/
    ├── geom.step           # or geom.stl, depending on path
    ├── mesh_L0.msh         # coarse level
    ├── mesh_L1.msh         # medium level
    ├── mesh_L2.msh         # fine level
    ├── quality.csv
    ├── verification.json
    └── plots/
        ├── quality_hist_<metric>.png
        ├── bl_coverage.png
        ├── gci_plot.png
        └── worst_cells_3d.png
```

## verification.json schema (summary)

The full JSON Schema lives in `docs/verification.schema.json`. Key required fields:

- `spec_path`, `target_method`, `target_solver`
- `tool_versions` (object, tool → version string)
- `git_commit` (short SHA of the scaffold repo)
- `geometry_checks` (array of analytic-geometry comparisons)
- `quality_envelope` (per-metric threshold + measured + pass/fail)
- `grid_convergence` (levels, quantity, values, observed_order, gci_fine, asymptotic)
- `benchmark_comparison` (reference, computed, rel_error, passes) when applicable
- `hypotheses` (per-hypothesis verdict)
- `success_criteria_met` (boolean)
- `provenance` (host, arch, python, spec_hash, scaffold_hash, wall_time, peak_mem)

## Apple Silicon / ARM notes

For Indian-context and frugal-hardware paths, the following tools are ARM-native and work cleanly on M-series Macs:

- Gmsh — official ARM wheels available via PyPI.
- meshio — pure Python.
- trimesh — pure Python (with optional ARM-native deps).
- CadQuery / build123d — work via the OCCT wheels on ARM (some assembly required as of late 2024 / early 2026).
- AMReX — ARM-clean; build from source with the standard toolchain.
- MMG — builds cleanly via CMake on ARM.

Tools requiring extra care on ARM:

- **snappyHexMesh** — needs an ARM-native OpenFOAM build (the OpenFOAM-for-arm fork or the official OpenFOAM v2306+ on M-series) or Docker via Rosetta 2.
- **cfMesh** — same status as snappyHexMesh (it ships inside OpenFOAM).
- **TIOGA** — builds on ARM with care; some examples assume x86 MPI specifics; document the workaround in the worked example.

The scaffold records the host architecture and the exact tool build path in `verification.json` provenance.

## Subdomain routing

Inside the single `mesh-scaffold` skill, the backend is chosen by reading the spec's target method:

| Spec target method | Default backend | Alternates |
|---|---|---|
| FEM (conforming) | Gmsh → FEniCSx | Netgen → NGSolve; Gmsh → MFEM |
| FDM (structured) | Custom Cartesian generator | meshio export to plot file |
| FVM hex-dominant | cfMesh (Cartesian template) | snappyHexMesh (alternate) |
| IBM | Gmsh / Cartesian + STL | AMReX background + EB |
| Overset | Two Gmsh / blockMesh grids + TIOGA | OpenFOAM overset |
| Cut-cell | AMReX EB | Basilisk |
| DG / SE | Gmsh high-order | MFEM mesh |

## Versioning policy

- Major version bumps: backend additions that change the spec contract.
- Minor version bumps: new tool wrappers, new worked examples, new analysis checks.
- Patch version bumps: fixes, doc updates.

v0.1.0 is the initial release. v0.2.0 will harden the TIOGA path and add an anisotropic MMG worked example.
