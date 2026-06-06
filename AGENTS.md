# AGENTS.md — DhyutimaanMesh

Guidance for agentic runtimes (Claude Code, Claude.ai, Cursor, Antigravity/Google, ClawHub) consuming the DhyutimaanMesh bundle.

## Purpose

DhyutimaanMesh is a four-skill pipeline for CAD-to-mesh workflows in CFD and computational mechanics, using only open-source software. Skills are chained by named markdown / JSON artifacts.

## Skill chain

```
literature-survey-mesh   →   mesh-knowledge-base.md
                                       ↓
                              mesh-problem-spec
                                       ↓
                                problem-spec.md
                                       ↓
                                 mesh-scaffold
                                       ↓
                              mesh_run/ + verification.json
                                       ↓
                              mesh-analysis-report
                                       ↓
                              analysis-report.md
```

## Trigger discipline for agents

- Trigger `literature-survey-mesh` when the user asks for prior work, state of the art, or grounding on CAD/meshing. Do not trigger for solver internals or post-processing.
- Trigger `mesh-problem-spec` when the user wants to frame, specify, or set up a meshing task. The spec must declare a target discretisation method (FEM/FDM/FVM/IBM/overset/cut-cell/DG/SE); if missing, force the question.
- Trigger `mesh-scaffold` only when `problem-spec.md` exists. Do not hallucinate a spec.
- Trigger `mesh-analysis-report` only when a `mesh_run/outputs/` directory with `quality.csv` and `verification.json` exists. Do not fabricate numbers.

The four skills are independent and should never auto-chain. Each writes its artifact; the user decides when to invoke the next.

## Artifact contracts

| Artifact | Producer | Consumer | Schema |
|---|---|---|---|
| `mesh-knowledge-base.md` | `literature-survey-mesh` | `mesh-problem-spec` | Markdown sections per the skill's `## Output` block |
| `problem-spec.md` | `mesh-problem-spec` | `mesh-scaffold`, `mesh-analysis-report` | 10 numbered markdown sections; downstream skills parse headers |
| `mesh_run/outputs/quality.csv` | `mesh-scaffold` | `mesh-analysis-report` | One row per cell or aggregated; columns per quality metric in spec section 5 |
| `mesh_run/outputs/verification.json` | `mesh-scaffold` | `mesh-analysis-report` | Schema in `docs/verification.schema.json` |
| `analysis-report.md` | `mesh-analysis-report` | next iteration's `mesh-problem-spec` | 11 numbered markdown sections |

Headers and filenames are stable contracts. Do not rename.

## Frameworks-wrapped principle

All meshing and CAD tools are wrapped, not reimplemented. The scaffold's job is orchestration, provenance capture, and quality assertion. The skills should never generate from scratch what Gmsh, cfMesh, snappyHexMesh, MMG, TIOGA, or AMReX EB already do.

Concretely:

- Python bindings (`gmsh`) where available
- Subprocess + dictionary file (`cfMesh`, `snappyHexMesh`, `MMG`) where Python bindings are absent
- Native input files (AMReX, Basilisk) where the tool is C++-centric

## Provenance discipline

Every `verification.json` carries:

- Tool versions (Gmsh, meshio, OpenFOAM, AMReX, MMG, etc.) as resolved at run time
- Host architecture (x86_64 / arm64)
- Python version
- Git commit of the scaffold repo (short SHA)
- SHA-256 of the consumed `problem-spec.md`
- Wall-time and peak-memory for each pipeline phase

The analysis skill verifies these fields exist. Missing provenance is a hard finding.

## Indian-context triggers

The bundle responds to India/Bharat trigger words by shifting defaults: geometry choices toward agri/healthcare/outdoor relevance, hardware footprint toward single-workstation viability, tool choices toward ARM/Apple-Silicon-clean paths. This is additive — global rigor is unchanged.

## Cross-bundle handoffs

| Origin → DhyutimaanMesh | DhyutimaanMesh → consumer |
|---|---|
| DhyutimaanCV (3DGS/NeRF) ships STL → meshed for solver | DhyutimaanRobotics consumes the mesh for sim |
| DhyutimaanPI ships a hypothesis-driven PDE → mesh is the reference domain | DhyutimaanPI uses the GCI as PINN comparison reference |
| DhyutimaanComplexity ships a manifold or domain → mesh provides the discrete topology | |

## Anti-patterns for agents

- Triggering scaffold without a spec
- Triggering analysis without a `verification.json`
- Auto-chaining skills without user assent
- Reimplementing what an OSS mesher does
- Reporting quality metric maxima only (the 99.9th percentile is where failure modes hide)
- Treating GCI outside the asymptotic range as conclusive
- Claiming watertightness from input STL only — recheck on the final mesh

## Skill development

Skills follow the `skill-creator` constraints:

- Description: ≤ 1024 characters, single-line, no nested colons in unquoted YAML values
- SKILL.md: ≤ 500 lines
- One worked example per skill (in the SKILL.md or in `examples/`)
- Anti-patterns section mandatory

Validate before declaring work complete.

## Distribution

Packaged as a tarball under DhyutiLABS GitHub org; optionally via ClawHub.

For ClawHub:

```
clawhub skill publish DhyutimaanMesh/skills/literature-survey-mesh
clawhub skill publish DhyutimaanMesh/skills/mesh-problem-spec
clawhub skill publish DhyutimaanMesh/skills/mesh-scaffold
clawhub skill publish DhyutimaanMesh/skills/mesh-analysis-report
```

Versions are pinned in `CITATION.cff` and `.zenodo.json`.
