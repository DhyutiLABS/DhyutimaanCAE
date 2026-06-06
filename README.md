# DhyutimaanMesh

**Dhyutimaan** (द्युतिमान्, Sanskrit: "radiant, illuminating") · **Mesh** — a Claude-native skill bundle for CAD and meshing in CFD and computational mechanics workflows, using open-source software.

Part of the **DhyutiLABS** family of agentic research bundles. Sibling bundles: DhyutimaanPI (physics-informed neural networks), DhyutimaanCV (computer vision and reconstruction), DhyutimaanForensics (media forensics), DhyutimaanComplexity (complex systems and nonlinear dynamics), DhyutimaanRobotics (robotics and AI-in-robotics).

## What this bundle does

It turns a fuzzy "I want to mesh X for solver Y" request into a verified, reproducible, method-aware mesh — from CAD source to a solver-native mesh file, with a falsifiable quality envelope and reference comparison at every step.

The bundle is **method-aware**: every spec must declare its target discretisation method (FEM, FDM, FVM, IBM, overset, cut-cell, DG, spectral element), and every downstream choice (element type, quality envelope, BL treatment, export format) is constrained by that declaration.

The bundle is **OSS-first**. No commercial mesher is in the loop. Wrapped tools: Gmsh, Netgen, cfMesh, snappyHexMesh, blockMesh, MMG/ParMmg, TetGen, CGAL, libigl, OpenCASCADE/pythonOCC, FreeCAD, CadQuery, build123d, meshio, trimesh, TIOGA (overset, research-grade), AMReX EB (cut-cell), Basilisk.

The bundle is **falsifiable**. Every problem spec carries hypotheses and a quality envelope expressed as numbers. The scaffold's `verification.json` either meets them or doesn't. The analysis skill is adversarial by default.

## Pipeline

Four skills, chained by artifact contracts:

```
literature-survey-mesh   →   mesh-knowledge-base.md
mesh-problem-spec        →   problem-spec.md
mesh-scaffold            →   mesh_run/ + verification.json
mesh-analysis-report     →   analysis-report.md
```

| Skill | Purpose | Output artifact |
|---|---|---|
| `literature-survey-mesh` | Operational survey of CAD/meshing literature, with method-aware taxonomy | `mesh-knowledge-base.md` |
| `mesh-problem-spec` | Force a falsifiable, method-aware spec | `problem-spec.md` |
| `mesh-scaffold` | Generate the Python pipeline wrapping the chosen OSS tools | `mesh_run/` package + `verification.json` |
| `mesh-analysis-report` | Adversarial audit of mesh quality, BL coverage, topology, and reference comparison | `analysis-report.md` |

Artifact filenames are inter-agent contracts. `verification.json` (with `verification.schema.json`) is the non-negotiable handoff between scaffold and analysis.

## Supported target methods

| Method | Primary tools | Reference strategy |
|---|---|---|
| FEM (conforming tet/hex) | Gmsh → FEniCSx / MFEM / NGSolve | MMS on discrete operator; GCI on benchmark |
| FDM (Cartesian structured) | Custom + meshio | MMS on FD stencil; GCI |
| FVM hex-dominant / polyhedral | cfMesh / snappyHexMesh → OpenFOAM | OpenFOAM `checkMesh` envelope; GCI on canonical case |
| IBM (immersed boundary) | Gmsh / Cartesian + STL → AMReX or custom | Sphere drag at Re=100; analytic geometry |
| Overset (Chimera) | Two grids + TIOGA → OpenFOAM overset | Couette analytic; overlap mass-conservation |
| Cut-cell (embedded boundary) | AMReX EB / Basilisk | Sphere drag; small-cell stability |
| DG / SE | Gmsh high-order → MFEM | MMS on high-order operator |

## Installation and use

This bundle is consumed by Claude — drop the `skills/` directory contents into your Claude environment's user-skill path. Skills auto-trigger from natural-language requests matching their `description` patterns.

For ClawHub-style distribution, the tarball-packaged form of this repository is the unit; see `docs/architecture.md`.

The generated mesh-pipeline code requires (depending on the chosen backend):

- Python 3.11+
- `gmsh`, `meshio`, `trimesh`, `numpy`, `scipy`
- `cadquery` and/or `build123d` (parametric CAD)
- OpenFOAM (cfMesh, snappyHexMesh, blockMesh) — ARM users see Apple Silicon notes in `docs/architecture.md`
- FEniCSx (FEM target)
- AMReX (cut-cell or IBM target)
- MMG binaries (`mmg3d_O3`) on `PATH` for remeshing path

All dependencies are open source. No commercial license required.

## Indian-context layer

When India/Bharat trigger words appear (NAL, IISc, IITM, IITB, IITH, ANRF, DST, MeitY, CSIR-4PI, "frugal", "low-cost cluster"), the bundle shifts to:

- Geometries from agri/healthcare/outdoor deployment (drip-irrigation channel, low-Re biomedical pipe, monsoon-stormwater drain, small wind turbine)
- Hardware sized for a single workstation or commodity cluster
- ARM/Apple-Silicon-clean tool choices (Gmsh, cfMesh, MMG, AMReX); snappyHexMesh ARM caveats flagged
- Indian-group references where available

## Cross-bundle synergies

| With | Synergy |
|---|---|
| **DhyutimaanPI** | PINN problem-specs that need a body-fitted reference domain pull meshes from DhyutimaanMesh; conversely, a PINN can serve as the reference for hard-to-solve regions where FEM/FVM grid convergence is expensive |
| **DhyutimaanCV** | 3DGS / NeRF reconstructions feed STL surfaces to DhyutimaanMesh (real-to-sim); DhyutimaanMesh hands the meshed digital twin to physics-based simulation |
| **DhyutimaanRobotics** | Differentiable / GPU simulators (Brax, Warp, AMReX) consume DhyutimaanMesh outputs for terrain, swarm-arena, and contact-rich environments |
| **DhyutimaanComplexity** | Network topologies on meshed manifolds; reaction-diffusion on adaptive meshes via MMG |
| **DhyutimaanForensics** | Provenance discipline (tool versions, git commit, spec hash) is mirrored 1-for-1 in `verification.json` |

## Versioning

DhyutimaanMesh follows semantic versioning. v0.1.0 ships:

- All four skill files
- Four worked examples (Poisson MMS FEM, lid-driven-cavity FVM, sphere-in-channel IBM, Couette overset)
- Architecture, quality-metrics, and cross-bundle-synergies docs
- Provenance discipline in `verification.json`

v0.2.0 roadmap:

- Hardened TIOGA path with a second worked overset example
- Anisotropic MMG remeshing worked example
- High-order DG meshing path via Gmsh's `Mesh.HighOrder*` controls
- Multi-tool cross-check automation in `mesh-analysis-report`

## License

MIT. See `LICENSE`.

## Citation

If this bundle contributes to your work, cite via `CITATION.cff` or the Zenodo DOI listed in `.zenodo.json` (once minted).

## Authorship

DhyutiLABS, by Rahul. Contact via the DhyutiLABS GitHub organisation.
