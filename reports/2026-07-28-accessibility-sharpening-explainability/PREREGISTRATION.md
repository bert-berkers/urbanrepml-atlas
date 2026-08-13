# Pre-Registration — What Accessibility Message-Passing Injects Into the Representation (Sharpening-as-Explainability)

**Campaign slug**: `2026-07-28-accessibility-sharpening-explainability`
**Date frozen**: 2026-07-28
**Author / shard**: `cool-lingering-dew` (dynamic-graph / `/niche` execution mode)
**Git commit that froze this file**: `d6e21fdb`
<!-- rule: leave __FILL_AT_COMMIT__ as-is for the freeze commit (clean). Replace it with the
     freeze commit sha in the FIRST RESULTS commit. Verify with:
     git log --diff-filter=A --format=%H -- reports/2026-07-28-accessibility-sharpening-explainability/PREREGISTRATION.md -->

**Status of foundations**: ✅ **RATIFIED `[decided-by-human:2026-07-28]`** — in-chat, same
evening, terminal cool-lingering-dew. The human ratified all six foundations as proposed,
with a generosity directive applied to scope (verbatim: *"for the rest be generous we have
all night to do this well with a [parallel agent doing a rerun"*): all secondaries, symmetry
checks, and all three ring-agg variants stay IN (F5 extended — linear remains the registered
anchor; exponential and flat are supplementary descriptive evaluations). Analysis-only
campaign: no model is trained, so the
`.claude/rules/novel-research-escalate-dont-default.md` **training** gate does not fire —
the *analysis* foundations were nevertheless surfaced and closed by the human, not an agent.

---

## Purpose (overview first — self-evident to a stranger)

A companion campaign already answered one question and closed it: does routing a neural
network's message passing along a **travel-accessibility graph** (walk/bike/drive networks,
weighted by how much destination mass each hexagon attracts) instead of along the plain
hexagonal grid make the resulting embeddings better at predicting liveability and urban
morphology? The answer was **no measurable improvement** — across five model objectives and
three random seeds, the accessibility arm and the grid ("lattice") arm scored the same to
within noise (`reports/2026-07-18-accessibility-ablation/`). The human read that result and
accepted it without surprise, while making a second point explicitly: *a signal our chosen
targets do not reward is not thereby an absent signal.*

This campaign asks the second question. **Not "does accessibility help the score?" but
"what does accessibility actually DO to the representation?"** Two networks trained
identically except for which graph carries information between hexagons produce two
different 64-dimensional embeddings of the same country. This document freezes, in advance,
exactly how we will characterise the difference between them: how much of the accessibility
embedding is *not* a simple linear re-description of the lattice embedding (the **residual
subspace**), whether that residual is a spatially coherent field or noise, whether
accessibility makes neighbouring hexagons look *more* different from each other or *less*
(**sharpening vs smoothing**), and how far each graph choice moves the representation away
from its own input (**representational similarity**). It is an explainability campaign, not
a benchmark: no cell can "win", and the deliverable is a description of the difference.

**The failure surface pre-registration closes here.** With 30 frozen embeddings on disk, five
objective families, three seeds, two target families, several plausible sharpness metrics and
several plausible similarity measures, the space of "interesting differences we could go
looking for" is large enough that *something* will look striking by chance. Enumerating every
cell now — and committing to report all of them, including the null and the boring ones —
makes a favourable-subset narrative impossible to construct afterwards. This matters more
here than in a benchmark campaign, because a characterisation study has no natural loser: the
temptation is not to hide a bad number but to *promote whichever difference photographed
best*.

### Terms defined at first use

- **Hexagon / H3 res9**: the unit of analysis — an H3 resolution-9 cell (~0.1 km²) tiling the
  Netherlands. Canonical BES-free grid, regenerated 2026-06-07:
  `data/study_areas/netherlands/regions_gdf/`.
- **306D concat (the input)**: the embedding fed to every model in the source campaign —
  AlphaEarth (64D, `A00..A63`) + Tessera (128D, `T00..T127`) + POI/hex2vec (50D,
  `hex2vec_0..49`, `filter_id: filter_v2`) + Roads/highway2vec (64D, `R00..R63`), each block
  z-scored. Artifact:
  `data/study_areas/netherlands/stage2_multimodal/concat/embeddings/netherlands_res9_2022_306D.parquet`
  — 482,706 × 306, `region_id` index, sidecar green (`data_vintage: 2022`, block_map,
  block_health nan=0/inf=0). This is the **only** input fabric in scope.
- **FullAreaUNet (the backbone)**: the Stage-2 model — a 3-level graph U-Net over H3
  resolutions [9, 8, 7], channel dims [64, 128, 256], all output heads projecting to 64D.
  `stage2_fusion/models/full_area_unet.py`.
- **Accessibility arm ("acc")**: within-level message passing runs over the mode-matched
  travel graph — res9 = walk, res8 = bike, res7 = drive — with `edge_weight` = gravity
  (building-density-weighted, percentile-pruned). Graphs regenerated onto the canonical grid
  2026-07-10.
- **Lattice arm**: identical backbone, identical input, identical hyperparameters; message
  passing runs over the uniform H3 ring-1 neighbourhood at every level with
  `edge_weight = 1.0`. **The graph is the only difference between the two arms.**
- **Objective row / seed-paired pair**: five objective families were trained in both arms.
  Each row is a *pair* of cells sharing an objective and a seed, differing only in the graph:
  `plain` (C1 acc ↔ C2 lattice), `gVAE` (C3 ↔ C4), `PVAE` (C5 ↔ C6), `NAG-S0` (C7 ↔ C8),
  `NAG-S1` (C9 ↔ C10). Each pair exists at seeds {42, 43, 44} → **15 seed-paired pairs**.
  Artifacts:
  `data/study_areas/netherlands/stage2_multimodal/abl_c{01..10}_{objective}_{arm}_s{42,43,44}/embeddings/netherlands_res9_2022.parquet`,
  each **482,706 × 64** (`unet_0..unet_63`, `region_id` index).
- **Row-alignment (verified, load-bearing)**: the SHA-256 of the `region_id` column is
  **identical across all 30 embeddings** (`753a552b27ce…`, verified on disk 2026-07-28) and
  every embedding is non-finite-free. Cross-arm differencing therefore requires **no reindex
  and no join** — row *i* of the acc arm and row *i* of the lattice arm are the same hexagon.
- **Residual subspace `R` (H-RES)**: fit a linear (ridge) map `f` from the lattice arm's 64
  dimensions to the accessibility arm's 64 dimensions, out-of-fold. The residual is
  `R = acc − f(lattice)`: the part of the accessibility embedding that **cannot** be obtained
  by linearly re-expressing the lattice embedding. `R` is a 482,706 × 64 matrix, one row per
  hexagon; `‖R‖` denotes its per-hexagon Euclidean norm (a scalar field over the country).
- **Local contrast (H-SHARP)**: a per-hexagon scalar measuring how different a hexagon's
  embedding is from its immediate (H3 ring-1) neighbours. Higher = locally sharper (more
  high-frequency spatial structure); lower = locally smoother. Computed identically on both
  arms so the paired difference is meaningful.
- **Linear CKA (H-CKA)**: Centered Kernel Alignment with a linear kernel — a similarity score
  in [0, 1] between two matrices over the *same* rows, invariant to rotation and isotropic
  scaling of either representation. CKA = 1 means the two representations are the same up to
  an orthogonal transform; CKA near 0 means they encode unrelated structure. Computed exactly
  (no subsampling) via the feature-space identity
  `CKA = ‖Yᵀ X‖_F² / (‖Xᵀ X‖_F · ‖Yᵀ Y‖_F)` on column-centered matrices.
- **Ring-agg (the non-learned smoother)**: a zero-parameter spatial smoother — each hexagon's
  306D input vector replaced by a weighted mean over its k=10 hexagon-ring neighbourhood.
  Three weighting variants persisted at
  `data/study_areas/netherlands/stage2_multimodal/abl_ring_agg_306D/embeddings/netherlands_res9_2022_{exponential,linear,flat}.parquet`,
  each 482,706 × 306. **Provenance caveat (declared, see §Governance clause 7)**: these three
  matrices carry **no `*.run.yaml` sidecar**; their identity rests on the 2026-07-18 campaign's
  run layout and filenames, not on a sidecar field.
- **Targets**: **leefbaarometer** (Dutch national liveability index) —
  `data/study_areas/netherlands/target/leefbaarometer/leefbaarometer_h3res9_2022.parquet`,
  131,194 hexes, sidecar `data_vintage: '2024'` with `extra.filtered_year: 2022`, i.e. the
  **LBM3 Meting-2024 release, 2022 score year**; and **RUDIFUN** (urban-morphology indices
  FSI/GSI/MXI/L/OSR) —
  `data/study_areas/netherlands/target/rudifun/rudifun_h3res9_2022.parquet`, 238,072 hexes,
  `data_vintage: '2022'`.
- **Join scope (mandatory caption line)**: against the 482,706-hexagon embedding grid the
  targets join at **27.2 % (leefbaarometer, 131,194 hexes)** and **49.3 % (RUDIFUN, 238,072
  hexes)**. Every target-based number and every target-based figure caption in this campaign
  carries its join-scope line verbatim. Embedding-only cells (H-RES-a, H-RES-c, H-SHARP,
  H-CKA, the visual battery) run on the **full 482,706-hexagon frame** and say so.
- **Δ (delta)**: throughout, a signed paired difference `acc − lattice` computed within a
  single objective row and a single seed. Positive means the accessibility arm is larger on
  that metric.
- **DEAD embedding**: an embedding whose training collapsed — near-zero variance in most
  dimensions and probe R² at or below zero. **C4 (gVAE × lattice) is a known DEAD embedding**
  at all three seeds (family R² −0.006 leefb / −0.002 RUDIFUN, sd ≈ 0.0002; receipt
  `reports/2026-07-18-accessibility-ablation/results/confirmatory_stats.md` §Anomalies #1 and
  `maps/2026-07-18/02_C4_collapse_receipt_3seeds.png`). Its handling rule is pre-declared in
  §Governance clause 6 — **`[blocked:human-decision]` F1**.

---

## Governance (the integrity contract)

1. **Commit-then-run (order gate).** This file is committed **before** the first analysis cell
   runs; its freeze sha (header) predates every result it governs
   (`.claude/rules/artifact-provenance.md` clause (c)). Producer scripts are committed before
   they run — each run's sidecar `git_hash` names the code that actually ran.
2. **All enumerated cells reported.** Every cell in §"The Full Grid" appears in the final
   report — nulls, ties, and prediction-contradicting values included. **This clause binds
   hardest in a characterisation campaign**: "the residual was structureless noise" and
   "accessibility neither sharpened nor smoothed" are *results*, published with the same
   prominence as any positive finding.
3. **No post-hoc cells, metrics, or targets.** No metric, arm, objective row, target, or
   figure family outside this document is added after results exist. A new question requires a
   new pre-registration. In particular: **no cross-fabric comparison.** Every claim is internal
   to the 306D-input embedding family; the 178D fabric is owned by a peer lane and is OUT OF
   SCOPE (`.claude/plans/2026-07-28-…ops.md` §Scope).
4. **Verdicts quote their rule.** Each hypothesis closes with a verdict quoting its own frozen
   decision branch verbatim, beside its evidence table.
5. **Confirmatory vs exploratory marked.** **H2** (H-RES-b), **H3** (H-RES-c), and **H4**
   (H-SHARP, primary metric = cell SHARP-1) are CONFIRMATORY — their tests are on evidence not
   yet observed. **H1** (H-RES-a, descriptive magnitude), cell SHARP-2 (robustness metric), and
   all three **H5** (H-CKA) cells are EXPLORATORY / DESCRIPTIVE and
   are **excluded from every Holm family**, declared here, before results exist. The visual
   battery is the pre-registered **primary practical-significance channel** and carries no
   statistics at all (clause 8).
6. **The DEAD-embedding rule (pre-declared, applies to C4 / the gVAE row).**
   `[decided-by-human:2026-07-28]` **F1** — the drafted proposal, pending the human's stamp:
   *analyse-and-report-with-caveat*. The gVAE row (C3 ↔ C4) runs every cell like any other
   row; every gVAE number is tagged **COLLAPSE-CAVEAT** in every table and caption; and the
   gVAE row is **excluded from the Holm correction families** (because a
   collapsed-vs-functional contrast measures collapse, not accessibility) while still being
   fully reported. The alternative considered and NOT chosen in the draft: exclude the gVAE
   row entirely (loses a genuine data point about what a graph swap does when one arm is
   degenerate). **Whichever the human picks is frozen here before any cell runs** — the
   forbidden move is deciding after seeing whether the gVAE row is interesting.
7. **Input-provenance honesty.** The 30 cell embeddings and the 3 ring-agg matrices carry
   **`run_info.json` only, no `*.run.yaml` sidecar** (verified on disk 2026-07-28). Their
   identity is therefore established by the run-directory layer
   (`specs/run_provenance.md`) plus this campaign's own on-disk verification (shape,
   `region_id` SHA, finiteness), **not** by the artifact-level sidecar gate
   (`.claude/rules/artifact-provenance.md` clause (a)). This is stated in the report's
   limitations, not buried. **Every NEW artifact this campaign writes gets a real
   `SidecarWriter` sidecar** (`utils/provenance.py`) with `seed` populated.
8. **The visual battery is a registered channel, not decoration.** Per the human's standing
   preference and the source campaign's ratified two-channel rule, **practical significance is
   adjudicated by the human's visual inspection of the map battery**, and there is **no numeric
   SESOI** (`[decided-by-human:2026-07-28]` **F4** — confirm the 2026-07-18 no-SESOI ruling carries
   to this campaign). The statistical channel is secondary: it says whether a difference is
   distinguishable from noise, never whether it matters.
9. **Retired numbers labeled.** None. This campaign reports no figure that supersedes a prior
   one; it consumes the 2026-07-18 results as *background*, and any 2026-07-18 number quoted
   here is quoted as-published with its source path.

### Locked protocol constants

| Constant | Value | Source |
|---|---|---|
| Study area / resolution / year | Netherlands / H3 res9 / 2022 | campaign scope |
| Embedding frame (embedding-only cells) | 482,706 hexes, `region_id` index, row-aligned across all 30 | verified on disk 2026-07-28 |
| Target frames | leefbaarometer 131,194 (27.2 % join) · RUDIFUN 238,072 (49.3 % join) | verified on disk 2026-07-28 |
| Seeds | `{42, 43, 44}` — every cell is seed-paired within a seed | inherited from the source campaign |
| Objective rows | 5 pairs: plain (C1↔C2), gVAE (C3↔C4), PVAE (C5↔C6), NAG-S0 (C7↔C8), NAG-S1 (C9↔C10) | inherited |
| Backbone / HPs | FullAreaUNet [9,8,7], dims [64,128,256], heads→64D, `num_convs=5`, lr 1e-2, 500 epochs — **all frozen, none re-trained here** | `reports/2026-07-18-accessibility-ablation/PREREGISTRATION.md` §Locked constants |
| CV scheme | 5-fold **spatial-block**, 10 km blocks, EPSG:28992 (Dutch RD New); fold seed = cell seed | `.claude/rules/domain-guardrails.md` CRS rider + source-campaign convention |
| Residual map estimator | `RidgeCV(alphas=[0.1, 1, 10, 100, 1000])`, α CV-selected per fold, per-fold `StandardScaler` on X and Y; residual taken **out-of-fold** | this document |
| Residual map direction | **lattice → acc** (primary); acc → lattice reported as a symmetry check | `[decided-by-human:2026-07-28]` **F3** |
| Probe (H-RES-b) | `DNNProbeRegressor`, `--num-layers 0` (single `nn.Linear`, no activation, MSE) = true linear regression | `CLAUDE.md` §Probe Infrastructure (DNN-probe-preferred) |
| Local-contrast primary metric | mean cosine distance from a hexagon's 64D embedding to its H3 ring-1 neighbours' embeddings | `[decided-by-human:2026-07-28]` **F2** |
| Local-contrast secondary metric | local Moran's I of the arm's PC1 over the ring-1 neighbourhood | `[decided-by-human:2026-07-28]` **F2** |
| CKA | linear CKA, column-centered, **exact on all 482,706 rows** (feature-space identity; no subsampling, so no subsample seed to declare) | this document |
| Ring-agg comparator for CKA-3 | **anchor**: `netherlands_res9_2022_linear.parquet` (k=10, linear weighting); **supplementary (F5 generous extension)**: exponential + flat variants evaluated descriptively; prediction (ii) is adjudicated on the linear anchor only | `[decided-by-human:2026-07-28]` **F5** |
| Spatial-autocorrelation neighbourhood | H3 ring-1, row-standardised weights, built via SRAI `H3Neighbourhood` | `.claude/rules/srai-spatial.md` |
| Moran's I inference | 999-permutation null, `seed = 42` | this document |
| Bootstrap | percentile bootstrap, `n_boot = 10000`, `seed = 42` | `scripts/science/stats.py` defaults |
| Paired test | paired permutation (sign-flip), `n_perm = 10000`, `seed = 42` | `scripts/science/stats.py` |
| Compute constraint | **CPU-side only** — a peer terminal owns the GPU for an all-night training campaign; no cell in this grid requires a GPU | `.claude/rules/training-discipline.md` serial discipline |

---

## Statistical-honesty declarations (REQUIRED — one row per §(d) field)

- **Seeds & seed-noise band**: seeds = `{42, 43, 44}`, three per objective row, every headline
  number reported as **mean ± sd over the three seeds**. The seed band ε is computed *per
  metric per objective row* as the across-seed sd of that row's Δ, and is reported
  **descriptively beside every Δ** — it is not converted into a decision threshold, because
  this campaign declares no numeric SESOI (see below). Reporting it is what makes a Δ smaller
  than its own seed wobble visible as such. — `stats.py: seed_band`
- **Decision thresholds in seed-sd units**: **none are used as gates.** Every Δ ships its
  calibration margin `|Δ| / ε` (`SeedBand.sd_units`) as a *reported descriptive column* so a
  reader can see at a glance whether an effect is large or small relative to seed noise. A Δ
  whose margin is below 1.0 is annotated "within seed noise" in the table regardless of its
  p-value. — `stats.py: seed_band`
- **CI on every headline number**: percentile bootstrap over the relevant hexagon frame,
  `n_boot = 10000`, `seed = 42`. Every reported point estimate that has a per-hexagon
  decomposition ships its interval. **Declared exception**: linear CKA is a whole-matrix
  statistic with no per-example decomposition, so a per-hexagon bootstrap is not defined for
  it; the H-CKA cells therefore report the **across-seed band (n=3) instead of a bootstrap
  CI**, and are declared EXPLORATORY/DESCRIPTIVE for exactly that reason. — `stats.py: bootstrap_ci`
- **Multiple-comparison correction**: **Holm–Bonferroni applied within each confirmatory
  family separately**, families declared now:
  - **Family A (H2 / H-RES-b, residual target-legibility)** — 10 contrasts = 5 objective rows × 2
    target families. If the DEAD-embedding rule F1 resolves to *analyse-with-caveat*, the gVAE
    row's 2 contrasts are **excluded from the family** (clause 6) → family size **8**; if F1
    resolves to *exclude*, family size is likewise 8. Either way the family is declared before
    results.
  - **Family B (H3 / H-RES-c, residual spatial structure)** — 5 Moran's-I permutation tests, one
    per objective row; gVAE excluded per clause 6 → family size **4**.
  - **Family C (H4 / H-SHARP, cell SHARP-1, local-contrast paired difference)** — 5 paired
    permutation tests, one per objective row; gVAE excluded per clause 6 → family size **4**.
  - **Excluded from every family** (declared): H1 (RES-a), cell SHARP-2, H5 (CKA-1/2/3), the visual
    battery, and the acc→lattice symmetry check. — `stats.py: holm`
- **Severity (per hypothesis)**: stated in each H block below — one line naming what the test
  would likely produce **if that hypothesis were false**. A characterisation test that could
  not have come out flat is not evidence; each severity line names the flat outcome explicitly.
  — (prose; discriminating tests via `paired_permutation` / permutation-null Moran's I)
- **SESOI**: **NO numeric SESOI is declared** — `[decided-by-human:2026-07-28]` **F4** (confirm the
  2026-07-18 Foundation-3 ruling carries to this campaign: *"sesoi is not that important, I
  don't care for probes that much … let my visual inspection determine."*). Consequence, stated
  plainly so it cannot be walked back later: the statistical channel can only say
  *distinguishable / not distinguishable from noise*; **practical significance is adjudicated
  by the human's reading of the visual battery** (§"The visual battery"). No `sesoi_check` call
  is made in this campaign, and no verdict below contains a ΔR² or Δ-contrast cutoff.
  — `stats.py: sesoi_check` (deliberately unused; the omission is the declaration)
- **Paired tests where arms share a frame**: the accessibility arm and its lattice twin are
  **the same 482,706 hexagons in the same row order** (region_id SHA verified identical), and
  for target cells the same joined subset. Every arm-vs-arm comparison is therefore a **paired
  permutation test on per-hexagon values** (sign-flip on the per-hexagon differences), never an
  unpaired comparison. — `stats.py: paired_permutation`
- **Per-example predictions persisted (MANDATORY)**: every cell writes its per-hexagon vector
  to `data/` with a `SidecarWriter` sidecar — residual norms, per-hexagon local contrast,
  per-hexagon probe predictions (`*_preds.npy`). Without them, `bootstrap_ci` and
  `paired_permutation` have no input in Phase 3. Exact paths are given per cell in §The Full Grid.
- **Claim scope**: this campaign's verdicts generalize over
  `{ seeds: {42, 43, 44} (n=3) ; objective families: 5 (plain, gVAE, PVAE, NAG-S0, NAG-S1) ;
  target families: 2 (leefbaarometer LBM3 Meting-2024/2022-score-year, RUDIFUN 2022) }`
  and are **N=1 — explicitly not generalized — on every one of**:
  `input fabric (306D concat only, N=1)` · `H3 resolution (res9, N=1)` ·
  `study area (Netherlands, N=1)` · `backbone (FullAreaUNet [9,8,7], dims [64,128,256],
  num_convs=5, N=1)` · `hyperparameter setting (the single frozen 2026-07-18 config, N=1)` ·
  `accessibility-graph construction (walk/bike/drive gravity-weighted, percentile-pruned,
  2026-07-10 regen, N=1)` · `target coverage (27.2 % / 49.3 % joined subsets, not the full
  grid)`. Any sentence in the final report that generalises past this list is a defect.

---

## Hypotheses

**Numbering ↔ mnemonic map (both names are used throughout; they denote the same rows).** The
numbered form is canonical — `scripts/science/order_gate.py` detects hypothesis rows by a
heading matching `\bH\d+\b`, so a mnemonic-only heading would leave that row out of the
integrity receipt.

| Canonical | Mnemonic | Short name | Kind |
|---|---|---|---|
| **H1** | H-RES-a | Residual magnitude | exploratory / descriptive |
| **H2** | H-RES-b | Residual target-legibility | CONFIRMATORY (Family A) |
| **H3** | H-RES-c | Residual spatial structure | CONFIRMATORY (Family B) |
| **H4** | H-SHARP | Sharpening vs smoothing | CONFIRMATORY (Family C) |
| **H5** | H-CKA | Representational similarity | exploratory / descriptive |

### The residual subspace: what accessibility adds beyond a linear re-description (three sub-hypotheses below)

**Framing.** If the accessibility embedding were merely a rotated/rescaled version of the
lattice embedding, a linear map would recover it exactly and there would be nothing to
explain. H-RES measures how much is left over, whether that leftover reads any target, and
whether it is a coherent spatial field or noise. It is three sub-hypotheses on one object.

#### H1 — Residual magnitude · mnemonic `H-RES-a` (EXPLORATORY / DESCRIPTIVE)

- **Background**: purely a measurement; no prior campaign has fit a lattice→acc map. Nothing
  retrodictive supports a specific magnitude.
- **Prediction (directional, pre-registered)**: `f_resid = 1 − R²(lattice → acc)` is **strictly
  between 0.001 and 0.5** for every functional objective row — i.e. the arms are neither
  numerically identical nor unrelated. No sharper prediction is offered, and none is
  retrofitted later.
- **Decision rule (frozen, descriptive labels — not a hypothesis test)**:
  | Observed outcome (per objective row, mean over 3 seeds) | Verdict label |
  |---|---|
  | `f_resid < 0.001` (map recovers acc essentially exactly) | DEGENERATE — arms are a linear re-description of each other; **STOP RULE SR-1 fires for this row** (see §Stop rules) |
  | `0.001 ≤ f_resid < 0.05` | SMALL residual subspace |
  | `0.05 ≤ f_resid < 0.30` | SUBSTANTIAL residual subspace |
  | `f_resid ≥ 0.30` | LARGE residual subspace — the graph swap restructured the representation |
  | primary × confirmatory (if any) | n/a — single estimator (ridge, out-of-fold) |
- **Severity**: if the graph swap did nothing beyond a linear re-expression, this cell would
  report `f_resid` at or below 0.001 and SR-1 would fire on every row. That outcome is fully
  available and would end the campaign's H-RES-b/c branches — which is what makes the cell a
  test rather than a formality.
- **Replay / reproduction gate**: before any map is fit, each of the 30 embeddings passes a
  load check — shape exactly `482,706 × 64`, `region_id` index present and unique, `region_id`
  column SHA-256 equal to `753a552b27ce…`, zero non-finite values. A failure fires **SR-2**.
- **RESULT**: All 15 row × seed cells ran; **no stop rule fired** (SR-1/SR-2/SR-3 all clear, so no row
  was DEGENERATE and every downstream cell stayed live). `f_resid = 1 − R²(lattice → acc)`, out-of-fold,
  variance-weighted over the 64 output dims, **mean ± sd over seeds {42, 43, 44}**:

  | Row | `f_resid` (mean ± sd) | forward R² | symmetry R²(acc → lattice) | frozen band |
  |---|---|---|---|---|
  | plain | **0.0414 ± 0.0088** | 0.9587 | 0.9564 ± 0.0039 | SMALL |
  | gVAE ⚠**COLLAPSE-CAVEAT** | **0.9713 ± 0.0528** | 0.0287 | 0.1614 ± 0.2798 | LARGE |
  | PVAE | **0.5389 ± 0.0078** | 0.4612 | 0.4919 ± 0.0098 | LARGE |
  | NAG-S0 | **0.1129 ± 0.0680** | 0.8871 | 0.8637 ± 0.1343 | SUBSTANTIAL |
  | NAG-S1 | **0.0483 ± 0.0048** | 0.9517 | 0.9592 ± 0.0051 | SMALL |

  Evidence: `data/study_areas/netherlands/acc_sharpening/2026-07-28/residual/{row}_s{seed}/map_r2.json`.
  **The pre-registered prediction is CONTRADICTED on one row and holds on the other three.** The
  prediction was `0.001 < f_resid < 0.5` for **every functional** objective row; PVAE's **0.5389 exceeds
  the 0.5 upper bound**. plain (0.0414), NAG-S0 (0.1129) and NAG-S1 (0.0483) fall inside the predicted
  interval. This is recorded as a contradicted directional prediction, not rounded down to "roughly as
  expected".
- **VERDICT**: per-row, quoting the frozen decision-rule branches verbatim:
  - plain — `0.001 ≤ f_resid < 0.05` → "**SMALL residual subspace**"
  - NAG-S1 — `0.001 ≤ f_resid < 0.05` → "**SMALL residual subspace**"
  - NAG-S0 — `0.05 ≤ f_resid < 0.30` → "**SUBSTANTIAL residual subspace**"
  - PVAE — `f_resid ≥ 0.30` → "**LARGE residual subspace — the graph swap restructured the
    representation**"
  - gVAE ⚠COLLAPSE-CAVEAT — `f_resid ≥ 0.30` → "**LARGE residual subspace — the graph swap restructured
    the representation**". The label is reported for completeness but measures collapse, not
    accessibility: C4 is a DEAD embedding, so "what the lattice arm cannot express" is trivially almost
    everything. The near-zero symmetry R² (0.1614 ± 0.2798, sd larger than the mean) is the collapse
    signature, not an accessibility finding.
  - No row met `f_resid < 0.001`, so **STOP RULE SR-1 did not fire anywhere** and no row is DEGENERATE.

#### H2 — Residual target-legibility · mnemonic `H-RES-b` (CONFIRMATORY)

- **Background**: **retrodictive prior-input, named as such** — the 2026-07-18 campaign found
  no target-legible difference between the arms on either target family (`results/master_results_table.csv`).
  That is *already-observed* evidence and is used here only to set the prior; it is **not** the
  evidence this cell is tested on. The evidence here is new: a probe of the residual matrix `R`,
  which was never computed.
- **Prediction (directional, pre-registered)**: the residual carries **no target-legible
  signal** — probe R² of `R` against each target family is statistically indistinguishable from
  zero. This is a null prediction, stated up front so that a non-null is a surprise the design
  cannot bury.
- **Decision rule (frozen)**:
  | Observed outcome (per objective row × target family) | Verdict |
  |---|---|
  | bootstrap CI on residual-probe R² **includes or lies below zero** | NULL CONFIRMED — accessibility's residual is target-illegible (the predicted outcome) |
  | bootstrap CI **excludes zero above** AND Holm-adjusted paired-permutation p < 0.05 within Family A | **SURPRISE — residual carries target-legible signal.** Reported LOUD, in the report's headline, with its map; not softened, not buried in an appendix |
  | CI excludes zero but Holm-adjusted p ≥ 0.05 | AMBIGUOUS — reported as ambiguous, never promoted to a positive finding |
  | primary × confirmatory (if any) | n/a — single estimator (DNN probe, `num_layers=0`) |
- **Severity**: if the residual *did* carry target signal, this cell would show a positive
  probe R² with a CI clear of zero on at least one of ten row×family contrasts — a probe of
  482,706-row (joined: 131,194 / 238,072) data with 64 predictors has ample power to detect
  even a small genuine R². A flat result is therefore informative, not merely underpowered.
- **Replay / reproduction gate**: the probe driver is run once on the **acc arm itself**
  (not the residual) for the plain row at seed 42 and must reproduce that cell's published
  2026-07-18 R² to 3 decimals (`results/master_results_table.csv`). If it does not, **STOP** —
  the frame or the probe protocol has drifted, and no residual number is reported until the
  discrepancy is diagnosed (`.claude/rules/reproduce-before-diagnose.md`).
- **RESULT**: 🔴 **THE NULL PREDICTION IS BROKEN.** The residual carries target-legible signal on
  **7 of 8** Family-A contrasts under the seed-averaged reading, and on **4 of 8** under the conservative
  worst-seed reading. This is the pre-registered SURPRISE outcome and it is stated here, at the top of
  the results, exactly as clause 2 and the frozen branch require.

  **Replay / reproduction gate — SATISFIED ON THE ORIGINAL DEVICE.** The gate ran first on CPU and
  FAILED (9/11 components deviating, |d| 0.0025–0.0087). Diagnosis
  (`…/replay_gate/diagnosis.json`) established CPU-side that the configs are identical except `device`,
  and that the input artifacts, join counts, prediction index, target values (bitwise) and the 5-fold
  spatial-block partition are all identical — the entire non-torch pipeline reproduces bit-exactly, and
  `torch.randperm(n, device=…)` draws from the device's own RNG stream. Re-running the **same frozen
  protocol on CUDA** (the device the published numbers were born on) reproduced all 11 components
  **bit-exactly** (max |d| ≈ 5.6e-17), with a self-check confirming the config differed only in `device`
  (`…/replay_gate/replay_gate_cuda.json`). The probe protocol and frame are intact; the CPU deviations
  are device numerics. No residual number below was produced before that gate passed.

  **Aggregation of seeds — the frozen text underdetermines it, so BOTH readings are reported.** The
  pre-registration fixes the family (row × target family, gVAE excluded → size 8) and says headline
  numbers are "mean ± sd over the three seeds", but never states how three seeds collapse into one
  contrast-level p-value. Both defensible readings were computed **together**, before either was
  inspected; neither was selected for its outcome:
  - **Reading 1 — seed-averaged per-hexagon** (the convention this campaign already used for Family C /
    SHARP-1): average the three seeds' per-hexagon R² contribution vectors, then one paired permutation.
  - **Reading 2 — worst per-seed p** (conservative): a contrast is only as strong as its weakest seed.

  The bootstrap CI is the seed-averaged CI in both readings (the headline number is the seed mean);
  only the Holm-adjusted p differs. **The frozen decision rule is CI-first** — a CI that includes zero
  is NULL CONFIRMED regardless of p — and is applied that way below.

  | Contrast | R² (mean ± sd) | \|Δ\|/ε | 95 % CI (seed-avg) | Holm adj-p R1 | Holm adj-p R2 | Verdict R1 | Verdict R2 |
  |---|---|---|---|---|---|---|---|
  | leefb × plain | +0.0403 ± 0.0108 | 3.75 | [+0.0332, +0.0476] | 8.0e-04 | 8.0e-04 | **SURPRISE** | **SURPRISE** |
  | leefb × PVAE | +0.0599 ± 0.0033 | 18.08 | [+0.0527, +0.0672] | 8.0e-04 | 8.0e-04 | **SURPRISE** | **SURPRISE** |
  | leefb × NAG-S0 | +0.0567 ± 0.0551 | 1.03 | [+0.0496, +0.0639] | 8.0e-04 | 7.9e-01 | **SURPRISE** | AMBIGUOUS |
  | leefb × NAG-S1 | +0.0469 ± 0.0020 | 23.34 | [+0.0398, +0.0540] | 8.0e-04 | 8.0e-04 | **SURPRISE** | **SURPRISE** |
  | RUDIFUN × plain | +0.0268 ± 0.0037 | 7.31 | [−0.0019, +0.0489] | 1.9e-02 | 1.6e-01 | NULL CONFIRMED | NULL CONFIRMED |
  | RUDIFUN × PVAE | +0.0570 ± 0.0037 | 15.39 | [+0.0284, +0.0790] | 8.0e-04 | 8.0e-04 | **SURPRISE** | **SURPRISE** |
  | RUDIFUN × NAG-S0 | +0.0465 ± 0.0364 | 1.28 | [+0.0179, +0.0687] | 8.0e-04 | 7.9e-01 | **SURPRISE** | AMBIGUOUS |
  | RUDIFUN × NAG-S1 | +0.0313 ± 0.0075 | 4.20 | [+0.0027, +0.0535] | 8.8e-03 | 1.6e-01 | **SURPRISE** | AMBIGUOUS |
  | *gVAE × leefb* ⚠**COLLAPSE-CAVEAT** | +0.2358 ± 0.0247 | 9.55 | per-seed CIs all > 0 | *excluded from Family A (clause 6)* | — | *not in family* | *not in family* |
  | *gVAE × RUDIFUN* ⚠**COLLAPSE-CAVEAT** | +0.2163 ± 0.0200 | 10.84 | per-seed CIs all > 0 | *excluded from Family A (clause 6)* | — | *not in family* | *not in family* |

  Join scope (mandatory caption): **leefbaarometer 131,194 / 482,706 = 27.2 %**; **RUDIFUN 238,072 /
  482,706 = 49.3 %**. Holm output: `…/residual_probe/summary/family_a_holm.json`. Per-cell evidence:
  `…/stage3_analysis/dnn_probe/2026-07-28/2026-07-28_accres_{row}_s{seed}_{leefb|rudifun}/`.
- **VERDICT**: **mixed, and reported as mixed** — each contrast quoting its own frozen branch verbatim:
  - **leefb × plain, leefb × PVAE, leefb × NAG-S1, RUDIFUN × PVAE** — under **both** readings: bootstrap
    CI "**excludes zero above** AND Holm-adjusted paired-permutation p < 0.05 within Family A" →
    "**SURPRISE — residual carries target-legible signal.** Reported LOUD, in the report's headline,
    with its map; not softened, not buried in an appendix". These four are the **robust core**: the
    finding does not depend on which aggregation a reader prefers.
  - **leefb × NAG-S0, RUDIFUN × NAG-S0, RUDIFUN × NAG-S1** — reading-dependent. Under Reading 1:
    "**SURPRISE — residual carries target-legible signal**". Under Reading 2: "CI excludes zero but
    Holm-adjusted p ≥ 0.05" → "**AMBIGUOUS — reported as ambiguous, never promoted to a positive
    finding**". Both are recorded; neither is promoted over the other.
  - **RUDIFUN × plain** — under both readings: bootstrap CI "**includes or lies below zero**" →
    "**NULL CONFIRMED — accessibility's residual is target-illegible (the predicted outcome)**". Note
    the Holm p rejects under Reading 1, but the frozen rule is CI-first and the CI straddles zero, so
    the frozen branch is NULL CONFIRMED. The rule is applied as written, not as it would flatter the
    headline.
  - **gVAE row** ⚠COLLAPSE-CAVEAT — excluded from Family A per clause 6, so no Holm verdict is issued.
    Its large R² (≈ +0.22 to +0.24) measures the collapsed-vs-functional contrast, i.e. collapse, not
    accessibility. Reported, never counted.

  **What this means, stated plainly**: accessibility message-passing injects into the representation a
  component that a linear re-description of the lattice arm cannot reproduce **and that reads the target**
  — signal the 2026-07-18 whole-embedding probes could not see, because there it was swamped by the ~95 %
  of the representation the two arms share. The effect is small in absolute terms (R² ≈ 0.03–0.06 on
  joined subsets) and is **not** a claim that the accessibility arm predicts better overall — the source
  campaign's null on whole-embedding probes stands as published.

#### H3 — Residual spatial structure · mnemonic `H-RES-c` (CONFIRMATORY)

- **Background**: no prior evidence. A residual that is a coherent spatial field (concentrated
  along corridors, around stations, in the Randstad) means something structural; a residual
  that is spatially white means the graph swap perturbed the representation without organising
  it. Both are publishable and neither is favoured.
- **Prediction (directional, pre-registered)**: the per-hexagon residual norm `‖R‖` is
  **spatially autocorrelated** — Moran's I > 0 with permutation p < 0.05. Rationale for the
  direction: the accessibility graph is itself a spatially structured object (corridors,
  network density), so any effect it has should inherit that structure. This is a genuine
  directional prediction and the flat/negative outcome is fully available.
- **Decision rule (frozen)**:
  | Observed outcome (per objective row, mean over 3 seeds) | Verdict |
  |---|---|
  | Moran's I ≥ 0.30 AND Holm-adjusted permutation p < 0.05 (Family B) | STRONGLY STRUCTURED — accessibility's contribution is a coherent spatial field |
  | 0 < Moran's I < 0.30 AND Holm-adjusted p < 0.05 | WEAKLY STRUCTURED |
  | Holm-adjusted p ≥ 0.05 OR Moran's I ≤ 0 | UNSTRUCTURED — residual is spatially noise-like (prediction contradicted; reported as such) |
  | primary × confirmatory (if any) | n/a — single estimator |
  The 0.30 boundary is a **descriptive band label**, not an effect-size gate: it separates
  "strongly" from "weakly" in the verdict wording and changes no reject/accept decision (that
  is made by the permutation p alone). Recorded as `[decided:analysis-default]` —
  conventional moderate-autocorrelation line — and surfaced as `[decided-by-human:2026-07-28]` **F6**
  for a nod or a waiver, since it is the one number in this document a reader could mistake
  for a threshold.
- **Severity**: if accessibility perturbed the representation without organising it, `‖R‖`
  would be spatially white and the 999-permutation null would not be exceeded — an outcome
  this test produces routinely on unstructured fields.
- **Replay / reproduction gate**: the ring-1 weight matrix is built with SRAI
  `H3Neighbourhood` on the canonical `regions_gdf` and its row count must equal 482,706 with
  every hexagon having ≥1 in-frame neighbour after the isolated-hex prune; the pruned count is
  reported, not silently absorbed.
- **RESULT**: The residual is a **spatially coherent field on every row**, not noise. Ring-1
  row-standardised weights were built via SRAI `H3Neighbourhood` on the canonical `regions_gdf`; the
  weight matrix covers all **482,706** hexagons with **0 isolated hexagons pruned** (the replay gate for
  this cell — every hexagon has ≥ 1 in-frame neighbour, so nothing was silently absorbed). Moran's I of
  `‖R‖`, 999-permutation null, permutation seed 42, one-sided greater (matching H3's directional
  prediction); row value = mean over seeds {42, 43, 44}.

  | Row | Moran's I (mean ± sd) | per-seed p | Holm adj-p (Family B) | reject |
  |---|---|---|---|---|
  | plain | **0.5177 ± 0.0081** | 0.001, 0.001, 0.001 | **0.0040** | yes |
  | PVAE | **0.2949 ± 0.0128** | 0.001, 0.001, 0.001 | **0.0040** | yes |
  | NAG-S0 | **0.3854 ± 0.0699** | 0.001, 0.001, 0.001 | **0.0040** | yes |
  | NAG-S1 | **0.5291 ± 0.0080** | 0.001, 0.001, 0.001 | **0.0040** | yes |
  | gVAE ⚠**COLLAPSE-CAVEAT** | **0.4740 ± 0.1968** | 0.001, 0.001, 0.001 | *excluded from Family B (clause 6)* | — |

  Every seed sits at the 999-permutation floor (1/1000 = 0.001), so the seed aggregation is not
  load-bearing: mean, median and max all return 0.001. Family B size 4 (gVAE excluded). Holm output:
  `…/residual/summary/family_b_holm.json`; per-cell evidence:
  `…/residual/{row}_s{seed}/moran.json`. The pre-registered directional prediction (Moran's I > 0 with
  permutation p < 0.05) is **corroborated on every row**.
- **VERDICT**: per-row, quoting the frozen branches verbatim:
  - **plain** (I = 0.5177), **NAG-S1** (I = 0.5291), **NAG-S0** (I = 0.3854) — `Moran's I ≥ 0.30 AND
    Holm-adjusted permutation p < 0.05 (Family B)` → "**STRONGLY STRUCTURED — accessibility's
    contribution is a coherent spatial field**"
  - **PVAE** (I = 0.2949) — `0 < Moran's I < 0.30 AND Holm-adjusted p < 0.05` → "**WEAKLY STRUCTURED**".
    Note the 0.30 boundary is a descriptive band label (F6), not an effect-size gate: PVAE sits 0.005
    below it and its reject/accept decision is made by the permutation p alone, which it passes.
  - **gVAE** ⚠COLLAPSE-CAVEAT (I = 0.4740 ± 0.1968) — excluded from Family B per clause 6, so no
    Holm verdict is issued. Descriptively it would fall in the STRONGLY STRUCTURED band, but with an sd
    over 40 % of the mean and one arm dead, it measures collapse geometry, not accessibility.
  - **No row was UNSTRUCTURED.** The "prediction contradicted" branch did not fire anywhere.

---

### H4 — Does accessibility sharpen or smooth local structure? · mnemonic `H-SHARP` (CONFIRMATORY)

- **Background**: the campaign title says "sharpening", but the honest position is that the
  direction is **genuinely open**, and pre-registering it as open is the point. Two mechanisms
  pull opposite ways: the accessibility graph carries **long-range** edges (a hexagon exchanges
  information with places that are *travel-close*, not *geometrically adjacent*), which can
  make geometric neighbours end up less alike → **sharpening**; but the same graph is
  **gravity-weighted and denser** in built-up areas, aggregating over more sources per step,
  which can wash out local differences → **smoothing**. No prior result on this codebase
  adjudicates between them.
- **Prediction (directional, pre-registered)**: **NONE — this is registered two-sided with an
  explicit declaration of no directional prior.** Registering "we expect sharpening" and then
  reporting smoothing as a surprise would be a retrofitted prior; registering the absence of a
  prior is the honest form. What *is* pre-registered is that the difference is non-zero in one
  direction or the other (i.e. we predict against the exact null).
- **Decision rule (frozen, two-sided; primary metric = mean ring-1 cosine distance)**:
  | Observed outcome (per objective row, Δ = acc − lattice, mean over 3 seeds) | Verdict |
  |---|---|
  | Δ > 0 AND Holm-adjusted paired-permutation p < 0.05 (Family C) | **SHARPENS** — accessibility increases local contrast between geometric neighbours |
  | Δ < 0 AND Holm-adjusted p < 0.05 | **SMOOTHS** — accessibility decreases local contrast |
  | Holm-adjusted p ≥ 0.05 | NULL — no detectable change in local contrast (the exact-null outcome; reported as a finding) |
  | primary × secondary metric (§H-SHARP-2) | **mixed-outcome policy, frozen now**: if the primary (cosine) and the secondary (Moran's I of PC1) give **opposite signed significant** verdicts on the same row, the row's verdict is **METRIC-DEPENDENT** and is reported as such — the primary does *not* silently win, and neither metric is promoted after the fact. If the secondary is null while the primary is significant, the verdict is the primary's with a "not corroborated by the secondary metric" annotation. |
- **Severity**: if the graph swap left local spatial structure untouched, the per-hexagon
  paired differences would centre on zero and the sign-flip permutation null would not be
  exceeded on any of the four family rows — with 482,706 paired hexagons, a genuine shift of
  even ~1 % of the metric's sd would be detected, so a null here is a strong null, not an
  underpowered one. (Corollary stated up front: at this n, **statistical significance is
  nearly guaranteed for any non-zero effect** — which is precisely why practical significance
  is delegated to the visual channel and the `|Δ| / ε` seed-noise margin is reported beside
  every Δ.)
- **Replay / reproduction gate**: the local-contrast metric is computed on both arms by the
  same function in the same call; a self-comparison (an arm against itself) must return
  Δ exactly 0.0 and permutation p exactly 1.0 before any cross-arm number is reported.
- **RESULT**: PENDING
- **VERDICT**: PENDING

---

### H5 — Representational similarity: which objective lets the graph restructure most? · mnemonic `H-CKA` (EXPLORATORY)

- **Background**: CKA is a descriptive similarity measure; there is no null hypothesis here and
  none is manufactured. The question is comparative and internal to the 306D family: across the
  five objectives, which one's representation is moved *most* by swapping the graph, and how far
  has each trained arm moved from its own 306D input — including whether a trained U-Net output
  ends up looking like nothing more than a **smoothed copy of its input** (the ring-agg
  comparison).
- **Prediction (pre-registered, weak and explicitly low-confidence)**: (i) CKA(acc, lattice)
  within a row will be **high (> 0.7)** for every functional row — consistent with the source
  campaign's null, since two representations that score identically on every probe are unlikely
  to be far apart; (ii) CKA(arm, raw-306D input) will be **lower** than CKA(arm, ring-agg-linear
  input), i.e. the trained outputs sit closer to a smoothed version of the input than to the raw
  input. Prediction (ii) is the interesting one and is stated *before* the numbers exist so that
  a contradiction is visible.
- **Decision rule (frozen — descriptive, no reject/accept)**:
  | Observed outcome | Verdict label |
  |---|---|
  | CKA(acc, lattice) ≥ 0.90 on a row | REPRESENTATIONALLY NEAR-IDENTICAL — the graph swap barely moved this objective's representation |
  | 0.70 ≤ CKA(acc, lattice) < 0.90 | MODERATELY RESTRUCTURED |
  | CKA(acc, lattice) < 0.70 | SUBSTANTIALLY RESTRUCTURED — this objective is where the graph choice matters representationally |
  | CKA(arm, ring-agg-lin) > CKA(arm, raw-306D) | the arm's output resembles a **smoothed** input more than the raw input (prediction (ii) corroborated) |
  | CKA(arm, ring-agg-lin) ≤ CKA(arm, raw-306D) | prediction (ii) contradicted — reported as contradicted |
  | primary × confirmatory (if any) | n/a — single measure |
- **Severity**: not applicable in the strict sense (no null test) — and saying so is the honest
  move rather than dressing a descriptive measure in inferential clothing. The falsifiable
  content is prediction (ii), which the ring-agg comparison can contradict outright.
- **Replay / reproduction gate**: CKA of any matrix with itself must return exactly 1.0 (to
  1e-10) and CKA of a matrix with an independent Gaussian matrix of the same shape must return
  < 0.05, both checked before any campaign number is computed.
- **Scope note (declared)**: all CKA comparisons are **within the 306D-input family**. The 178D
  fabric is out of scope for this campaign (§Governance clause 3).
- **RESULT**: PENDING
- **VERDICT**: PENDING

---

### The visual battery — the registered PRIMARY practical-significance channel

Per §Governance clause 8, the human's visual inspection is the **primary** channel and the
statistics are secondary. This is not a figure appendix; it is the deliverable against which
"does this matter?" is decided, and it is registered here with the same force as any hypothesis.

- **Form**: page-after-page maps, one page per objective row, large and clearly organised, each
  page carrying a **full-Netherlands panel and a South Holland zoom** (the province the human
  knows by eye), reusing the source campaign's bounding box and layout.
- **Machinery**: regenerated from the builder
  `scripts/one_off/2026-07-18-abl-map-battery.py`'s shared primitives
  (`stage3_analysis.render.{join_geometry, aggregate}`,
  `stage3_analysis.visualization.clustering_utils`, `utils.visualization.detect_embedding_columns`)
  via a new campaign builder. **Regenerated, never copied** — a copied render carries the source
  campaign's framing baked into pixels where no text grep can see it
  (`.claude/rules/viz-discipline.md` §Figure-creation, D62). Every PNG gets a
  `*.provenance.yaml` sidecar. Full per-panel raster resolution (`RASTER_W × RASTER_H`, never
  divided by panel count).
- **What the human is asked to judge** (stated now, so the question is not invented after seeing
  the maps): *does the accessibility residual land where accessibility should land* — along
  corridors, around stations, across the Randstad — *or is it diffuse?* And: *does the
  local-contrast difference map show a spatially interpretable pattern, or speckle?*
- **VERDICT (human)**: PENDING <!-- rule: filled only by the human's own reading, transcribed verbatim with date and channel. Never authored on their behalf. -->

---

## The Full Grid

Every cell below runs and is reported. **A cell producing a null, a tie, or a
prediction-contradicting value is reported exactly like any other.** All cells are
**analysis-only and CPU-side** — no training, no GPU (a peer terminal owns the GPU tonight).

Abbreviations: **rows** = the 5 objective pairs (plain, gVAE, PVAE, NAG-S0, NAG-S1);
**arms** = the 10 cell embeddings (acc + lattice per row); frames as declared above.

| Cell | target / columns | frame | arms | seeds | probe / statistic | confirmatory? | in correction family? | persisted per-example output | RESULT |
|---|---|---|---|---|---|---|---|---|---|
| **RES-a1** plain | none (embedding-only) | 482,706 | C1 ↔ C2 | 42,43,44 | out-of-fold RidgeCV map lattice→acc; `f_resid = 1 − R²` | no (descriptive) | no | `…/acc_sharpening/2026-07-28/residual/plain_s{seed}/residual.parquet` + `resid_norm.npy` | PENDING |
| **RES-a2** gVAE | none | 482,706 | C3 ↔ C4 ⚠COLLAPSE-CAVEAT | 42,43,44 | same | no | no | `…/residual/gvae_s{seed}/` | PENDING |
| **RES-a3** PVAE | none | 482,706 | C5 ↔ C6 | 42,43,44 | same | no | no | `…/residual/pvae_s{seed}/` | PENDING |
| **RES-a4** NAG-S0 | none | 482,706 | C7 ↔ C8 | 42,43,44 | same | no | no | `…/residual/nag_s0_s{seed}/` | PENDING |
| **RES-a5** NAG-S1 | none | 482,706 | C9 ↔ C10 | 42,43,44 | same | no | no | `…/residual/nag_s1_s{seed}/` | PENDING |
| **RES-b-leefb** | leefbaarometer (6 domains) — *join scope 131,194 / 482,706 = 27.2 %* | 131,194 | residual `R` of all 5 rows | 42,43,44 | DNN probe `num_layers=0`, 5-fold spatial-block; bootstrap CI on R² | **yes** | **Family A** (gVAE row excluded per clause 6) | `…/stage3_analysis/dnn_probe/2026-07-28/2026-07-28_accres_{row}_s{seed}_leefb/*_preds.npy` | PENDING |
| **RES-b-rudifun** | RUDIFUN 2022 (FSI/GSI/MXI/L/OSR) — *join scope 238,072 / 482,706 = 49.3 %* | 238,072 | residual `R` of all 5 rows | 42,43,44 | same | **yes** | **Family A** | `…/2026-07-28_accres_{row}_s{seed}_rudifun/*_preds.npy` | PENDING |
| **RES-c** | none | 482,706 | `‖R‖` of all 5 rows | 42,43,44 | Moran's I of `‖R‖`, ring-1 row-standardised, 999-perm null (seed 42) | **yes** | **Family B** (gVAE excluded) | `…/acc_sharpening/2026-07-28/residual/{row}_s{seed}/resid_norm.npy` (reused) | PENDING |
| **SHARP-1** *(primary metric)* | none | 482,706 | all 10 arms, paired within row | 42,43,44 | per-hexagon mean ring-1 cosine distance; paired permutation on Δ; bootstrap CI on Δ | **yes** | **Family C** (gVAE excluded) | `…/acc_sharpening/2026-07-28/local_contrast/cosine/{cell}_s{seed}.parquet` | PENDING |
| **SHARP-2** *(secondary metric)* | none | 482,706 | all 10 arms, paired within row | 42,43,44 | local Moran's I of PC1 per arm; paired permutation on Δ | no (robustness) | no | `…/local_contrast/moran_pc1/{cell}_s{seed}.parquet` | PENDING |
| **CKA-1** | none | 482,706 | acc ↔ lattice, per row | 42,43,44 | exact linear CKA; across-seed band (no bootstrap — see §(d) exception) | no (exploratory) | no | `…/acc_sharpening/2026-07-28/cka/cka_arm_vs_arm.csv` | PENDING |
| **CKA-2** | none | 482,706 | each of 10 arms ↔ raw-306D concat | 42,43,44 | exact linear CKA | no | no | `…/cka/cka_vs_raw306.csv` | PENDING |
| **CKA-3** | none | 482,706 | each of 10 arms ↔ ring-agg {linear (anchor), exponential, flat} 306D | 42,43,44 | exact linear CKA; prediction (ii) adjudicated on linear anchor; exp/flat supplementary descriptive (F5 generous extension) | no | no | `…/cka/cka_vs_ringagg_{linear,exponential,flat}.csv` | PENDING |
| **VIS-1** | none | 482,706 | `‖R‖` per row | 42 | residual-norm choropleth, full NL + South Holland zoom | n/a (visual channel) | no | PNG + `*.provenance.yaml` in `reports/2026-07-28-accessibility-sharpening-explainability/maps/2026-07-28/` | PENDING |
| **VIS-2** | none | 482,706 | `R` per row | 42 | residual PC-RGB map, full NL + SH zoom | n/a | no | same dir | PENDING |
| **VIS-3** | none | 482,706 | Δ local contrast per row | 42 | signed local-contrast difference map (acc − lattice), diverging colormap centred at 0, full NL + SH zoom | n/a | no | same dir | PENDING |

**Cell count**: 5 (RES-a) + 2 (RES-b) + 1 (RES-c) + 2 (SHARP) + 3 (CKA) + 3 (VIS) = **16 cells**,
each a reported row in the final report. Underlying fits: RES-a = 15 ridge maps (5 rows × 3
seeds); RES-b = 30 probe fits (5 rows × 3 seeds × 2 target families); RES-c = 15 Moran's I
tests; SHARP-1/2 = 30 arm-metric computations each; CKA-1/2/3 = 15 + 30 + 90 CKA evaluations
(CKA-3 = 30 per ring-agg variant × 3 variants, F5 generous extension).

**Producer scripts (to be committed BEFORE they run — order gate):**

- `scripts/one_off/2026-07-28-acc-residual-subspace.py` — RES-a, RES-c; writes `residual.parquet`,
  `resid_norm.npy`, `map_r2.json` per row × seed, each with a `SidecarWriter` sidecar.
- `scripts/one_off/2026-07-28-acc-residual-probe.py` — RES-b; drives `stage3_analysis/dnn_probe.py`
  over the residual matrices, persists `*_preds.npy` per cell.
- `scripts/one_off/2026-07-28-acc-local-contrast.py` — SHARP-1, SHARP-2.
- `scripts/one_off/2026-07-28-acc-cka.py` — CKA-1/2/3.
- `scripts/one_off/2026-07-28-acc-residual-map-battery.py` — VIS-1/2/3; imports the 2026-07-18
  battery's shared primitives, regenerates (never copies), emits `*.provenance.yaml` per PNG.
- Statistics throughout via `scripts/science/stats.py` (`bootstrap_ci`, `paired_permutation`,
  `seed_band`, `holm`), imported by the single sanctioned pattern
  (`sys.path.insert(0, "scripts/science"); import stats`).

---

## Stop rules (pre-registered; halt-branch-continue-queue semantics)

Per `specs/preregistration_science_lane.md` §(e), a fired stop rule halts **only the affected
row's downstream cells**; every independent frozen cell continues. Nothing is improvised in
place of a halted cell — the halt is recorded and the queue advances.

| ID | Trigger | Halts | Continues |
|---|---|---|---|
| **SR-1** | `R²(lattice → acc) > 0.999` on a row (arms are a linear re-description of each other — there is no residual to analyse) | that row's RES-b, RES-c, VIS-1, VIS-2 | that row's RES-a (reported as DEGENERATE), SHARP-1/2, CKA-1/2/3; all other rows entirely |
| **SR-2** | an embedding fails its load check (shape ≠ 482,706 × 64, `region_id` SHA ≠ `753a552b27ce…`, duplicate `region_id`, or any non-finite value) | every cell involving that arm | every cell not involving that arm |
| **SR-3** | residual degenerate by magnitude: `‖R‖ < 1e-9` for > 99 % of hexagons on a row | same as SR-1 for that row | same as SR-1 |
| **SR-4** | a target join returns fewer rows than the verified counts (131,194 leefb / 238,072 RUDIFUN) — silent-shrink guard | that target family's RES-b cells | the other target family; all embedding-only cells |
| **SR-5** | any `stats.py` helper raises (non-finite input, malformed p-values, n<2 seed band) | the cell that raised | everything else; the raise is reported verbatim, never caught-and-defaulted (`.claude/rules/no-fallbacks.md`) |

A fired stop rule is a **reported result**, not a failure to hide: each firing appears in the
final report with its trigger value.

---

## What Would Change Our Minds

The decision rules restated as falsifiable one-liners — the integrity signature of this
campaign. *Reproduced verbatim at the top of the final report; every verdict there quotes the
rule it answers to.*

- **The whole campaign collapses to "nothing to explain"** if **H1** (H-RES-a) reports
  `f_resid < 0.001` on every objective row — the accessibility embedding would then be a linear
  re-description of the lattice embedding and SR-1 would fire across the board.
- **H2's (H-RES-b) null prediction is broken** if the residual's probe R² has a bootstrap CI
  excluding zero above **and** survives Holm within Family A on any of the row × target-family
  contrasts. That outcome is reported LOUD and in the headline — it would mean accessibility
  carries target-legible signal that the 2026-07-18 whole-embedding probes could not see.
- **H3's (H-RES-c) structure prediction is contradicted** if Moran's I of `‖R‖` is ≤ 0, or fails the
  999-permutation null after Holm within Family B — i.e. accessibility perturbs the
  representation without organising it spatially.
- **H4 (H-SHARP) has no pre-registered direction to be wrong about, by design** — but its *exact
  null* is falsifiable: it is contradicted if the paired per-hexagon local-contrast difference
  survives Holm within Family C on any row, in either direction. "Accessibility sharpens" and
  "accessibility smooths" are both live pre-registered outcomes; whichever fires, the other was
  equally available.
- **H4's verdict is downgraded to METRIC-DEPENDENT** if the primary (ring-1 cosine
  distance) and secondary (Moran's I of PC1) metrics give opposite significant signs on the same
  row — the primary does not silently win.
- **H5's (H-CKA) prediction (ii) is contradicted** if `CKA(arm, ring-agg-linear) ≤ CKA(arm, raw-306D)`
  — the trained outputs would then sit closer to their raw input than to a smoothed version of
  it, contradicting the "a trained U-Net is mostly a learned smoother" reading.
- **Any statistical verdict is void as a practical claim** if the human's visual reading of the
  battery finds no interpretable spatial pattern — practical significance is the human's
  channel, there is no numeric SESOI, and a Holm-surviving p at n = 482,706 is not by itself a
  finding.
- **Any verdict is void as a general claim** the moment it is stated beyond the claim scope:
  306D input only, res9 only, Netherlands only, one backbone, one hyperparameter setting, one
  accessibility-graph construction, two target families on 27.2 % / 49.3 % joined subsets.

---

## Foundations — RATIFIED `[decided-by-human:2026-07-28]`

Analysis-only campaign — the `.claude/rules/novel-research-escalate-dont-default.md` **hard
training gate** does not fire (no model is trained; probing existing artifacts is analysis, per
`specs/preregistration_science_lane.md` §Implementation Notes). These six were nevertheless
design choices the human owned; all six were surfaced as `[blocked:human-decision]` and closed
by the human in-chat before the freeze commit (see the ratification record below the table).

| ID | Question | Drafted proposal (NOT ratified) | Alternative considered | Status |
|---|---|---|---|---|
| **F1** | How is the DEAD gVAE-lattice embedding (C4) handled? | **Analyse-and-report-with-caveat**: the gVAE row runs every cell, every number carries a COLLAPSE-CAVEAT tag, and the row is excluded from all three Holm families (a collapsed-vs-functional contrast measures collapse, not accessibility) | Exclude the gVAE row entirely — cleaner families, but discards a real observation about what a graph swap does when one arm is degenerate | `[decided-by-human:2026-07-28]` |
| **F2** | Which local-contrast metric is PRIMARY for H-SHARP? | **Mean cosine distance from a hexagon's 64D embedding to its H3 ring-1 neighbours** — direct, unit-free, no PCA dependency, defined identically on both arms | Local Moran's I of PC1 (drafted as the secondary/robustness metric) — interpretable and standard, but depends on a per-arm PCA whose axes differ between arms | `[decided-by-human:2026-07-28]` |
| **F3** | Which direction is the residual map fit? | **lattice → acc** (predict the accessibility arm from the lattice arm; the residual is then "what accessibility adds"), with **acc → lattice reported as a symmetry check** | acc → lattice as primary (residual = "what the lattice retains that accessibility drops") — a different question, equally valid, but not the campaign's question | `[decided-by-human:2026-07-28]` |
| **F4** | Does the 2026-07-18 **no-numeric-SESOI** ruling carry to this campaign? | **Yes — carry it.** No SESOI; practical significance is the human's visual reading of the map battery; the statistical channel only says distinguishable-from-noise | Declare a numeric SESOI per metric (would require the human to name a meaningful ΔR², Δ-cosine, and ΔMoran — three numbers they previously declined to name) | `[decided-by-human:2026-07-28]` |
| **F5** | Which ring-agg variant anchors CKA-3, and is its unsidecar'd provenance acceptable? | **linear weighting** (`netherlands_res9_2022_linear.parquet`), used as-is with the provenance caveat declared in §Governance clause 7 | Run all three variants (exp/lin/flat) → 30 extra CKA evaluations, cheap; and/or regenerate the ring-agg matrices with proper sidecars before use | `[decided-by-human:2026-07-28]` |
| **F6** | Is the Moran's I 0.30 "strongly/weakly structured" band label acceptable as a **descriptive label** (it gates no decision — the permutation p does)? | Accept as `[decided:analysis-default]` (conventional moderate-autocorrelation line) | Drop the band entirely and report Moran's I as a bare number with its p — removes the only number a reader might mistake for a threshold | `[decided-by-human:2026-07-28]` |

**Ratification record.** All six rows were ratified `[decided-by-human:2026-07-28]` in-chat
(terminal cool-lingering-dew, same evening, immediately before the freeze commit). The human's
words, verbatim: *"what was holm again? for the rest be generous we have all night to do this
well with a [parallel agent doing a rerun. ready for clear niche?"* — read as: all proposals
accepted; generosity applied to scope (every secondary, symmetry check, and ring-agg variant
stays IN — F5 extended to evaluate all three variants, linear remaining the registered anchor;
F1's Holm-exclusion of the dead-C4 row retained as a validity matter, with the gVAE row still
running and reported in every cell). The Holm question was answered in-chat before ratification.
Only the human may write the `[decided-by-human:<date>]` tag; these stamps transcribe their
in-chat decision, per `.claude/rules/novel-research-escalate-dont-default.md`.

---

## Cross-references

- Contract: `specs/preregistration_science_lane.md` (report-form §(a), order gate §(c),
  statistical-honesty layer §(d), autonomy/halt semantics §(e), report layer §(g)).
- Template: `specs/templates/PREREGISTRATION.template.md` (this file is a filled copy —
  verified byte-identical at copy time).
- Source campaign (background evidence, NOT re-tested here):
  `reports/2026-07-18-accessibility-ablation/` — `PREREGISTRATION.md`, `README.md`
  (incl. §"Human visual verdict (RECORDED 2026-07-28)"), `results/master_results_table.csv`,
  `results/confirmatory_stats.{json,md}`, `maps/2026-07-19/`.
- Input artifacts (identity established by `run_info.json` + on-disk verification; **no
  `*.run.yaml` sidecars** — §Governance clause 7):
  `data/study_areas/netherlands/stage2_multimodal/abl_c{01..10}_{objective}_{arm}_s{42,43,44}/embeddings/netherlands_res9_2022.parquet`
  (30 × 482,706 × 64);
  `…/stage2_multimodal/concat/embeddings/netherlands_res9_2022_306D.parquet` (sidecar green,
  `data_vintage: 2022`, hex2vec `filter_id: filter_v2`);
  `…/stage2_multimodal/abl_ring_agg_306D/embeddings/netherlands_res9_2022_{exponential,linear,flat}.parquet`.
- Targets: `…/target/leefbaarometer/leefbaarometer_h3res9_2022.parquet` (LBM3 Meting-2024
  release, 2022 score year; 131,194 hexes; 27.2 % join);
  `…/target/rudifun/rudifun_h3res9_2022.parquet` (`data_vintage: 2022`; 238,072 hexes; 49.3 % join).
- Order-gate reporter: `scripts/science/order_gate.py` — run against this file after the freeze
  commit (pre-run: expect every row `NO-EVIDENCE-FOUND`, which is the correct Phase-1 signal)
  and again after results land (expect all-PASS).
- Stats helpers: `scripts/science/stats.py` (`bootstrap_ci`, `paired_permutation`, `seed_band`,
  `holm`; `sesoi_check` deliberately unused per §(d)).
- Visual machinery to regenerate from (never copy): `scripts/one_off/2026-07-18-abl-map-battery.py`,
  `stage3_analysis/render.py`, `stage3_analysis/visualization/clustering_utils.py`,
  `utils/visualization.detect_embedding_columns`.
- Rules: `.claude/rules/artifact-provenance.md` (commit-then-run; sidecar-read; the
  `unet_` prefix is still unregistered in `MODALITY_REGISTRY` — carried `[open]`),
  `.claude/rules/domain-guardrails.md` (grid/frame asserts; EPSG:28992 rider),
  `.claude/rules/novel-research-escalate-dont-default.md` (foundations are the human's to close),
  `.claude/rules/viz-discipline.md` (figure-creation: regenerate from builder, provenance
  sidecars, full raster resolution),
  `.claude/rules/training-discipline.md` (CPU-side; the GPU is a peer terminal's tonight),
  `.claude/rules/no-fallbacks.md` (SR-5: a raising helper is reported, never defaulted),
  `.claude/rules/data-lifecycle.md` (all outputs additive under date-keyed dirs; nothing deleted).
- Ops plan / lane: `.claude/plans/2026-07-28-interrogate-what-accessibility-contributes-as-model-structure-train.ops.md`
  (Wave-2 = this registration; Wave-3 = execute; Wave-4 = report).
- Artifact inventory basis (read-only, verified on disk 2026-07-28):
  `.claude/scratchpad/stage3-analyst/2026-07-28-cool-lingering-dew.md` §20:10.
