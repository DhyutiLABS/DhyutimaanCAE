# Cross-Bundle Synergies

DhyutimaanMesh is the geometry-and-discretisation backbone of the DhyutiLABS family. This doc names the concrete handoffs.

## With DhyutimaanPI

**Mesh → PINN**

- A PINN problem-spec that targets a non-trivial geometry can pull a meshed domain (and its boundary tags) from DhyutimaanMesh to use as the collocation sampling region. The mesh's geometry is the ground truth; the PINN sampling uses the mesh's boundary primitives, not the mesh elements directly.
- The mesh's GCI-converged reference solution (FEM/FVM on L2) is a natural reference for PINN verification, replacing or augmenting MMS.

**PINN → Mesh**

- A trained PINN can serve as a *cheap* approximate solver inside a residual-driven mesh adaptation loop. The PINN residual at quadrature points becomes the local error indicator that MMG's metric field consumes.
- Residual physics: a PINN trained on a coarse-mesh PDE residual can correct under-resolved regions, with the mesh providing the physical domain and the PINN providing a sub-grid model.

**Failure modes shared**

- Both bundles enforce falsifiable hypotheses with explicit thresholds.
- Both bundles treat MMS as a non-negotiable when no analytic reference exists.

## With DhyutimaanCV

**CV → Mesh (real-to-sim)**

- 3D Gaussian Splatting (3DGS) and NeRF reconstructions emit dense surface representations. DhyutimaanCV exports a clean watertight STL; DhyutimaanMesh consumes it as the geometry source for the `mesh-problem-spec` skill's "imported STL" path.
- The handoff includes a defeaturing-tolerance estimate from CV (reconstruction noise floor) that the spec consumes directly.

**Mesh → CV**

- A solved physics field on the meshed digital twin (temperature, flow, deformation) re-projects to texture / displacement for the CV reconstruction, closing the loop.

**Failure modes shared**

- Watertightness and manifoldness checks are mirrored. DhyutimaanCV's reconstruction-time check and DhyutimaanMesh's pre-mesh check use the same trimesh-based Euler-characteristic computation.

## With DhyutimaanRobotics

**Mesh → Robotics**

- Differentiable / GPU simulators (Brax, Warp, dflex, AMReX) consume meshes for terrain, swarm-arena floors, obstacle fields, contact surfaces. DhyutimaanMesh's `export.py` adds a Robotics-target export that writes to the simulator's native format (e.g. URDF + meshes, MuJoCo XML + STLs).
- Indian-context geometries (agri-robotics fields, healthcare-clinic floor plans, MSME workshop layouts) flow from DhyutimaanMesh as the geometric substrate.

**Robotics → Mesh**

- A robot's perception module's reconstructed environment STL flows in as a geometry source. The mesh-then-resimulate loop closes here.

**Failure modes shared**

- Both bundles use TIOGA-style overset / multi-body coordination at the moving-component boundary. The connectivity audit is the same logic.
- Both bundles share the "demonstration-data hygiene" discipline from DhyutimaanForensics for any data-driven scaffolds (learned mesh-quality estimators, learned signed-distance fields).

## With DhyutimaanComplexity

**Mesh → Complexity**

- Reaction-diffusion and pattern-formation studies on non-trivial manifolds need a meshed domain. DhyutimaanMesh produces the simplicial complex; DhyutimaanComplexity reads it as a graph (cells = nodes, faces = edges).
- Adaptive remeshing (MMG metric driven by local Lyapunov exponents from DhyutimaanComplexity) is a planned v0.2.0 worked example.

**Complexity → Mesh**

- A complex-network topology can serve as the input "skeleton" for procedural CAD via build123d, producing a meshed lattice or scaffold structure for downstream simulation.

## With DhyutimaanForensics

**Provenance discipline**

- DhyutimaanForensics's `verification.json` provenance discipline (tool versions, host arch, git commit, content hashes) is mirrored 1-for-1 in DhyutimaanMesh's `verification.json`. Cross-bundle audit tooling can ingest either schema.

**Surface and texture provenance**

- For real-to-sim pipelines that chain DhyutimaanCV → DhyutimaanMesh → solver, the geometric provenance (reconstruction quality, defeaturing tolerance, mesh quality envelope) becomes a chain-of-custody record that DhyutimaanForensics-style attestation can sign.

**Failure modes shared**

- Cross-distribution evaluation is mandatory in both bundles. Forensics enforces it on detector accuracy; Mesh enforces it on quality-metric consistency across tools.

## Summary table

| Bundle | Hands DhyutimaanMesh | Receives from DhyutimaanMesh |
|---|---|---|
| DhyutimaanPI | A PDE + sampling primitives; a PINN-as-coarse-solver | A meshed domain + GCI reference; an adaptive-metric field |
| DhyutimaanCV | A watertight STL + defeaturing tolerance | A meshed digital twin |
| DhyutimaanRobotics | A reconstructed env STL; a contact / terrain need | A solver-native mesh export + connectivity for overset |
| DhyutimaanComplexity | A network skeleton; a local error indicator | A simplicial complex; an adaptive remeshing pass |
| DhyutimaanForensics | Provenance-discipline expectations | A `verification.json` honoring the same schema conventions |
