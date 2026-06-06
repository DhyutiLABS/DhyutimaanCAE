---
name: mesh-analysis-report
description: Analyze the output of a CAD-to-mesh pipeline run and produce a structured report comparing results against the problem spec's quality envelope, hypotheses, and success criteria. Use this skill whenever the user has finished generating a mesh and wants to know "is this mesh good enough?", asks to "analyze", "review", "evaluate", "audit", or "report on" mesh results, has a `quality.csv` / `verification.json` / `outputs/` folder from a previous mesh run, or wants ablation comparisons across mesh runs (different tools, different resolutions, different boundary-layer settings). Be adversarial — flag quality metrics gamed by averaging, BL coverage that looks fine in aggregate but is sparse in critical regions, GCI that is computed outside the asymptotic range, orphan cells in overset, sliver elements hidden under aggregate stats, and hypothesis confirmations that look like coincidence. Output is `analysis-report.md`.
---

# Mesh Analysis & Report

This skill is the adversarial reader of mesh-generation output. Its job is not to celebrate that the mesh exists — it is to figure out whether the mesh will hold up under the solver and the loading regime, and to surface what *didn't* work as clearly as what did.

A good mesh analysis is suspicious by default. Mean quality can hide a tail of bad cells, boundary-layer coverage can be fine globally but absent in the stagnation region, GCI can be computed without checking the asymptotic range, "watertight" can mean closed at machine precision but with non-manifold edges, and a "successful" hypothesis can be confirmed by a measurement that was never going to fail. This skill exists to catch those patterns before the mesh is handed to a solver run that wastes a week of compute.

## When to use this skill

Trigger this skill when:

- A mesh-generation run has produced `quality.csv`, `verification.json`, mesh files, and/or plots
- The user asks "is this mesh good?" or "did the mesh work?"
- The user wants to compare runs (e.g., snappyHexMesh vs cfMesh on the same STL; L1 vs L2; Gmsh tet vs Netgen tet)
- The user wants a writeup of a completed meshing experiment

Do not trigger this skill for active mesh-generation questions (use `mesh-scaffold`), problem framing (`mesh-problem-spec`), or literature questions.

## Inputs

Expected inputs from a completed mesh run:

- `problem-spec.md` — the original spec; the report must compare results against its quality envelope, hypotheses, and success criteria
- `mesh_run/outputs/quality.csv` — per-cell or aggregated quality metrics
- `mesh_run/outputs/verification.json` — final metrics and hypothesis verdicts from the scaffold's verification harness, including tool versions and provenance
- `mesh_run/outputs/mesh*.msh` (or polyMesh/, AMReX plotfile) — the mesh files
- `mesh_run/outputs/plots/*.png` — quality histograms, BL coverage maps, GCI plots

If any of these are missing, ask the user before fabricating analysis. Specifically: never produce numbers that aren't grounded in the input files.

## Output: `analysis-report.md`

Always produce a single markdown file with this exact structure:

```markdown
# Mesh Analysis Report: <run name>

## 1. Headline
One sentence: did the run meet its success criteria, yes or no, with the key numbers (e.g., GCI on Cd = 2.7%, max non-orthogonality 62°, quality envelope satisfied).

## 2. Provenance and reproducibility
Tool versions, host arch, git commit, spec hash, scaffold hash. Flag anything missing.

## 3. Hypothesis verdicts
A table with one row per hypothesis from the spec. Columns:
- Hypothesis (short)
- Predicted threshold
- Measured value
- Verdict (confirmed / refuted / inconclusive)
- Confidence note (e.g., "single mesh", "MMS reference", "GCI outside asymptotic range")

## 4. Quality envelope audit
For each metric in spec section 5:
- Threshold from spec
- Measured: min, mean, median, 95th, 99th, 99.9th percentile, max
- Pass/fail
- Spatial distribution: where are the worst cells located? (interior, boundary, near a sharp edge, in concave regions, at refinement interfaces)
- Reading: are the bad cells in a region the solver will probe heavily?

## 5. Boundary layer audit (if applicable)
- Number of layers achieved vs requested
- Growth ratio achieved vs requested
- y+ distribution: fraction within target band, where the outliers sit
- Collapse / addition failures: where, how many cells
- Coverage on critical surfaces (stagnation, separation, no-slip walls)

## 6. Geometry and topology audit
- Watertightness, manifoldness
- Feature-edge capture (sharp edges preserved? filleted edges treated correctly?)
- Cell count at each refinement level
- For overset: orphan cells, fringe cells, donor-receptor map sanity
- For cut-cell / IBM: smallest EB cell volume fraction, signed-distance accuracy near surface

## 7. Reference comparison
The reference strategy from spec section 6, evaluated:
- Analytic geometry checks: relative error per quantity
- MMS: discrete-operator L2 error vs analytic; mesh dependence
- GCI: observed order, asymptotic-range check, GCI on the fine grid
- Benchmark: comparable quantity, relative error against published reference

## 8. Failure-mode audit
Walk through each failure mode listed in spec section 9 and report whether it occurred.

## 9. What this run does NOT tell us
Honest limitations.

## 10. Cross-tool / cross-distribution check
If multiple meshing tools or refinement levels were run, summarize the comparison. If only one tool was used, recommend a second (e.g. Gmsh's quality field vs OpenFOAM's checkMesh on the same volume) to expose tool-specific blind spots.

## 11. Recommended next steps
2–4 concrete next experiments, each phrased as a hypothesis the next iteration could test.
```

## Workflow

### Step 1: Read all inputs

Load `problem-spec.md`, `quality.csv`, and `verification.json`. View any plots in the outputs directory. If you have a code-execution environment, load the CSV with pandas and *actually compute* the summary statistics — don't eyeball the file.

If `quality.csv` has columns the spec didn't promise (or is missing columns the spec did promise), note this as a data-quality issue at the top of the report.

If `verification.json` is missing fields the scaffold's schema requires (tool_versions, provenance), flag it. A mesh without provenance is unreproducible.

### Step 2: Compute quality signals

From `quality.csv`, for each metric:

- Compute min, max, mean, median, std, 95th, 99th, 99.9th percentile
- Identify cell IDs (or coordinates) of the worst 0.1% of cells per metric
- Cross-reference worst cells with geometric features (sharp edges, BL prisms, refinement interfaces)
- Plot a histogram if not already plotted; include in report by reference

Flag if:

- The 99.9th percentile exceeds the envelope by more than 10× the max-mean gap — there's a tail-failure mode
- The worst cells cluster in a critical region (stagnation point, no-slip wall, leading edge of an airfoil) — the global pass is misleading
- The histogram is bimodal — likely two element-quality regimes (interior vs BL prisms, or different refinement levels)
- More than 0.01% of cells fail a hard threshold — many solvers degrade non-linearly with bad cell count

### Step 3: Audit the boundary layer (if applicable)

If the spec called for prism BL layers:

- Compute the achieved layer count per wall cell. The histogram should peak at the requested count; a long left tail means collapse.
- Compute the growth ratio per layer; deviation from the requested ratio indicates layer-addition struggle.
- y+ band coverage: compute the fraction of wall cells with y+ in the target band. Fraction below the target is the headline number.
- Identify regions of collapse: typically concave corners, sharp edges, regions where two BL surfaces meet.

For IBM:

- Number of refinement-band cells across the estimated BL thickness $\delta$ at the target Re — under 3 is danger.
- Signed-distance error near the surface — sample at random points on the STL and compare to computed SDF.

### Step 4: Audit topology

- Watertightness: re-check Euler characteristic on the exported mesh, not just the input STL. Tools can introduce sub-element gaps.
- Manifoldness: check for non-manifold edges or vertices via meshio or trimesh.
- Feature-edge capture: for STEP-derived meshes, compare the mesh's sharp-edge angle distribution to the geometry's analytic edge list.
- Boundary tags: every named boundary in spec section 4 should have a non-zero cell count in the exported mesh. Missing tags → silent solver failures.

For overset:

- Orphan-cell count (cells in the fringe that have no valid donor): should be zero. Any positive count is a setup failure.
- Fringe-cell count and ratio to total: too thin means insufficient overlap, too thick means wasted cells.
- Donor-receptor map: spot-check that receptor cells are interior to a donor grid, not on its boundary.

For cut-cell:

- Smallest EB cell volume fraction: below 0.001 means a time-step penalty if not merged; below machine precision means a numerical problem.
- Signed-distance smoothness: jumps in SDF across cell faces indicate STL artifacts.

### Step 5: Audit the reference comparison

From `verification.json`:

- **Analytic geometry**: report relative errors. If above ~1%, the mesh is geometrically inaccurate even before discretisation; investigate before trusting the rest.
- **MMS**: report L2 error of the discrete operator on the manufactured field. If MMS error is much larger than expected for the element order, the mesh is degenerate even if quality metrics look fine.
- **GCI**: compute observed order $p$ from the three levels. Compare to expected order (typically 2 for second-order schemes on smooth problems). **If $|p - p_\text{expected}| > 0.3 p_\text{expected}$, flag GCI as outside the asymptotic range and unreliable.** Then report the GCI value, but with the asymptotic-range caveat.
- **Benchmark**: relative error against the published reference. Note the reference's own uncertainty if known.

### Step 6: Compare to hypotheses

For each hypothesis in the spec:

1. Find the measurement it predicts (read carefully — many hypotheses specify a specific region or condition).
2. Compute or extract that measurement from the verification output.
3. Apply the threshold.
4. Mark confirmed, refuted, or inconclusive.

Mark "inconclusive" — not "confirmed" — when:

- The measurement is in the right direction but the threshold wasn't quantitative ("better")
- A single mesh isn't enough to support the claim (mesh-generation is deterministic but sensitive to seed-like inputs e.g. node ordering, Gmsh algorithm choice)
- The measurement was made on a region the hypothesis didn't specify
- The GCI was outside the asymptotic range
- The reference itself has uncertainty larger than the measured gap

Be willing to refute hypotheses the user is rooting for. The whole point of a falsifiable hypothesis is that it can fail.

### Step 7: Failure-mode audit

For each failure mode in spec section 9, check the corresponding signal:

- **Sliver elements / aspect-ratio explosion**: look at the 99.9th percentile of aspect ratio and min scaled Jacobian.
- **BL collapse**: fraction of wall cells with achieved layer count below requested.
- **Non-orthogonality spike at refinement interfaces**: spatial map of non-orthogonality; clusters at refinement transitions are the signature.
- **Orphan cells / fringe-donor failures**: TIOGA / overset connectivity report.
- **Watertightness loss after Boolean operations**: re-check Euler characteristic on the final mesh.
- **IBM band too thin for target Re**: estimate $\delta$ from Re, compare to cell count across $\delta$.
- **Hanging-node count**: if FEM solver requires conforming, hanging-node count > 0 is a hard fail.

For each flagged issue, give one concrete remediation (e.g., "increase max-aspect-ratio constraint and re-mesh", "switch from frontal-Delaunay to Delaunay 3D for robustness on this geometry", "refine the BL transition zone by one level", "reduce the cfMesh `maxCellsPerProcess` to avoid the layer-merge edge case").

### Step 8: Write the limitations section

This is the section users skip and the section that matters most. Include at minimum:

- Single-mesh vs multiple-mesh seeds (Gmsh outputs depend on algorithm and seed-like inputs; one mesh is one sample)
- Reference quality: MMS gives perfect ground truth for the discrete operator only; FD/FEM/benchmark refs have their own error
- Evaluation grid resolution for residual / error fields
- Whether the solver was actually run, or whether only mesh-side metrics were checked
- What the success criterion actually tested — pass on quality envelope is necessary but not sufficient for solver accuracy
- For overset and cut-cell: OSS tooling is research-grade; report tool version and known issues from the maintainer's issue tracker

### Step 9: Cross-tool / cross-distribution check

A mesh that looks good in Gmsh's quality scoring may be flagged by OpenFOAM's `checkMesh`. The two tools define metrics differently and a true robust mesh passes both. If the run only used one tool, recommend re-running quality through a second:

- Gmsh mesh → also run through OpenFOAM's `checkMesh` after conversion
- cfMesh / snappyHexMesh mesh → also run through meshio's quality checks
- Any tet mesh → run through MMG's `-O` mode for an independent quality audit

Document the cross-check result. If both tools agree, robustness goes up.

### Step 10: Propose next experiments

Each suggestion should be a hypothesis the next iteration's `problem-spec.md` could test. Don't recommend "use a finer mesh" — recommend specific interventions with specific predictions.

Good suggestion: "Switch from Gmsh frontal-Delaunay to HXT on this geometry and re-mesh; predict 99.9th-percentile aspect ratio drops below 50 and BL collapse fraction drops below 1%."

Bad suggestion: "Refine the mesh."

### Step 11: Write `analysis-report.md`

Write the file to the working directory. Keep the report tight — aim for 600–1200 words plus the hypothesis table and the quality envelope table. The user should be able to read it in 3–5 minutes and walk away knowing exactly what happened, whether the mesh is safe to hand to the solver, and what to do next.

## Adversarial reading checklist

Before declaring the analysis complete, ask yourself:

- Is the headline supported by a number in the inputs, or am I inferring it?
- Did I check the 99.9th percentile of every quality metric, not just the max/mean?
- Did I check that the worst cells are not in a critical region?
- Did I audit every failure mode the spec listed?
- Did I check whether GCI was computed inside the asymptotic range?
- Did I report at least one thing that didn't work, or that we can't conclude?
- Did I verify that boundary tags survived the export?
- Did I avoid praising the mesh? (Praise is the user's job; this skill's job is to report.)

If any answer is no, fix it before handing over.

## Worked example output excerpt

Given a sphere-in-channel IBM run at Re=100 with Cd=1.10, hypothesis was Cd within 5% of 1.09:

> ## 1. Headline
> Run met its primary success criterion: Cd = 1.10 against Mittal et al. (2008) Cd = 1.09 (0.9% relative error). GCI on Cd across L0/L1/L2 = 2.7%, observed order 1.78. **Caveat: BL band is 3 cells across estimated $\delta$ at the equator, below the recommended minimum of 4; the prediction is currently insensitive but a higher-Re extension would be unreliable.**
>
> ## 4. Quality envelope audit
> Max non-orthogonality 62.4° (threshold 70°, pass). 99.9th percentile aspect ratio 47 (threshold 1000, pass). However, worst 12 cells cluster on the refinement-band-to-background interface, not in the bulk. Recommend explicit transition refinement.
>
> ## 5. Boundary layer audit
> ...

Note the report flags the BL-band thinness even though the run "succeeded" — that's the job. The 0.9% Cd error is real; the BL band is a latent failure mode that will surface at higher Re.

## Comparing multiple runs

If the user provides multiple run directories (e.g., `mesh_run_gmsh/`, `mesh_run_cfmesh/`, or three refinement levels), produce a *comparative* report with the same structure but add:

- A side-by-side table of final metrics
- A paired comparison for each hypothesis
- A GCI computation across the refinement levels if not already done
- An explicit statement about whether the comparison is fair (same geometry? same BL settings? same export format? same solver post-processing?)

If the runs are not directly comparable (different geometry tolerances, different BL settings, different solver coupling), say so. Don't average across incomparable runs.

## Anti-patterns

- **Reporting numbers you didn't load from a file.** Every quantitative claim must be traceable to an input.
- **Praising the mesh.** "The mesh resolved the geometry beautifully" is not analysis.
- **Reporting only max and mean of quality metrics.** The 99.9th percentile is where the failure mode hides.
- **Confirming a hypothesis on a single mesh without flagging mesh-generation sensitivity.** Gmsh algorithm choice changes the mesh; one mesh is one sample.
- **Treating quality envelope pass as solver success.** Envelope is necessary, not sufficient. The reference / GCI is what closes the loop.
- **Ignoring failure modes the spec listed.** They're there for a reason — audit each one.
- **Recommending "use a finer mesh" or "tune parameters."** Those aren't experiments.
- **Trusting watertightness from the input STL alone.** Re-check on the final mesh — tools can introduce sub-element gaps.
- **Reporting GCI without an asymptotic-range check.** Outside the asymptotic range, GCI is decoration.
