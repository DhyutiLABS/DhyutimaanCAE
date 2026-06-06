---
name: mesh-problem-spec
description: Turn an informal description of a CAD/meshing task into a structured, falsifiable problem specification that downstream scaffolding and analysis agents can act on. Use this skill whenever the user wants to set up a mesh for a CFD or computational mechanics study, asks to "frame", "specify", "draft", or "set up" a meshing problem, mentions a geometry and a target solver, or hands over literature-survey notes and asks "what should we mesh?". Also trigger when the user says things like "I want a tet mesh for FEniCSx on this STEP file", "I need a snappyHexMesh setup for flow over this body", "let's IBM this sphere", "give me a Cartesian background plus STL for cut-cell", or asks for hypotheses, ablations, or a problem statement for a meshing study. The output is a markdown problem-spec document that the mesh-scaffold skill consumes as input. The spec forces a target-discretisation-method declaration and a quality envelope.
---

# Mesh Problem Spec

This skill turns a fuzzy "I want to mesh X for solver Y" request into a concrete, machine-readable problem spec. The spec is the contract: every downstream agent (scaffolding, mesh generation, quality audit, solver-prep, analysis) reads it and acts on its fields. Vague specs produce meshes that look fine but fail in the solver, so the work here is to *force* precision early — even at the cost of asking clarifying questions or stating assumptions explicitly.

The single most important field in the spec is the **target discretisation method**. Every other choice (element type, quality envelope, boundary-layer treatment, refinement strategy, export format) is downstream of it.

## When to use this skill

Trigger this skill when:

- The user wants to set up a mesh for a solver, even casually ("let's mesh this for OpenFOAM")
- The user has literature-survey notes and asks what to mesh next
- The user is unclear on a geometry-to-solver pipeline and wants help framing it
- A previous spec needs revision (new geometry, different solver, new resolution target)

Do *not* trigger this skill for pure literature questions, pure implementation questions on an already-specified mesh, or analysis of completed mesh runs. Those are different skills.

## Output: the problem spec document

Always produce a single markdown file named `problem-spec.md` with the following exact section structure. Downstream skills parse these headers, so do not rename or reorder them.

```markdown
# Mesh Problem Spec: <short name>

## 1. Problem statement
One paragraph in plain language. What geometry are we meshing, for which solver, and why.

## 2. Target discretisation method
Exactly one of: FEM (conforming), FDM (Cartesian structured), FVM (polyhedral / hex-dominant), IBM (immersed boundary, Cartesian + STL surface), Overset (Chimera, multiple grids + hole-cutting), Cut-cell (embedded boundary), DG (discontinuous Galerkin), SE (spectral element).
State the target solver explicitly: OpenFOAM (which solver application), FEniCSx, MFEM, SU2, NGSolve, AMReX (which app), Basilisk, libMesh, MOOSE, custom.
State the expected export format (.msh, .vtu, polyMesh, .e/.exo, .xdmf, MFEM v1.0, AMReX plotfile, etc.).

## 3. Geometry source
- Source: parametric (CadQuery/build123d), imported STEP/IGES, imported STL/OBJ, signed-distance field, analytic primitive
- Watertightness: confirmed yes / suspected yes / unknown / known leaks
- Defeaturing required: yes/no (and if yes, which features to drop and the tolerance)
- Coordinate system and units (mm/m/non-dimensional)
- Characteristic length scales: smallest geometric feature, largest, ratio

## 4. Domain and boundary tagging
- Outer domain bounds (or, for external flow, the farfield box size relative to the body characteristic length)
- Named boundaries with role: inlet, outlet, wall (no-slip / slip / wall-function), symmetry, periodic, farfield, interface, immersed surface
- Each boundary must carry a tag that survives export to the solver format

## 5. Resolution and quality envelope
- Target cell count (order of magnitude)
- Bulk cell size and near-wall first-cell height
- Boundary layer: number of prism layers, growth ratio, total thickness; target y+ band (if relevant)
- Quality envelope, with metric and threshold:
  - max non-orthogonality (OpenFOAM convention): default 70°
  - max skewness: default 4
  - max aspect ratio: default 1000 (FVM) / 100 (FEM)
  - min scaled Jacobian (FEM): default 0.1
  - min element quality (Gmsh SICN or similar): default 0.1
  - Override any of these in the spec; downstream code asserts them.

## 6. Reference / ground truth
How we will know the mesh is "right". One of, or combinations of:
- Analytic geometry test (e.g. sphere volume, surface area, channel length) with bound
- Method of manufactured solutions (MMS): a smooth field whose Laplacian/divergence is known, evaluated on this mesh and compared to the analytic value
- Grid convergence study (Roache GCI): L0/L1/L2 meshes, observed order of convergence, asymptotic-range check
- Canonical solver benchmark (lid-driven cavity, flow over cylinder, Poiseuille, sphere drag at known Re) compared to a published reference
- None available (flag this — it limits what the analysis agent can do; only acceptable when the user explicitly accepts a qualitative review)

## 7. Hypotheses (falsifiable)
2–4 hypotheses, each phrased so it can be confirmed or refuted by a specific measurement on the generated mesh or on a solver run on it.
Bad: "the mesh will be good."
Good: "On the lid-driven cavity at Re=100, the centerline u-velocity profile from this mesh matches Ghia et al. (1982) within 2% L2 across the three refinement levels, and observed order of convergence on the centerline velocity is in [1.5, 2.5]."
Good: "Boundary-layer prisms achieve y+ < 1 in 95% of wall cells on this geometry at the target inflow Reynolds number, without layer collapse in concave regions."

## 8. Success criteria
Specific numbers. E.g., "max non-orthogonality < 65°, max skewness < 3, and GCI on drag coefficient < 5% across L0/L1/L2."

## 9. Known failure modes to watch for
Method-specific things the analysis agent should explicitly check. Examples:
- snappyHexMesh layer collapse in concave regions
- Tet mesh sliver elements near sharp edges
- IBM band too thin to resolve boundary layer at the target Re
- Overset orphan cells / fringe-cell donor failures
- Hanging-node ratio outside FEM solver tolerance
- Aspect-ratio explosion in BL prisms over curved walls
- Watertightness loss after Boolean operations

## 10. Out of scope
What we are explicitly NOT doing in this iteration. Keeps follow-on questions grounded.
```

## Workflow

Follow these steps in order. Do not skip ahead.

### Step 1: Establish the problem

Read the conversation for the geometry, target solver, and goal. If any of the following are missing, ask the user — but ask in a single batched question, not one at a time:

- The geometry (and whether it is parametric, imported, or to be built from primitives)
- The target discretisation method and solver
- The flow/loading regime (Re, Mach, load magnitude) that determines resolution
- Whether a reference solution exists (benchmark, analytic, MMS, or none)

If the user is exploring and doesn't know yet ("I want to try meshing something simple"), suggest 2–4 canonical options with their tradeoffs and let them pick:

- **2D Poisson MMS on a unit square, FEM tet/tri mesh via Gmsh → FEniCSx** — fastest, cleanest, no boundary layer. Best for first contact.
- **Lid-driven cavity, FVM hex via blockMesh → OpenFOAM** — structured, exercises FVM quality conventions, has Ghia et al. (1982) reference.
- **Flow over sphere, IBM with Cartesian background + STL → AMReX or custom** — exercises immersed boundary; reference is sphere drag at Re=100.
- **Couette annulus with inner cylinder rotating, overset with TIOGA-style connectivity → OpenFOAM overset or AMReX** — exercises overset; reference is analytic Couette profile.

### Step 2: Pin the target discretisation method

This is the most important decision. Without it the mesh has no quality envelope, and the analysis skill cannot tell success from confident failure. Push the user to name exactly one method per spec. If they want to compare methods, write two specs.

Each method-element-quality combination has natural defaults:

- **FEM conforming**: linear or quadratic tetrahedra; min scaled Jacobian > 0.1; aspect ratio < 100; conforming faces; no hanging nodes (unless solver supports them).
- **FDM Cartesian**: uniform or stretched structured grid; aspect-ratio cap from CFL; no geometry conformance — geometry is approximated by ghost cells or IBM.
- **FVM hex-dominant / polyhedral**: cfMesh / snappyHexMesh / foamyHexMesh output; OpenFOAM non-orthogonality < 70°, skewness < 4; layer addition for walls.
- **IBM**: Cartesian background mesh + closed manifold STL surface; narrow refinement band around the surface; signed-distance computed at cell centers.
- **Overset**: two or more grids; inner body-fitted, outer background; explicit hole-cutting; donor-receptor connectivity; TIOGA or equivalent.
- **Cut-cell**: structured background with cells trimmed by geometry; small-cell stabilization or merging required.
- **DG / SE**: high-order elements; curved boundary elements; element Jacobian quality far stricter than linear FEM.

### Step 3: Choose the reference strategy

Prefer in this order:

1. **Analytic geometry tests** — sphere volume, channel length, surface area. Cheap to compute, gives a tight bound on geometry-side error.
2. **MMS on the mesh** — pick a smooth field $u$, evaluate $\nabla u$ or $\nabla^2 u$ analytically, compare to the same quantity computed by the solver's discrete operator on the generated mesh. Reveals discretisation error directly.
3. **Grid convergence (Roache GCI)** — three refinement levels (L0/L1/L2) with a fixed refinement ratio (typically $r=2$ in each direction). Compute observed order of convergence and the GCI. This is the standard for solver-coupled verification.
4. **Canonical benchmark with published reference** — lid-driven cavity (Ghia), flow over cylinder (Schäfer & Turek), sphere drag, Poiseuille, Couette. Use only when the geometry matches.
5. **No reference** — only acceptable if the user explicitly accepts a qualitative quality-metric-only review.

If you propose MMS or GCI, write out the manufactured field, the source term, the refinement ratios, and the asymptotic-range check criterion in section 6 fully evaluated. Downstream code will use these directly.

### Step 4: Propose mesh-generation tool and configuration

Default tool choices (the user can override):

- **FEM conforming tet on STEP/STL** → Gmsh (frontal-Delaunay 3D), export `.msh` v4.1 → meshio → FEniCSx `.xdmf`.
- **FEM conforming tet on parametric CAD** → CadQuery/build123d for the geometry, export STEP, then Gmsh as above.
- **FVM hex on simple geometry** → blockMesh (OpenFOAM native).
- **FVM hex-dominant on complex STL** → snappyHexMesh or cfMesh, with explicit BL layer count and BL thickness in spec.
- **IBM Cartesian + STL** → Gmsh or manual uniform/stretched Cartesian generator + meshio for the STL.
- **Overset** → two separate Gmsh / blockMesh grids + TIOGA connectivity step.
- **Cut-cell** → AMReX EB (embedded boundary) or Basilisk; geometry from STL or signed-distance.
- **Adaptive remeshing** → MMG (anisotropic) on an existing Gmsh mesh.

Surface mesh: keep it conforming with the volume mesher. For FVM hex tools that build their own surface, the input is just a clean STL; for FEM tet on STEP, Gmsh handles surface and volume together.

### Step 5: Draft falsifiable hypotheses

This is where the skill earns its keep. Push back if the user gives a vague hypothesis. Each hypothesis must specify:

- The intervention or comparison (what changes)
- The metric (what we measure on the mesh or on a solver run)
- The threshold or direction (what counts as confirmation)

Examples of well-formed hypotheses:

- "On the unit cube Poisson MMS with $u=\sin(\pi x)\sin(\pi y)\sin(\pi z)$, the discrete Laplacian computed on the Gmsh tet mesh has L2 error decreasing as $h^{2}$ to within $\pm 0.3$ in observed order of convergence across L0/L1/L2."
- "snappyHexMesh with 5 BL layers and growth ratio 1.2 achieves y+ < 1 in 95% of wall cells on the sphere at Re=100 on the L1 background mesh."
- "Drag coefficient on flow over sphere at Re=100 from IBM on a 256³ Cartesian background lies within 3% of Mittal et al. (2008) Cd = 1.09."
- "GCI on lid-driven-cavity centerline u-velocity at $y=0.5$ across L0/L1/L2 blockMesh hex grids is below 5%, with observed order in [1.5, 2.5]."

Examples that need rewriting:

- "The mesh will be good" → not falsifiable.
- "Hex is better than tet for this" → too broad; under which metric, for which solver, at what resolution.

### Step 6: Specify known failure modes to watch for

This section feeds directly into the analysis skill. For the canonical configurations, the typical entries:

- **Gmsh tet on STEP with sharp edges**: sliver elements at edges; quality cliff in regions of high curvature. Watch min scaled Jacobian and aspect ratio of the worst 1% of cells.
- **snappyHexMesh on complex STL**: layer addition fails in concave regions; non-orthogonality spikes at refinement interfaces. Watch BL coverage and non-orthogonality distribution, not just the maximum.
- **cfMesh hex-dominant**: feature-edge capture failures on filleted geometry; watch the surface mesh quality before volume meshing.
- **IBM Cartesian**: BL band too thin for the target Re; watch the number of cells across $\delta$ (estimated BL thickness).
- **Overset (TIOGA-style)**: orphan cells, fringe-cell donor failures, mass-conservation drift across overlap. Watch the donor-receptor map and the post-cutting cell counts.
- **Cut-cell**: small-cell time-step penalty without merging; non-physical cell volumes below machine tolerance.
- **MMG remeshing**: metric blow-up at points with under-resolved Hessian; watch the metric-field max and the cell-size ratio across one element.

### Step 7: Write out the spec and confirm

Write `problem-spec.md` to the working directory. Then summarize the spec to the user in 5–7 lines covering: geometry, target method-solver pair, reference strategy, the headline hypothesis, and the success threshold. Ask if anything should change before scaffolding begins.

## Worked example

User says: *"Let's mesh a sphere in a box for IBM in AMReX, just a demo."*

The skill should:

1. Pin method to IBM, solver to AMReX (incompressible/compressible app depending on context), export to AMReX plotfile.
2. Geometry: parametric sphere (radius $R$) inside a box $[-L, L]^3$ with $L = 8R$. Source: build123d → STL or signed-distance computed in code.
3. Resolution: Cartesian L0 base 64³, with a refinement band of 4 cells around the sphere; L1 = 128³, L2 = 256³.
4. Reference: sphere drag at Re=100 → Cd ~ 1.09 (Mittal et al. 2008); analytic sphere volume as a geometry check.
5. Headline hypothesis: "IBM Cartesian with 4-cell band at L2 yields Cd within 5% of 1.09 and observed order of convergence in [1.0, 2.0] across L0/L1/L2."
6. Failure modes: BL underresolution, signed-distance computation error near the surface, refinement-band leakage.
7. Out of scope: moving sphere, multiphase, compressible.

Write the spec, confirm, hand off to `mesh-scaffold`.

## Indian-context layer

When India/Bharat trigger words appear, bias defaults toward:

- Geometries from agri/healthcare/outdoor deployment (drip-irrigation channel, low-Re biomedical pipe, water-purifier flow path, monsoon-stormwater drain, small wind turbine blade)
- Hardware footprint sized for a single workstation or a small commodity cluster
- Tool choices that run cleanly on ARM/Apple Silicon (Gmsh, cfMesh, MMG, AMReX); flag snappyHexMesh's ARM caveats explicitly
- Reference data from Indian groups (NAL, IISc, IITM, IITB) where available

This is additive; the core methodological discipline does not change.

## Anti-patterns

- **Specifying without a target discretisation method.** The method-name field is mandatory; every other choice flows from it.
- **Specifying without a quality envelope.** "Make it look good" is not a spec. Numbers and metrics, always.
- **Treating mesh-quality metrics as proof of solver accuracy.** They are necessary not sufficient. The reference / GCI is what closes the loop.
- **Over-specifying mesh resolution before the verification strategy is clear.** Resolution is a hypothesis, not a given.
- **Mixing literature review into the spec.** That belongs in survey notes; the spec is operational.
- **Skipping the boundary-layer plan for wall-bounded viscous flows.** Either explicitly say "no BL needed" or specify layer count, growth, target y+.

## Handoff

When the spec is written and confirmed, tell the user: "Spec is at `problem-spec.md`. Ready for `mesh-scaffold`, which will generate the meshing pipeline code from this." Do not invoke the next skill automatically — give the user a chance to refine.
