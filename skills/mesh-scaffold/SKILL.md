---
name: mesh-scaffold
description: Generate verifiable CAD-to-mesh implementation code from a problem-spec.md file. Use this skill whenever the user wants to implement, scaffold, or write code for a mesh-generation pipeline, asks to "build", "code up", "implement", or "set up" a mesh for a CFD or computational mechanics solver, or hands over a problem spec and asks for an implementation. Also trigger when the user asks for a Gmsh script, a CadQuery/build123d parametric model, a snappyHexMesh case, a cfMesh setup, an MMG remeshing pipeline, a TIOGA overset connectivity step, an AMReX embedded-boundary geometry, or any pipeline that wraps OSS mesh tools (Gmsh, Netgen, cfMesh, snappyHexMesh, MMG, TetGen, CGAL, libigl, pythonOCC, FreeCAD). Produces a self-contained Python package wrapping the chosen tool(s), with quality-audit, reference-comparison, and a verification.json artifact for the analysis skill.
---

# Mesh Scaffold

This skill generates working, verifiable CAD-to-mesh code from a problem spec. The default backbone is **Python + Gmsh + meshio**, with tool-specific extensions (cfMesh / snappyHexMesh / MMG / TIOGA / AMReX EB) selected by the target discretisation method declared in the spec.

The generated code must be runnable end-to-end and must produce both a mesh file in the solver's native format *and* a `verification.json` artifact that compares mesh quality and (where defined) solver-coupled metrics against the spec's reference and quality envelope.

**Frameworks are wrapped, not reimplemented.** Gmsh, cfMesh, snappyHexMesh, MMG, TIOGA, AMReX EB, OpenCASCADE — none of these is rebuilt. The scaffold is a thin Python wrapper that drives the tool, captures provenance, and asserts the spec's envelope.

## When to use this skill

Trigger this skill when:

- The user has a `problem-spec.md` (or a clear equivalent) and wants the mesh-generation implementation
- The user asks to "scaffold", "implement", "build", or "code up" a mesh-generation pipeline
- The user wants to swap meshing backends on an existing problem (e.g., "give me the cfMesh version of this snappyHexMesh case")
- The user wants to add an adaptive remeshing pass (MMG) on top of an existing mesh
- The user wants to add overset connectivity (TIOGA) to two existing grids

Do not trigger this skill for problem framing (use `mesh-problem-spec`) or for analysis of an already-generated mesh (use `mesh-analysis-report`).

## Backend selection

Choose by reading the spec's section 2 ("Target discretisation method") and section 5 ("Resolution and quality envelope"):

- **Gmsh** (default for FEM tet/hex, IBM background, surface meshing): Python bindings via `gmsh` PyPI package. Best for transparency and reproducibility — the entire mesh recipe is a Python script.
- **cfMesh** (preferred for FVM hex-dominant on STL): driven via OpenFOAM's case-dictionary mechanism, wrapped from Python with `subprocess`.
- **snappyHexMesh** (alternate for FVM hex-dominant): same wrapping pattern as cfMesh; mention the ARM/Apple-Silicon caveats explicitly.
- **MMG / ParMmg** (anisotropic adaptive remeshing on an existing tet mesh): wrapped via subprocess; metric field is computed in Python from a Hessian estimator.
- **TIOGA** (overset connectivity): library bindings or subprocess; orchestrate two-grid setup then call TIOGA's connectivity step.
- **AMReX EB** (cut-cell embedded boundary): native AMReX inputs file + Python pre-processor for the geometry.
- **CadQuery / build123d** (parametric CAD upstream of any of the above): produces STEP or STL that feeds the mesher.

If the user has not specified, default to Gmsh + meshio and state why. Offer to emit a second backend as a comparison.

## Output structure

Generate the following files in the working directory. Names matter — downstream skills (analysis, reporting) look for them.

```
mesh_run/
├── geometry.py         # Parametric CAD or STEP/STL ingestion; outputs <case>.step / <case>.stl
├── mesh.py             # Mesh-generation script: Gmsh / cfMesh dict / snappyHexMeshDict / MMG call / etc.
├── quality.py          # Quality audit: computes the metrics declared in spec section 5
├── verify.py           # Verification harness: geometry checks, MMS / GCI, benchmark comparison
├── export.py           # Convert mesh to the solver's native format (FEniCSx .xdmf, OpenFOAM polyMesh, AMReX, etc.)
├── run.py              # Top-level entry point: geometry → mesh → quality → verify → export
└── outputs/            # Created at runtime
    ├── mesh.msh        # Or polyMesh/, plt files, etc.
    ├── mesh_L0.msh     # Refinement levels for GCI
    ├── mesh_L1.msh
    ├── mesh_L2.msh
    ├── quality.csv     # Per-cell or aggregated quality metrics
    ├── verification.json
    └── plots/          # quality histograms, BL coverage map, GCI plot
```

Each file should be self-contained and runnable. Imports at the top, no hidden dependencies. Tool versions captured at run time and written into `verification.json`.

## Workflow

### Step 1: Read the spec

Read `problem-spec.md` from the working directory. If it does not exist, ask the user for it before generating any code — do not hallucinate a spec. Extract:

- Target discretisation method and solver (section 2)
- Geometry source (section 3)
- Boundary tagging (section 4)
- Quality envelope and resolution (section 5)
- Reference strategy (section 6) — analytic, MMS, GCI, benchmark, or none
- Hypotheses (section 7) — these become test cases in `verify.py`
- Success criteria (section 8) — these become the pass/fail thresholds asserted by the verification harness

### Step 2: Generate `geometry.py`

This file is the geometry, isolated from meshing. It should contain one of:

- **Parametric path** (CadQuery / build123d): a function `build_geometry(params: dict) -> Path` that produces a STEP or STL file at a known path. Parameters from the spec.
- **Import path**: a function `load_geometry(path: Path) -> Path` that reads an existing STEP/STL, performs basic healing (closing small gaps, removing tiny features below the defeaturing tolerance), and writes the cleaned file.
- **Analytic path**: explicit primitive construction (box, sphere, cylinder) via Gmsh's OpenCASCADE kernel.

Always emit a watertightness check at the end: for STL, compute Euler characteristic and report manifoldness via `trimesh` or equivalent.

Example skeleton for an analytic sphere-in-box:

```python
# geometry.py
import gmsh
from pathlib import Path

def build_geometry(R: float = 0.5, L: float = 4.0, out: Path = Path("outputs/geom.step")) -> Path:
    gmsh.initialize()
    gmsh.model.add("sphere_in_box")
    sphere = gmsh.model.occ.addSphere(0, 0, 0, R)
    box = gmsh.model.occ.addBox(-L, -L, -L, 2 * L, 2 * L, 2 * L)
    gmsh.model.occ.cut([(3, box)], [(3, sphere)])
    gmsh.model.occ.synchronize()
    out.parent.mkdir(parents=True, exist_ok=True)
    gmsh.write(str(out))
    gmsh.finalize()
    return out
```

### Step 3: Generate `mesh.py`

This is the meshing recipe. Backend-specific:

**Gmsh path** (FEM, IBM background, surface mesh):

- Open the geometry file, set up physical groups corresponding to the named boundaries in spec section 4.
- Set mesh size fields per the resolution targets in spec section 5. Use `Threshold` fields for graded sizing, `BoundaryLayer` field for prism BL.
- Choose algorithm: `Mesh.Algorithm3D = 1` (Delaunay) or `10` (HXT) for 3D; `Algorithm = 6` (frontal-Delaunay) for 2D.
- Generate mesh, export `.msh` v4.1.
- For GCI, regenerate at L0/L1/L2 with refined size fields and save separately.

**cfMesh path** (FVM hex-dominant on STL):

- Write a `meshDict` with the STL path, max cell size, BL layers, feature-edge angle.
- Drive via `subprocess.run(["cartesianMesh"])` and check return code.
- Run `checkMesh` and capture stdout to `quality.csv`.

**snappyHexMesh path** (FVM hex-dominant alternate):

- Write `blockMeshDict` for the background mesh, `snappyHexMeshDict` with the STL and refinement levels.
- Drive via `subprocess.run(["blockMesh"])` then `subprocess.run(["snappyHexMesh", "-overwrite"])`.
- Repeat at L0/L1/L2 by halving `nCellsBetweenLevels` and the background cell count.
- Capture `checkMesh` output.
- Note: snappyHexMesh native binaries on Apple Silicon may require running through OpenFOAM-for-arm or Docker; document the path used.

**MMG path** (anisotropic remeshing on a tet mesh):

- Read the input `.mesh` (Medit format) or convert from Gmsh via meshio.
- Compute or load a metric field — for verification, use a constant or analytic Hessian; for production, an a posteriori estimator.
- Run `mmg3d_O3 -in mesh.mesh -sol mesh.sol -out adapted.mesh -hmin <h> -hmax <H>`.
- Convert back to the target format.

**TIOGA overset path**:

- Generate two grids in two Gmsh scripts (or blockMesh + snappy for OpenFOAM overset).
- Run TIOGA's exchange-connectivity step with each grid's node list and connectivity.
- Save the donor-receptor map; expose it to the solver via the solver's expected interface.
- Audit: orphan-cell count, fringe-cell count, mass-imbalance estimate across the overlap.

**AMReX EB path** (cut-cell):

- Define the embedded geometry in a Python pre-processor using AMReX's `EB2::IndexSpace`-style constructors (or via signed-distance from STL).
- Emit an AMReX inputs file specifying domain, base grid, max refinement level, and EB geometry.
- Run the AMReX app or a minimal `eb-mesh-only` driver.

### Step 4: Generate `quality.py`

This file computes the quality metrics declared in spec section 5. Do not reinvent metrics — wrap the tool's native output where possible:

- Gmsh: `gmsh.model.mesh.getElementQualities("minSICN")` and `"gamma"`, etc. — these are the documented Gmsh quality fields.
- OpenFOAM: parse `checkMesh` output for non-orthogonality, skewness, aspect ratio.
- meshio: aspect ratio and volume from element coordinates for any format.
- For FEM Jacobian and condition number, compute from element node coordinates using standard finite-element formulas (give a small helper module).

Output: a `quality.csv` with one row per cell (or aggregated per region for very large meshes) and per-metric histograms saved to `outputs/plots/`. Also emit aggregate stats (min, max, mean, median, 95th and 99th percentile) for each metric.

Assert: each metric is within the envelope from spec section 5. Failed assertions go into `verification.json` as failures, not Python exceptions — the analysis skill needs to see them.

### Step 5: Generate `verify.py`

This is the verification harness. It must:

- For analytic geometry checks: load the mesh, compute the geometric quantity (sphere volume, channel length, surface area), compare to the analytic value, report relative error.
- For MMS: evaluate the manufactured field on cell centers / nodes, compute the discrete operator (Laplacian via finite differences on Cartesian, via shape-function gradients on FEM) using a small reference implementation in Python or by calling the solver, and compare against the analytic source.
- For GCI: load the L0/L1/L2 meshes, run the solver on each (subprocess to OpenFOAM/FEniCSx/AMReX), extract the target quantity (drag, centerline velocity, L2 of solution), compute observed order of convergence and GCI per Roache.
- For benchmark comparison: run the canonical case, extract the comparable quantity, compute relative error against the published reference.
- For each hypothesis in spec section 7, compute the measurement, apply the threshold, mark confirmed / refuted / inconclusive.
- Save all numerical outputs to `outputs/verification.json`.

Verification output schema (`outputs/verification.json`):

```json
{
  "spec_path": "problem-spec.md",
  "target_method": "IBM",
  "target_solver": "AMReX",
  "tool_versions": {"gmsh": "4.12.2", "meshio": "5.3.5", "amrex": "24.05"},
  "git_commit": "<short-sha-of-scaffold-repo>",
  "geometry_checks": [
    {"name": "sphere_volume", "analytic": 0.5236, "computed": 0.5219, "rel_error": 0.0032, "passes": true}
  ],
  "quality_envelope": {
    "max_non_orthogonality_deg": {"threshold": 70, "measured": 62.4, "passes": true},
    "max_skewness": {"threshold": 4, "measured": 3.2, "passes": true},
    "min_scaled_jacobian": {"threshold": 0.1, "measured": 0.14, "passes": true}
  },
  "boundary_layer": {"target_yplus": 1.0, "fraction_below_target": 0.96, "passes": true},
  "grid_convergence": {
    "levels": ["L0", "L1", "L2"],
    "quantity": "Cd",
    "values": [1.18, 1.13, 1.10],
    "observed_order": 1.78,
    "gci_fine": 0.027,
    "asymptotic": true
  },
  "benchmark_comparison": {
    "reference": "Mittal et al. 2008, Cd=1.09",
    "computed": 1.10,
    "rel_error": 0.009,
    "passes": true
  },
  "hypotheses": [
    {
      "id": 1,
      "statement": "Cd within 5% of 1.09 on L2",
      "result": "confirmed",
      "evidence": "rel_error 0.9% < 5%"
    }
  ],
  "success_criteria_met": true,
  "provenance": {
    "host": "<hostname>", "arch": "arm64", "python": "3.12.x",
    "spec_hash": "<sha256>", "scaffold_hash": "<sha256>"
  }
}
```

The analysis skill reads this file. Keep the keys stable.

### Step 6: Generate `export.py`

Convert the mesh to the solver's native format:

- Gmsh `.msh` → FEniCSx `.xdmf` via meshio.
- Gmsh `.msh` → OpenFOAM polyMesh via `gmshToFoam`.
- cfMesh / snappyHexMesh polyMesh — already native.
- Cartesian + STL → AMReX inputs file with EB geometry definition.
- MMG `.mesh` → target format via meshio.

Preserve boundary tags; they must survive the export. The analysis skill will check that boundary cell counts match between mesh and exported case.

### Step 7: Generate `run.py`

A thin entry point that runs geometry → mesh → quality → verify → export. No surprises — just orchestration. Capture tool versions and git commit into the provenance block at the start.

### Step 8: Smoke-test the generated code

Before handing back to the user, attempt to run `run.py` end-to-end at the coarsest resolution (L0 only). If you cannot execute (tool missing, no GPU/MPS available, no OpenFOAM in the env), at minimum:

- Run `python -m py_compile` on every generated file
- Run `gmsh -check` on the geometry if Gmsh is available
- Print a clear note about what was skipped and why

Report explicitly which checks ran and which were skipped. The analysis skill should not assume a clean smoke test.

## Backend-specific notes

### Gmsh

- Python API: `import gmsh`, `gmsh.initialize()`, `gmsh.finalize()` always wrapped in try/finally.
- Use the OpenCASCADE kernel (`gmsh.model.occ`) for Boolean operations on imported STEP / built primitives; use the built-in kernel only for very simple cases.
- Physical groups are how named boundaries survive export. Always assign them — `meshio` will not invent them.
- For 3D, prefer `Mesh.Algorithm3D = 10` (HXT) for speed; fall back to `1` (Delaunay) for robustness.
- For boundary layers in 3D, use the `BoundaryLayer` field on surfaces — note Gmsh's BL is single-sided and has known limitations on concave geometry.

### cfMesh and snappyHexMesh

- Both are best driven through OpenFOAM-style case directories with `system/` and `constant/` subdirectories.
- snappyHexMesh's three phases (castellate, snap, addLayers) should be controlled independently; assert success of each phase.
- Layer addition is the most failure-prone phase; always emit a BL-coverage diagnostic.
- Apple Silicon: snappyHexMesh requires either an ARM-native OpenFOAM build (OpenFOAM-for-arm fork) or Docker; document which path the scaffold expects.

### MMG / ParMmg

- Input format is Medit `.mesh` + `.sol`; convert from Gmsh `.msh` via meshio.
- Metric field must be SPD; check eigenvalues before passing.
- For ParMmg, partition the mesh first; the scaffold should use ParMmg only if the mesh exceeds a size threshold (e.g. > 1M cells).

### TIOGA

- TIOGA is research-grade OSS; expect to read its examples to set up the donor-receptor exchange.
- For the v0.1.0 demo, prefer a simple two-grid case (background + body-fitted patch) and pin a tested combination of grids.

### AMReX EB

- Define geometry via signed-distance from STL (`amrex::EB2::STLtoSDF`) or via primitive intersections.
- Watch small-cell time-step penalty: assert min EB cell volume above a fraction of the regular cell volume, or enable AMReX's small-cell merging.

## Common failures and how to avoid them

- **Mesh generated but boundary tags lost**: always assign Gmsh physical groups before export; meshio preserves them only if they exist.
- **Quality looks fine but solver diverges**: usually a quality metric on the worst 0.01% of cells. Always report 99th and 99.9th percentile, not just max/mean.
- **MMS test passes but real solver fails**: MMS tests the discretisation, not the solver's auxiliary terms (turbulence model, multiphase coupling). Be explicit that MMS scope is the discrete operator only.
- **GCI computed without asymptotic-range check**: GCI is meaningless outside the asymptotic range. Always check observed order against expected order; if they differ by more than ~30%, flag GCI as unreliable.
- **Tool version drift breaks reproduction**: capture exact versions and pin them in `verification.json` provenance. The analysis skill compares.

## Worked example skeleton: 2D Poisson MMS, Gmsh → FEniCSx

Spec specifies: unit square, Dirichlet $u=0$ on all sides, MMS with $u=\sin(\pi x)\sin(\pi y)$, target FEM, three refinement levels (h = 1/16, 1/32, 1/64), success at observed order in [1.7, 2.3] for $P_1$ elements.

Scaffold produces:

- `geometry.py`: Gmsh OCC unit square with physical group `boundary` on all four edges
- `mesh.py`: three Gmsh runs at the three mesh sizes; export `mesh_L0.msh`, `mesh_L1.msh`, `mesh_L2.msh`
- `quality.py`: aspect ratio histogram, min scaled Jacobian
- `verify.py`: load each mesh in FEniCSx, solve the Poisson problem with the MMS source, compute L2 error against the exact $u$, fit the observed order, emit `verification.json`
- `export.py`: meshio conversion to `.xdmf`
- `run.py`: orchestration
- Smoke test: compile-check all files; run `mesh.py` at L0 only if Gmsh present

Total scaffold: ~400 lines, all human-readable, all wrapping rather than reimplementing.

## Indian-context layer

When India/Bharat trigger words appear:

- Prefer Gmsh + meshio + FEniCSx + AMReX (all ARM/Apple-Silicon native and frugal)
- Flag snappyHexMesh's ARM caveats with a clear "use Docker or ARM-OpenFOAM" note
- For the worked example, default geometries to agri/healthcare/outdoor relevance when a generic primitive is interchangeable
- Capture memory footprint and wall-time in `verification.json` provenance; document single-workstation viability

## Handoff

After the files are written and smoke-tested, tell the user:

> Scaffold is at `mesh_run/`. To generate, audit, and verify: `python mesh_run/run.py`. Outputs land in `mesh_run/outputs/`. Once the run finishes, the `mesh-analysis-report` skill will pick up `quality.csv` and `verification.json` to produce the report.

Do not auto-invoke the analysis skill. Let the user inspect the mesh first.

## Anti-patterns

- **Generating code that doesn't actually use the spec.** If you find yourself writing generic boilerplate, re-read the spec and bake in the specific geometry, BCs, and reference.
- **Reimplementing a mesher.** Gmsh / cfMesh / snappyHexMesh / MMG / TIOGA / AMReX are wrapped, never rebuilt. The scaffold's job is orchestration and verification.
- **Skipping the verification harness.** A mesh with no quantitative ground-truth check is a confident hallucination. The harness is non-negotiable.
- **Reporting only the max of a quality metric.** The 99th and 99.9th percentile carry the failure mode.
- **Writing code you didn't even syntax-check.** Always at minimum compile the files before declaring done.
- **Hiding the geometry tagging step.** Boundary tags are how the mesh talks to the solver; they must be explicit and reproducible.
