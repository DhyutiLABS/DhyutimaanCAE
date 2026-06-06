---
name: literature-survey-mesh
description: Conduct a focused literature survey on CAD-to-mesh pipelines and mesh generation for CFD and computational mechanics, and produce a structured knowledge base that the mesh-problem-spec skill can consume. Use this skill whenever the user asks for a literature survey, lit review, "state of the art", or "recent work" on parametric CAD (CadQuery, build123d, OpenCASCADE, FreeCAD), mesh generation (Gmsh, Netgen, cfMesh, snappyHexMesh, MMG, TetGen, CGAL), mesh adaptation, boundary-layer extrusion, overset/Chimera connectivity (TIOGA), cut-cell or embedded-boundary methods, IBM-compatible meshing, polyhedral meshing, mesh quality metrics, or method-aware discretisation choices (FEM/FDM/FVM/IBM/Overset/DG/spectral element). Also trigger when the user uploads meshing papers and asks for synthesis, or asks "what has been tried" for a specific geometry or solver. Produces a knowledge base with a meshing-specific taxonomy and a "what to try next" section that feeds problem framing.
---

# Mesh & CAD Literature Survey

This skill conducts a focused, structured literature survey on CAD-to-mesh pipelines for CFD and computational mechanics. The output is a knowledge base in a format the `mesh-problem-spec` skill consumes directly. The survey is *operational*: the question it answers is always "given a target discretisation method and geometry, what does the literature tell us to try next?", not "everything ever written about mesh generation".

This skill is a specialization of the general `literature-survey` skill (if present in the environment). The mesh specialization adds:

1. A method-aware taxonomy that maps every paper to a target discretisation (FEM, FDM, FVM, IBM, overset, cut-cell, DG, spectral element)
2. A solver-coupling view that surfaces which mesh formats and quality envelopes are required by which OSS solvers (OpenFOAM, FEniCSx, MFEM, SU2, AMReX, Basilisk, NGSolve)
3. A handoff format that the problem-spec skill reads directly

## When to use this skill

Trigger when the user:

- Asks for a literature survey, review, or summary on mesh generation, mesh adaptation, or CAD-to-CFD/FEM pipelines
- Mentions a specific geometry class and asks "what's been done for meshing this?"
- Uploads meshing or pre-processing papers and asks for synthesis
- Asks about a failure mode ("why does snappyHexMesh struggle with thin gaps?") and wants references
- Is preparing to write a problem spec and needs grounding in prior work
- Compares OSS tools for a given solver coupling (e.g. "Gmsh vs Netgen for FEniCSx")

Do *not* trigger this skill for solver-internals questions, post-processing/visualization questions, or general numerical-methods literature reviews unconnected to discretisation/meshing.

## Sources to search

Search in this order of priority:

1. **Journal of Computational Physics**, **Computer Methods in Applied Mechanics and Engineering**, **International Journal for Numerical Methods in Engineering / in Fluids**, **Engineering with Computers**, **CAD**, **Computer-Aided Design**, **Computational Geometry: Theory and Applications** — the canonical engineering and geometry venues
2. **arXiv** (cs.CE, cs.CG, math.NA, physics.flu-dyn) — for preprints and method papers
3. **International Meshing Roundtable (IMR)** proceedings — the field's dedicated venue; many landmark algorithms appear here first
4. **Procedia Engineering** — IMR proceedings are sometimes routed through here
5. **OpenFOAM Journal**, OpenFOAM workshop proceedings, **FEniCS conference papers** — for solver-coupling specifics
6. **GitHub** — for living implementations (Gmsh, cfMesh, MMG, snappyHexMesh, TIOGA, AMReX, libMesh, MFEM); README and issue trackers often document failure modes more honestly than papers
7. **Documentation sites** of the OSS tools themselves (gmsh.info, openfoam.com docs, fenicsproject.org, mfem.org, amrex-codes.github.io) — for tool capabilities, quality metric definitions, and recommended workflows
8. **Author and group pages** — Geuzaine and Remacle (Gmsh, Liège), Schöberl (Netgen, TU Wien), Frey and Borouchaki (MMG lineage, Sorbonne/INRIA), Si (TetGen, WIAS Berlin), Fabritius/Jasak (cfMesh, OpenFOAM lineage), Tomac/Mavriplis (overset), the AMReX team (LBNL), Popinet (Basilisk), and Indian groups working in mesh adaptation (IISc, IITM, IITB, NAL)

If MCP connectors for arXiv, Semantic Scholar, or institutional repositories are available, prefer them over web search — they return structured metadata.

## Query expansion

A good meshing survey query is rarely a single phrase. Expand the user's query along multiple axes before searching:

- **Algorithm class**: advancing front, Delaunay refinement, frontal-Delaunay, octree/quadtree, paving, sweep, block-structured, isogeometric
- **Element type**: tetrahedral, hexahedral, hex-dominant, polyhedral, prismatic boundary layer, mixed/hybrid, polygonal
- **Target method**: FEM conforming, FVM polyhedral, FVM hex-dominant, FDM Cartesian, IBM (immersed boundary), cut-cell / embedded boundary, overset / Chimera, DG (discontinuous Galerkin), spectral element, isogeometric analysis
- **Adaptation**: a posteriori error estimation, anisotropic adaptation, metric-based adaptation, hierarchical refinement (AMR), p-refinement, hp-adaptive
- **Quality metrics**: skewness, aspect ratio, Jacobian (min/scaled), orthogonality, non-orthogonality (OpenFOAM convention), warpage, dihedral angles, shape regularity, condition number of stiffness matrix
- **CAD interop**: STEP, IGES, STL, BREP, NURBS, watertightness, defeaturing, mid-surface extraction, healing
- **Boundary layer**: prism extrusion, structured BL, viscous spacing, y-plus targeting, layer collapse, layer addition failure modes
- **Tool names**: Gmsh, Netgen, cfMesh, snappyHexMesh, blockMesh, foamyHexMesh, MMG, ParMmg, TetGen, Triangle, CGAL, OpenCASCADE, OCCT, FreeCAD, CadQuery, build123d, pythonOCC, Salome, libMesh, MFEM mesh, MOAB, MeshKit, Pumi, PETSc DMPlex
- **Specific failure modes**: poor convergence at high-aspect-ratio cells, locking, hanging nodes, orphan cells in overset, hole-cutting failures, BL collapse in concave regions, thin-gap closure

For each axis, generate 2–3 search queries. The meshing literature uses heavily fragmented terminology (e.g. "skewness" is defined differently in OpenFOAM, ANSYS, and FEM texts) — synonym variation matters.

## Output: mesh knowledge base

Always produce a single markdown file named `mesh-knowledge-base.md` with this structure:

```markdown
# Mesh & CAD Literature Survey: <topic>

## Scope
What we searched for, what is in/out of scope, when this survey was done, target discretisation method(s) and geometry class.

## Headline findings
3–5 bullets. The most decision-relevant takeaways. What works, what is contested, what is open.

## Taxonomy of relevant work
Group papers by category. Categories to consider (only include those represented in the literature you found):

- Parametric CAD and BREP modelling
- CAD-to-mesh interop (STEP/STL/BREP, defeaturing, healing, watertightness)
- Unstructured tet meshing (advancing front, Delaunay, frontal-Delaunay)
- Hex-dominant / polyhedral meshing for FVM
- Boundary-layer mesh extrusion
- Block-structured and multi-block meshing
- Cartesian / immersed boundary mesh generation
- Cut-cell and embedded-boundary discretisation
- Overset / Chimera connectivity and hole-cutting
- Adaptive mesh refinement (AMR) — hierarchical and metric-based
- Mesh quality, repair, and remeshing
- Solver-coupling and mesh-exchange formats

For each category, give:
- 1-paragraph description
- Key papers (with year and venue)
- Tool(s) implementing the method
- Known limitations and open critiques

## Findings by target method
If the user has a specific discretisation in focus (FEM/FDM/FVM/IBM/overset/cut-cell/DG/SE), dedicate a section to it:
- What mesh types it demands
- What quality envelopes it tolerates
- What's been tried for the user's geometry class
- Reference benchmarks and canonical test cases
- Known failure modes specific to this method-geometry pairing

## Findings by geometry class
If the user's geometry has distinguishing features (thin gaps, sharp edges, multiscale features, complex internal channels, moving parts), dedicate a section.

## OSS tool landscape
For each tool the survey turned up:
- What it does, who maintains it, license
- Strengths and known weaknesses (cite where possible)
- Solver coupling: what formats it exports and which solvers ingest them cleanly
- Indian-context note where relevant: hardware footprint, frugal HPC viability, availability of training material

## Quality metric conventions
A short reference for which quality metrics are defined how, and by which community:
- OpenFOAM non-orthogonality, skewness, max-aspect-ratio
- Gmsh quality (SICN, gamma, etc.)
- FEM Jacobian (min, scaled, condition number)
- SU2 dual-mesh skew
- Where definitions disagree and how to translate

## Open problems and contested points
Where the literature disagrees. Examples: when is hex-dominant preferable to pure tet, whether quality metrics predict solver accuracy, whether AMR pays for itself in steady FVM, whether cut-cell can match body-fitted accuracy without stabilization.

## Common failure modes documented in the literature
For each failure mode, cite the paper(s) or issue threads that documented it and any proposed remedies.

## Recommended starting points for problem-spec
3–5 specific suggestions, each phrased so the problem-spec skill can act on it.
Example: "For a sphere-in-channel IBM study at Re~100, the canonical reference is Mittal et al. (2008); generate a uniform Cartesian background mesh with an STL surface, and target a y+ < 1 narrow band of 4 cells around the sphere."

## Sources
Numbered list with full citations and links/DOIs.
```

## Workflow

### Step 1: Establish scope

Read the user's request and the conversation context. Identify:

- The target discretisation method(s): FEM, FDM, FVM, IBM, overset, cut-cell, DG, spectral element
- The geometry class: simple primitive, parametric assembly, imported STEP/STL, scanned point cloud
- Specific failure mode or pain point if any
- Whether the user has papers already in scope, and whether they want a state-of-the-field or a targeted question

If the scope is broad ("survey mesh generation"), narrow with the user before searching — a broad survey is rarely operational. A useful framing question: "what's the target solver and the target geometry class?"

### Step 2: Expand queries and search

Generate 8–14 search queries across the axes listed above. Run them in parallel where possible. For each result:

- Capture title, authors, year, venue, link
- Extract: algorithm class, target element type, target method, claim, evidence
- Note whether code is available and which OSS tool implements the idea

Do not stop at the first page of results — landmark meshing papers from the 1990s and 2000s are sometimes 2–3 pages deep in modern search results.

### Step 3: Read the most important sources carefully

After the broad scan, pick 6–12 sources for closer reading based on:

- Citation impact (discount for very recent papers)
- Relevance to the target method-geometry pairing
- Implementation availability (papers backed by OSS tools are more useful as references)
- Whether they document failure modes, not just successes

For each, extract:

- Algorithm and element type
- Quality envelope claimed
- Test cases and ground truth
- Hardware and runtime reported
- Stated limitations

### Step 4: Build the taxonomy

Don't pretend every paper fits cleanly. Some methods straddle categories (e.g. frontal-Delaunay is both Delaunay and advancing-front; cut-cell methods sit between IBM and body-fitted). Flag papers that are over-cited but rarely independently reproduced.

### Step 5: Surface contested points

The meshing literature has real disagreements — examples include hex vs tet for FVM accuracy, whether quality metrics correlate with solver error in practice, whether cut-cell can avoid the small-cell time-step penalty without stabilization, and whether AMR pays for itself for steady flows. Include a "contested" section that names these debates and points to papers on each side.

### Step 6: Write the recommendations section

This is the handoff to `mesh-problem-spec`. Each recommendation should be specific enough that the problem-spec skill can use it as a starting point. Cite the source. Tie each recommendation to a target discretisation method and a verification strategy.

### Step 7: Be honest about gaps

If the OSS literature is thin on the user's specific question (overset is a classic example — most overset work cites commercial tools), say so explicitly and call out where the user will have to extrapolate or self-validate.

## Citation discipline

Every claim about what a paper or tool showed must be tied to that source, by author/tool and year at minimum. Do not paraphrase results from memory — if you can't find the source, leave the claim out. Meshing literature contains folklore ("snappyHexMesh always produces bad layers in concave regions", "tet meshes are always worse than hex for FVM") that is more cited than supported; flag folklore explicitly when you find it.

When searching the web, follow the citation rules in the environment's search instructions — paraphrase, limit quotes, attribute claims.

## Indian-context layer

When the conversation contains India/Bharat trigger words (NAL, IISc, IITM, IITB, IITH, IITKgp, ANRF, DST, MeitY, CSIR-4PI, "frugal", "low-cost cluster"), shift the survey to also surface:

- Indian groups publishing in this area (NAL aerospace meshing, IISc fluid mechanics, IITB CASDE, IITM CFD groups, CSIR-4PI HPC)
- Frugal hardware paths (single-workstation meshing, MPI on commodity nodes, ARM/Apple Silicon viability for pre-processing)
- Domain emphasis: rural-tech (small-scale water/agriculture flow), agri-engineering geometries, MSME-scale CAD-to-CFD pipelines
- ANRF/DST/MeitY funding calls that explicitly mention CAD-CFD-FEM tooling

This is additive, not a replacement — the global survey still happens first.

## Anti-patterns

- **Producing an encyclopedic list with no narrative.** The user wants to know what to try, not what exists.
- **Citing recent preprints as established results.** Note the venue and review status.
- **Confusing quality metrics across communities.** OpenFOAM's "skewness" is not Gmsh's "gamma" is not the FEM Jacobian; clarify.
- **Treating commercial tools as relevant when the user asked for OSS.** Mention them only as benchmarks; do not propose them as paths.
- **Hand-waving on what "doesn't work."** If a paper or tool documents a failure, cite it; otherwise it's folklore.
- **Pretending overset OSS is more mature than it is.** TIOGA is the credible option and it is research-grade; say so.
- **Reproducing benchmark mesh-quality numbers without their solver context.** A "good" mesh for FVM may be a bad mesh for FEM.

## Handoff

After the knowledge base is written, summarize for the user in 6–10 lines: scope, target method-geometry pairing, top 3 takeaways, and the 2–3 recommendations most relevant to their next problem-spec. Then say:

> Knowledge base is at `mesh-knowledge-base.md`. The "Recommended starting points for problem-spec" section is the natural input to the `mesh-problem-spec` skill when you're ready.
