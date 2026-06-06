# Changelog

All notable changes to DhyutimaanMesh will be documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/).

## [0.1.0] — 2026-06-06

Initial release.

### Added

- Four-skill pipeline: `literature-survey-mesh`, `mesh-problem-spec`, `mesh-scaffold`, `mesh-analysis-report`
- Method-aware spec contract — every problem-spec must declare one of FEM/FDM/FVM/IBM/Overset/CutCell/DG/SE
- Quality envelope discipline with 99th and 99.9th percentile reporting (not just max/mean)
- Falsifiable hypothesis enforcement in every spec
- Reference strategy required: analytic / MMS / GCI / benchmark, with asymptotic-range check before reporting GCI
- `verification.json` schema with provenance (tool versions, host arch, git commit, spec hash, scaffold hash, wall-time, peak memory)
- Four worked examples covering FEM, FVM, IBM, and overset subdomains
- Cross-bundle synergies documented with DhyutimaanPI, DhyutimaanCV, DhyutimaanRobotics, DhyutimaanComplexity, DhyutimaanForensics
- Indian-context trigger words shift defaults to agri/healthcare/outdoor geometries, ARM/Apple-Silicon-clean tool choices, single-workstation hardware budgets
- Cross-tool / cross-distribution quality audit guidance in `mesh-analysis-report` Step 9

### Audit findings carried into v0.1.0

The proposed audit pass surfaced these items, all addressed in v0.1.0 unless noted:

| Dimension | Status |
|---|---|
| Method-aware spec contract | Addressed — section 2 of problem-spec is mandatory |
| Falsifiable hypotheses | Addressed — section 7 enforced; bad/good examples in skill |
| Quality envelope reported with tail percentiles | Addressed — 99.9th percentile required by `mesh-scaffold` and `mesh-analysis-report` |
| Reference strategy enforced | Addressed — section 6 mandatory; "no reference" requires explicit user assent |
| Provenance in verification.json | Addressed — schema in `docs/verification.schema.json` requires it |
| Cross-distribution evaluation | Addressed — `mesh-analysis-report` Step 9 mandates a second-tool check |
| Frameworks wrapped, not reimplemented | Addressed — explicit in scaffold and AGENTS.md |
| Worked examples per subdomain | Addressed — four examples (FEM/FVM/IBM/overset); cut-cell and DG/SE deferred to v0.2.0 |
| Adversarial analysis | Addressed — `mesh-analysis-report` skill structured around it |
| Calibration / asymptotic-range check | Addressed — GCI requires explicit asymptotic-range verification |
| Indian-context layer | Addressed — additive in every skill |
| Cross-bundle synergies | Addressed — `docs/cross-bundle-synergies.md` |
| Apple Silicon / ARM notes | Addressed — `docs/architecture.md` and `mesh-scaffold` |
| Runtime / hardware budgets | Partial — cell-count budgets in `quality-metrics.md`; v0.2.0 will add wall-time benchmarks |
| Safety envelopes | N/A for meshing (this is a Robotics concern) |
| Demonstration-data hygiene | N/A; cross-referenced from Forensics in synergies doc |
| YAML frontmatter validity | Validated — all four skills parse with PyYAML safe_load |
| Description char limit | Validated — all under 1024 chars |
| SKILL.md line limit | Validated — all under 500 lines (max 335) |
| Numerical-stability notes (FP32 vs FP64) | Deferred to v0.2.0 — note added to roadmap |
| Multi-tool quality consensus automation | Deferred to v0.2.0 — manual cross-check in v0.1.0 |

## [0.2.0] — planned

### Planned

- **Hardened TIOGA path** with a second worked example (rotor-in-box dynamic overset) and cross-validation against OpenFOAM-native overset on the same geometry
- **Anisotropic MMG remeshing** worked example with a Hessian-based metric driven by a coarse-mesh solver pass
- **High-order DG meshing** via Gmsh's `Mesh.HighOrder*` controls, with an MFEM worked example
- **Cut-cell worked example** completing the subdomain coverage
- **Multi-tool cross-check automation** in `mesh-analysis-report` — automatic re-scoring of any Gmsh-produced mesh through OpenFOAM `checkMesh` and vice versa
- **Wall-time and peak-memory benchmarks** captured systematically in `verification.json` for the four worked examples on a reference Apple Silicon M-series workstation
- **Numerical-stability notes** — FP32 vs FP64 effect on mesh-quality metrics; documented threshold-sensitive cases
- **Optional `clawhub` publish manifests** for the four skills
