# Probabilistic fusion at res10 — the overnight report

**Session**: stone-gathering-storm (overnight autonomous, `/niche` W4). 2026-07-20.
**Audience**: you, with morning tea — plus future sessions and the eventual writeup.
**What this is**: the night's central deliverable. It answers one scaling question, re-baselines the
best full-area model on the canonical grid, records report-polish and figure work, and closes two
parked methods items. Every number below traces to an on-disk artifact — see [Artifact index](#8-artifact-index).

Terms are defined at first use. "res9 / res10" = H3 hexagon resolutions (~500 m / ~15 m edge). "178D" =
the fused per-hex feature vector (AlphaEarth 64 + POI/hex2vec 50 + roads 64). "OOF R²" = out-of-fold R²,
the probe's held-out predictive accuracy. "probe" = a small regression trained on frozen embeddings to
predict an urban outcome (liveability, density) — it measures how much signal the embedding carries.

---

## 1. THE VERDICT

**Question**: *Can we crack full-Netherlands res10 (3,752,940 hexes × 178D) with cone-sampled
probabilistic fusion (geo-MPC)?*

| | Answer | Why |
|---|---|---|
| **Geo-MPC as it stands tonight** | **NO** | The full-resident "windmill" trainer is hardcoded to South Holland / res9 and holds the whole study area in memory. The binding wall is **VRAM**: the per-anchor windmill geometry precompute scales ~70× (34,302 → ~2.4M anchors) into **tens of GB > the 3090's 24 GB**. RAM is fine (2.67 GB resident / ~10–15 GB load-peak « 64 GB). |
| **A cone-lazy trainer (on paper)** | **YES — but unbuilt** | The 284-cone res5→res10 cache already exists (12 GB on disk); `LazyConeBatcher` holds ~0.4–0.7 GB per batch-of-32, well inside both limits. **But no geo-MPC trainer consumes cones today**, and full-resident-vs-cone-lazy is a foundational `[blocked:human-decision]`. |
| **Was a res10 probe run tonight?** | **NO** | Item 6c (single-seed res10 feasibility run) was **not** attempted — the geo-MPC gate failed (see caveat) and the GPU was owned all night by the peer's sensitivity chain. |

**Science caveat, stated honestly**: the best geo-MPC arm (a7) is **statistically indistinguishable from
its own untrained twin** — Δ = **+0.0040** mean OOF R², p ≈ 0.57. A res10 run would therefore test the
*scaling* of an architecture class whose *training* does not yet add measurable value even at res9. That
is a legitimate engineering-feasibility question, but it is not a science-value one, and it is blocked on
four concrete items. Full detail: [`scaling_analysis.md`](scaling_analysis.md).

> **One number worth not confusing**: the res10 *tessellation* (`regions_gdf`) is 3,897,535 hexes; the
> res10 *concat* is 3,752,940 hexes — the ~4% smaller count is the hierarchical-consistency filter
> (hexes whose res5 parent falls outside the study area are dropped). Both counts are correct; they name
> different artifacts. This report uses **3,752,940** (the concat) for the res10 question.

**Bottom line**: res10 is *reachable* — the cone cache exists, RAM is ample, the model class runs at res9
— but only through a cone-lazy trainer that does not yet exist, and only after a human decision about what
a "glimpse" is at res10 scale. Full-resident res10 on a 24 GB card is a hard NO.

---

## 2. Ring-agg canonical re-baseline — the CLAUDE.md caveat is RESOLVED

**Ring-agg** = zero-parameter spatial smoothing: each hex's embedding is replaced by a distance-weighted
average of its k-ring neighbourhood (here k=10). It is the current best full-area performer. CLAUDE.md
carried a caveat that its "beats learned UNets" result was old-grid and awaited a **canonical-grid
re-test**. Tonight's lane ran that re-test.

**Headline**: **logarithmic weighting wins** (mean OOF R² **0.4849**; leefbaarometer family **0.5213**),
and ring-agg k=10 beats the best *learned* UNet cell by **~0.10–0.13 R²**. The caveat is **RESOLVED,
affirmative**, and — crucially — it is **not a Tessera artifact**: the win holds on the leaner canonical
178D concat (no Tessera) over the full 537,970-hex grid, against an ablation that used a richer 306D
Tessera concat on a restricted 482,706-hex grid.

*Metric: out-of-fold R², random 5-fold, seed 42, linear DNN probe (`--num-layers 0`). lbm/fys/onv/soc/vrz =
leefbaarometer sub-scores; fsi/gsi/mxi/l/osr/won = RUDIFUN land-use.*

| weighting | mean(leefb) | mean(rudi) | **mean(all)** | rank |
|---|---|---|---|---|
| **logarithmic** | **0.5213** | 0.4413 | **0.4849** | **1** |
| linear | 0.5210 | 0.4404 | 0.4844 | 2 |
| exponential | 0.4990 | 0.4435 | 0.4738 | 3 |
| flat | 0.4956 | 0.4022 | 0.4531 | 4 |

**Ring-agg vs the learned UNets** (from the frozen accessibility-ablation report; family-mean R²):

| embedding | input | mean(leefb) | mean(rudi) |
|---|---|---|---|
| ring-agg k10 **log** (tonight) | canonical 178D | **0.5213** | 0.4413 |
| ring-agg k10 lin (tonight) | canonical 178D | 0.5210 | 0.4404 |
| B-ring-lin (ablation baseline) | 306D | 0.525 | 0.447 |
| **C10 NAG-S1 lattice** (best *learned* UNet) | 306D→64D | **0.391** | **0.362** |
| C04 gVAE lattice (worst learned) | 306D→64D | -0.006 | -0.002 |

The zero-parameter smoother (0.521 leefb) beats the best learned fusion (0.391 leefb) by ~0.13 — with a
leaner input, on the full grid. The qualitative gap (~0.10) dwarfs any plausible random-vs-spatial-CV
delta (~0.01–0.03), so the verdict is robust to the fold-scheme question in §5. Full table + provenance:
[`ring_agg_rebaseline/ring_agg_res9_k10_table.md`](ring_agg_rebaseline/ring_agg_res9_k10_table.md).

---

## 3. Report polish — accessibility-ablation report (§3 method + 3 defects)

The frozen accessibility-ablation report got a writeup-grade method restructure and a defect pass.

**Method (§3) restructure**: overview-first "the experiment in one picture" opener (walks the pipeline
Fig 1 end-to-end); the five-objectives prose became a table (`tab:objectives`); every evaluation term is
now defined at first use (arm / backbone / objective / linear-probe=OLS / spatial-block-CV / family-R² /
reference-floor).

**Defects** (before/after PNGs in
`../2026-07-18-accessibility-ablation/qaqc_render_2026-07-20/` — private repo, not part of this public subset):

| ID | Defect | Status |
|---|---|---|
| **D1** | Fig-14 header painted over by hex swatches | **FIXED** — zorder fix; regenerated under new stem `fig_fund_collapse_2026-07-20` |
| **D2** | Float-page vertical centering glue | **FIXED** — `\@fptop=0pt` |
| **D4** | hyperref unicode warning | **FIXED** — `\texorpdfstring` |
| D3 | Diff-H* page sizing | **wontfix** tonight (figure-size fragility gate — see §7a) |
| D5 | (no visual defect) | **wontfix** |

**Build**: 53 pp, xelatex ×2 exit 0, 0 errors, hyperref warnings 1→0. Rebuilt PDF:
`../2026-07-18-accessibility-ablation/latex/report.pdf` (private repo, not part of this public subset).

---

## 4. G3 gVAE de-collapse maps — 18 panels, exploratory finding

The gVAE (variational-autoencoder) arm had collapsed in the frozen ablation cell (all embedding
dimensions near-zero). The de-collapsed reruns were rendered as an 18-panel map battery, wired into the
report as `fig:decollapse-maps` at the end of the de-collapse section.

- **All 6 runs verified healthy de-collapsed**: 482,706 × 64, per-dimension std ~0.12.
- **Exploratory finding**: the **lattice** arm (the one that collapsed in the frozen cell) now gives
  visibly **cleaner, more contiguous partitions** than the accessibility arm — consistent with its higher
  probe R² (leefb **0.304 lattice vs 0.264 accessibility**).
- **Caveat**: cluster colours are **not** cross-panel comparable (each panel is its own PCA8 + KMeans12,
  recorded in per-run provenance sidecars).

Panels: `../2026-07-18-accessibility-ablation/figures/maps_g3_2026-07-20/` (private repo, not part of this public subset).

---

## 5. Spatial-CV verdict (parked item P1, closed report-only)

**Question**: do the probe results leak spatial signal because of random (not geographic) cross-validation
folds? **Verdict: the P1 park stands.**

- Spatial block-CV (10 km blocks, 5 folds, EPSG:28992) is **already the default** in the standalone
  probes (`dnn_probe.py` / `linear_probe.py`).
- The recent campaign probes **deliberately** used random k-fold (`run_arm_probe_sweep.py`,
  `cv_scheme=random_kfold`) for comparability across the comparator matrix.
- 10 km is adequate: ~20–30× the ~500 m res9 hex-leakage scale, ~2–5× the leefbaarometer
  autocorrelation range.
- Tonight's ring-agg row used random folds for table comparability (§2), consistent with the campaign
  harness. The ~0.10 ring-agg-vs-learned gap is far larger than any random-vs-spatial delta, so §2's
  verdict is unaffected.

This is closed **report-only** — no rule or park status changed. See
`.claude/rules/domain-guardrails.md` (private repo) P1.

---

## 6. res10 concat status — verified green, NOT rebuilt

The res10 concat was verified, not rebuilt (it has been canonical since 2026-07-18).

| Check | Value |
|---|---|
| Shape | 3,752,940 × 178 |
| Variance share (per block) | AlphaEarth .464 / hex2vec .362 / roads .174 — healthy |
| POI filter | filter_v2 |
| Vintage | 2022 per block |
| NaN / inf | 0 / 0 |
| **Caveat** | sidecar records `git_dirty: true` at production commit `d666680` |

---

## 7. DECISIONS AWAITING YOU

### From the pre-registration draft — 5 foundational `[blocked:human-decision]` items

None is closed; each is options-with-evidence. Full text:
[`PREREGISTRATION_DRAFT.md`](PREREGISTRATION_DRAFT.md).

| # | Choice | Options (compressed) |
|---|---|---|
| **F1** | **Trainer architecture for res10** | (a) full-resident *(OOMs)*; (b) cone-lazy `LazyConeBatcher`; (c) res9-only, defer res10 |
| **F2** | **What "the cone / glimpse IS" at res10 scale** | (a) tile the res9 windmill per-anchor; (b) redefine the glimpse as a res5-rooted cone subtree; (c) hybrid. *This is the paper-mapping question the novel-research gate exists for — do not default.* |
| **F3** | **Anchor-set definition for NL res10 (`is_anchor`)** | (a) all cells anchors; (b) density/coverage-filtered subset; (c) cone-root-derived. *NL res10 grid has no `is_anchor` column yet.* |
| **F4** | **Readout convention** | (a) `average` (fixation-averaged, today's default); (b) `anchor` (single central fixation). *Worth ~0.04–0.06 R² on identical weights — changes every historical comparison.* |
| **F5** | **Does geo-MPC enter as res9-only or res10?** | (a) res9-only this campaign (runnable soon after minor fixes); (b) block on the res10 cone-lazy build |

### Three coordinator-surfaced calls

- **(a) D3 Diff-H\* page sizing** — deferred tonight under the figure-size fragility gate. **Fix, or accept
  as wontfix?**
- **(b) Ring-agg fold-scheme convention going forward** — **random k-fold** (comparability with the
  existing comparator matrix) vs **spatial 10 km folds** (leakage-robustness). Tonight used random; pick a
  standing convention.
- **(c) Register `PREREGISTRATION_DRAFT.md` as a `/science` campaign?** — it is a draft with no order-gate
  receipts and every foundation open. Registering it means settling F1–F5 first.

---

## 8. Artifact index

**This report dir**
- [`scaling_analysis.md`](scaling_analysis.md) — the res10 scaling verdict + memory model (file:line cited)
- [`PREREGISTRATION_DRAFT.md`](PREREGISTRATION_DRAFT.md) — the 5-foundation draft (unregistered)
- [`ring_agg_rebaseline/`](ring_agg_rebaseline/) — [table.md](ring_agg_rebaseline/ring_agg_res9_k10_table.md) · [results.json](ring_agg_rebaseline/ring_agg_res9_k10_results.json) · [table.parquet](ring_agg_rebaseline/ring_agg_res9_k10_table.parquet) · [sweep_run.log](ring_agg_rebaseline/sweep_run.log) · [arms_manifest.txt](ring_agg_rebaseline/arms_manifest.txt)

**Accessibility-ablation report (frozen dir; docs untouched except the rebuild)** — private repo, not part of this public subset:
- Rebuilt PDF: `../2026-07-18-accessibility-ablation/latex/report.pdf` (53 pp)
- G3 maps (18 panels): `../2026-07-18-accessibility-ablation/figures/maps_g3_2026-07-20/`
- QAQC before/after renders: `../2026-07-18-accessibility-ablation/qaqc_render_2026-07-20/`

**New data artifacts** (outside `reports/`)
- 3 ring-agg parquets: `data/study_areas/netherlands/stage2_multimodal/ring_agg/embeddings/netherlands_res9_2022_k10_{logarithmic,linear,flat}.parquet`
- 44 per-cell probe JSONs (+ 44 sidecars): `data/study_areas/netherlands/stage3_analysis/2026-07-20-ring-agg-rebaseline-probes/2026-07-20/`

**Commits tonight**: `0842b1b` (G3 renderer) · `e5748b7`/`225eced`/`110af3f` (D1) · `59defbe` (D2) ·
`f3d9172` (ring-agg aggregator).

---

## 9. Fill-the-night honesty note

Figure iteration tonight happened **as part of** the G3 map battery (§4) and the D1 regeneration (§3).
**No additional standalone figure-iteration rounds were run** — this note exists so "figures were
iterated" is not read as more work than occurred.
