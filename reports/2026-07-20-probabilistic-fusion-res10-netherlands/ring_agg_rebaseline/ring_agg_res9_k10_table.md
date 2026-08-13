# Ring-agg res9 K=10 canonical re-baseline — probe R^2 table

Metric: **out-of-fold R^2** (`overall_oof_r2`), random 5-fold (seed 42), linear DNN probe (`--num-layers 0`), CPU. Input: canonical 178D res9 concat (`netherlands_res9_2022`), ring-aggregated K=10 per weighting. Produced by `scripts/stage3/run_arm_probe_sweep.py` (campaign harness, F2a random k-fold) — directly comparable to the accessibility-ablation baseline table.

Vintage: embeddings=2022; leefbaarometer=2024 (NOT aligned); rudifun=2022 (aligned). leefbaarometer join = 131,194 rows (24.4% of arm, 100% of target); rudifun join = 238,072-row target.

### All targets (out-of-fold R^2)

| weighting | lbm | fys | onv | soc | vrz | won | fsi | gsi | mxi | l | osr | mean(leefb) | mean(rudi) | **mean(all)** |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| exponential | 0.2707 | 0.4074 | 0.4905 | 0.6369 | 0.7410 | 0.4475 | 0.4960 | 0.4532 | 0.5860 | 0.5569 | 0.1253 | 0.4990 | 0.4435 | **0.4738** |
| logarithmic | 0.2669 | 0.4133 | 0.5220 | 0.6281 | 0.8588 | 0.4387 | 0.5037 | 0.4438 | 0.5521 | 0.5689 | 0.1381 | 0.5213 | 0.4413 | **0.4849** |
| linear | 0.2658 | 0.4193 | 0.5201 | 0.6261 | 0.8533 | 0.4418 | 0.5046 | 0.4431 | 0.5468 | 0.5675 | 0.1402 | 0.5210 | 0.4404 | **0.4844** |
| flat | 0.2408 | 0.3721 | 0.5093 | 0.5738 | 0.8730 | 0.4044 | 0.4759 | 0.4066 | 0.4592 | 0.5332 | 0.1359 | 0.4956 | 0.4022 | **0.4531** |

### Mean OOF R^2 ranking (across all 11 targets)

| rank | weighting | mean(all) | mean(leefb) | mean(rudi) |
|---|---|---|---|---|
| 1 | logarithmic | 0.4849 | 0.5213 | 0.4413 |
| 2 | linear | 0.4844 | 0.5210 | 0.4404 |
| 3 | exponential | 0.4738 | 0.4990 | 0.4435 |
| 4 | flat | 0.4531 | 0.4956 | 0.4022 |

### Comparison vs the accessibility-ablation report (READ-ONLY, frozen `results/`)

Source: `reports/2026-07-18-accessibility-ablation/results/master_results_table.csv` + `confirmatory_stats.md`
(res9 Netherlands; linear probe `dnn_probe --num-layers 0`; family-mean R^2).

**Input-set caveat (load-bearing):** the ablation baselines/learned cells run on a **306D** concat
(AlphaEarth 64 + **Tessera 128** + hex2vec 50 + roads 64) over the Tessera-restricted **482,706-hex** grid;
this re-baseline runs on the canonical production **178D** concat (**no Tessera**) over the full **537,970-hex**
grid. Absolute numbers are therefore near-comparable but not identical-input; the learned cells are 64D
bottlenecks vs the baselines' raw feature count (a Foundation-4 floor comparison, not a matched-capacity test —
the ablation's own framing).

| row | source | input | mean(leefb) R^2 | mean(rudi) R^2 |
|---|---|---|---|---|
| ring-agg k10 **log** (this) | canonical 178D | 178D | **0.5213** | 0.4413 |
| ring-agg k10 **lin** (this) | canonical 178D | 178D | 0.5210 | 0.4404 |
| ring-agg k10 **exp** (this) | canonical 178D | 178D | 0.4990 | 0.4435 |
| ring-agg k10 **flat** (this) | canonical 178D | 178D | 0.4956 | 0.4022 |
| B-ring-lin | ablation | 306D | 0.525 | 0.447 |
| B-ring-exp | ablation | 306D | 0.508 | **0.456** |
| B-ring-flat | ablation | 306D | 0.494 | 0.403 |
| B-raw (concat, no smooth) | ablation | 306D | 0.480 | 0.437 |
| **C10 NAG-S1 lattice (best learned)** | ablation | 306D→64D | 0.391 | 0.362 |
| C09 NAG-S1 acc (2nd learned) | ablation | 306D→64D | 0.385 | 0.354 |
| C04 gVAE lattice (worst learned) | ablation | 306D→64D | -0.006 | -0.002 |

### Verdict — CLAUDE.md "canonical-grid re-test pending" caveat

**RESOLVED (affirmative): ring-agg k=10 STILL beats every learned UNet variant on the canonical grid.**
On the canonical 178D res9 production concat, every ring-agg weighting (mean-all R^2 0.453–0.485; best
leefbaarometer 0.521 log/lin) exceeds the ablation's best learned FullAreaUNet cell (C10 NAG-S1 lattice:
leefb 0.391, rudi 0.362) by ~0.10 mean R^2 (~0.13 on leefbaarometer) — and does so with the LEANER 178D
input (no Tessera) on the FULL grid, so the smoothing-beats-learning result is not a Tessera artifact. The
zero-parameter k-ring floor reproduces on the canonical production input; the March-2026 old-grid finding
holds on canonical. Best weighting: **logarithmic** (mean-all 0.4849), effectively tied with linear (0.4844);
flat is worst (0.4531). Caveat: the qualitative gap (~0.10) is far larger than any random-vs-spatial-CV
delta (~0.01–0.03), so the verdict is robust to the fold-scheme question.

---
*Provenance: cv_scheme=random_kfold (F2a, campaign harness `run_arm_probe_sweep.py`), n_folds=5, seed=42,
probe=DNN `--num-layers 0` (linear), device=cpu, resolution=9. 4 arms × 11 targets = 44 cells, all rc=0.
Ring-agg arms = canonical 178D res9 concat smoothed K=10 (exp = reused Jul-14 canonical `netherlands_res9_2022.parquet`;
log/lin/flat generated 2026-07-20, sidecars green: 537,970×178, nan=0 inf=0 zero_row_frac=0). Raw per-cell
JSON: `data/study_areas/netherlands/stage3_analysis/2026-07-20-ring-agg-rebaseline-probes/2026-07-20/`.
Aggregated raw: `ring_agg_res9_k10_results.json` + `ring_agg_res9_k10_table.parquet` (this dir).*
