# Pre-Registration — Sharpening Operators on Urban Embeddings at Res10 (Supervised-Probe)

**Campaign slug**: `2026-07-19-sharpen-vs-smooth`
**Status**: **READY TO FREEZE — all values concrete.** The two pilot-coupled values (λ grid, stability bound) were filled 2026-07-19 from the declared-exploratory bracketing pilot (`data/study_areas/netherlands/stage2_multimodal/sharpen/pilot/2026-07-19/`); β_min and the seed set {42,43,44} frozen in the same pass. Next: freeze commit + sha recorded per the two-step freeze (`specs/preregistration_science_lane.md` §Implementation Notes). No governed producer or probe runs until the freeze commit exists.
**Date drafted**: 2026-07-19
**Author / shard**: `deep-coasting-sky` (focused)
**Git commit that froze this file**: `da40124` (backfilled in the first results commit per the two-step freeze — never amended)

**Foundations basis (read first)**: every design choice below implements a STAMPED foundation from `specs/sharpening_operators_dossier.md` §8 (the stamping checklist, landed in-chat 2026-07-19 — three choices decided explicitly, the remainder delegated to the coordinator under the human's recorded *"i trust your judgmenet elsewhere; continue all night if need be."*). Each governed choice cites its §8 stamp. This pre-registration is the **report-form** of the science lane (`specs/preregistration_science_lane.md` §(a) — a planned grid, every cell enumerable before any run), NOT the ledger-form.

---

## Purpose (overview first — self-evident to a stranger)

The frozen predecessor campaign `2026-07-18-smooth-vs-learn-res10` (freeze `6ff336a`, **READ-ONLY substrate**) characterized — probe-free, learning-free — what *smoothing* operators do to the raw fused city embedding at H3 resolution 10. Its headline **D-H1 REJECT**: the exponential k-ring smoother **saturates by k=5** (rings beyond r=5 carry negligible weight, so `R10k5e` and `R10k10e` are near-identical arms). Smoothing is a **low-pass filter**: it destroys high-spatial-frequency information, and a heavily-saturating smoother destroys it hardest.

This campaign adds the axis the predecessor deliberately excluded: the **supervised probe**, and it turns the smoother around. A **sharpening operator** *re-amplifies* spatial high-frequency content — the part of each hexagon's vector that differs from its neighbours' (center-surround / unsharp masking / difference-of-Gaussians; the discrete analogue of retinal lateral inhibition). The user's framing, load-bearing for the whole design: many probe **targets are spatially sharp** (concentrated in a few places — a specific neighbourhood's liveability signal), so a sharpening operator on the embeddings may **recover probe performance that smoothing costs**, precisely on those sharp targets, while doing nothing (or hurting) on spatially-smooth targets where smoothing already won.

The campaign therefore does not ask "does sharpening raise mean R²." It asks an **interaction** question: does the sharpening benefit **grow with the target's spatial sharpness**? Targets are binned sharp-vs-smooth by a **frozen sharpness score computed before any probe runs**, and the primary confirmatory hypothesis predicts a positive dose×sharpness interaction. It simultaneously probes the **existing smoothed arms** (the predecessor's C10 base + 5 ring-agg + 3 windmill arms — probe-free until now) to complete the other half of the story: smoothing should be **non-positive** on sharp targets.

Pre-registration closes the **measure-shopping / target-shopping** surface: with ~14 arms × ~10 target columns × 2 probe depths, a favourable-subset story would be trivial to assemble post hoc. Every cell is enumerated below; every cell is reported — nulls, ties, and prediction-contradicting values included. The dossier's own inverse-D-H1 warning is committed to as a reportable outcome: **sharpening can *explode*** (noise blow-up) rather than help — if the manipulation check shows the dose does not move sharpness measures monotonically, or explodes, that is the reported headline.

### Terms defined at first use

- **Concat / C10 (178D)**: the canonical late-fusion embedding at res10 — AlphaEarth 64D + POI-hex2vec(filter_v2) 50D + roads-highway2vec 64D, per-block z-scored (CLAUDE.md §Stage 2). Artifact: `data/study_areas/netherlands/stage2_multimodal/concat/embeddings/netherlands_res10_2022.parquet` (3,752,940 × 178). **The confirmatory operand `x`** (dossier §8 (a) `[decided-by-human:2026-07-19:delegated]`: A1 raw z-scored C10) and the **λ=0 identity anchor** of the sharpening dose axis.
- **Smoother `S`**: the frozen zero-parameter ring-aggregation operator (`stage2_fusion/models/ring_aggregation.py`) — each hexagon replaced by a weighted mean of neighbours out to K rings, weights following a scheme (`exponential`/`linear`/`flat`). Linear, row-stochastic: `S = Σ_k w_k A_k`. The predecessor materialized these as on-disk arms.
- **Unsharp sharpener `H` (CONFIRMATORY, dossier §3.1 / §8 (b) B1)**: `H(x) = x + λ·(x − S(x)) = (1+λ)·x − λ·S(x)`. A center-surround linear kernel: positive center `(1+λ)`, negative surround `λ·w_k` on each ring. **One knob λ ≥ 0**: λ=0 is identity (=C10), larger λ sharpens harder. **Primary surround `S` = the frozen `k10_exponential` ring-agg arm** (`R10k10e`); `k1`/`k5` surrounds are exploratory "dose-of-smoother" variants. Producer `scripts/stage2/run_sharpening_operators.py --family unsharp` (commit `de61161`); the A1 path REUSES the frozen ring-agg parquet as `S(x)` (free, exact provenance).
- **DoG sharpener (EXPLORATORY, dossier §3.2 / §8 (b) B2)**: difference-of-ring-aggregations, the thesis "annulus-with-gap" geometry (Figs 46–47) — positive center ring, NEUTRAL gap ring, NEGATIVE outer annulus. Default geometry **c0-g1-a2-4** (center ring 0, gap ring 1, annulus rings 2–4). Producer `run_sharpening_operators.py --family dog`.
- **A2 smooth-then-sharpen companion (EXPLORATORY, dossier §8 (a) A2)**: sharpen an already-smoothed arm (operand = a ring-agg arm, `S` recomputed on it) — directly tests the "recover HF the smoother cost" recovery hypothesis. Producer `run_sharpening_operators.py --family unsharp --operand <arm> --recompute-smoother`.
- **Column-naming note (AD-1, from the producer)**: sharpened parquets **RETAIN the operand's original column names** (`A00..A63` / `hex2vec_0..49` / `R00..R63`), NOT `H000..H177`. Sharpening is a per-column linear combination of same-columned frames, so block identity is preserved 1:1 and the frozen predecessor's measure harness reuses the sharpened arms unchanged. The reserved `H` prefix (dossier §8 registry stamp) + the per-column transform are recorded in the `MODALITY_REGISTRY` `sharpen` row and each `*.run.yaml` sidecar — the prefix is provenance metadata, not the on-disk column names.
- **DNN probe (linear)**: `stage3_analysis/dnn_probe.py::DNNProbeRegressor` with `--num-layers 0` — a single `nn.Linear(input→output)`, no activation, MSE loss = **linear regression** (CLAUDE.md §Stage 3; the project's preferred probe). `--num-layers 1` (one hidden layer) is the exploratory capacity arm. Classification uses `DNNClassificationProber` (softmax analog). Runner `scripts/stage3/run_arm_probe_sweep.py` (commit `de61161`).
- **Target spatial sharpness (the stratifier)**: how spatially concentrated a target field is. Computed per target column by `scripts/stage3/run_spatial_stats.py --target` (commit `de61161`): global **Moran's I** + **Geary's C** (continuous columns) and **k-ring spatial entropy** on quantile/native-class bins (the **cross-target currency** — the only measure spanning both continuous and categorical targets). **Low Moran's I / low ring-entropy = SHARP** (concentrated); high = SMOOTH. Frozen definition + binning rule in §Governance.
- **ΔR²(λ)**: for a sharpened arm at dose λ and a target column `c`, `ΔR²_{c}(λ) = R²(arm at λ) − R²(C10 base)`. The dose-response quantity S-H2 fits its interaction slope on.
- **Sharp-bin / smooth-bin**: the two strata of target columns, assigned by the frozen median split on the stratifier score (§Governance) — fixed before any probe runs.
- **SESOI**: "smallest effect size of interest" — the smallest change treated as real rather than noise; frozen per-measure below, before any value exists.

---

## Governance (the integrity contract)

1. **Commit-then-run (order gate).** This file is committed **before** the first governed producer or probe run; its freeze sha predates every governed result (`.claude/rules/artifact-provenance.md` clause (c); `specs/preregistration_science_lane.md` §(c)). The sharpening producer (`de61161`), probe runner (`de61161`), and stratifier runner (`de61161`) are committed before they run; every producer sidecar `git_hash` names the code that ran. Freeze order verified by `scripts/science/order_gate.py` Mode A (reporter; expect all `NO-EVIDENCE-FOUND` pre-run, then all `PASS` post-run).
2. **Frozen predecessor untouched.** `reports/2026-07-18-smooth-vs-learn-res10/` (freeze `6ff336a`) is READ-ONLY substrate. Its 9 smoothed arms (C10, R10k1e, R10k5e, R10k10e, R10k10lin, R10k10flat, W10full, W10core, W10sect) are **INPUTS** to this campaign, consumed under their own provenance. **Probing them is consumption, not amendment** — the predecessor was probe-free by construction, so the entire probe surface is this campaign's (dossier §8 stamp: *"the probe surface is entirely this campaign's, so probing frozen-campaign artifacts is consumption, not amendment"*). No frozen cell is edited; no post-hoc cell is added to `6ff336a`.
3. **All enumerated cells reported** — nulls, ties, prediction-contradicting values included. New cells require a new pre-registration.
4. **Verdicts quote their rule** verbatim beside their evidence table.
5. **Confirmatory vs exploratory marked**: S-H1, S-H2, S-H3 CONFIRMATORY; DoG geometry, A2 recovery, 1-hidden-layer robustness, windmill-arm probing, and per-family patterns EXPLORATORY/descriptive; never relabeled.
6. **Retired numbers**: none retired by this campaign.
7. **Novel-method gate — not triggered.** Both operator families are **deterministic linear filters, no training, no RNG** (dossier §8 (e): Wiener DROPPED as heavy compute; triplet DROPPED — a contrastive objective on a frozen sampling geometry learns the sampling rule, not content). The only stochasticity is in the **probes** (DNN training + random folds), which is *analysis of existing artifacts*, not a novel model/objective/representation — `specs/preregistration_science_lane.md` §Implementation Notes ("pre-registering probes of an existing artifact is analysis, not a novel-method foundation — no human gate"). No `novel-research-escalate-dont-default.md` hard gate applies.

### Locked protocol constants

| Constant | Value | Source |
|---|---|---|
| Study area / resolution / year | netherlands / **res10 ONLY** / 2022 embedding vintage | dossier §8 (resolution) `[decided-by-human:2026-07-19]` ("res 10 like in the existing report") |
| Confirmatory operand `x` | raw z-scored **C10** (178D, 3,752,940 hexes) | dossier §8 (a) A1 `[decided-by-human:2026-07-19:delegated]` |
| Confirmatory sharpener | **unsharp** `H(x)=(1+λ)x−λS(x)`, whole-vector after per-block z-scoring (D1) | dossier §8 (b) B1 + (d) D1 `[…:delegated]` |
| Confirmatory surround `S` | frozen **`R10k10e`** (k10 exponential ring-agg), REUSED as parquet (A1) | dossier §8 (a) blur sub-choice; task design |
| Confirmatory dose axis | λ ∈ {0} (identity=C10) ∪ **{0.25, 0.5, 1, 2, 4}** (5-point grid, C2) — FROZEN from the 2026-07-19 bracketing pilot (SH res10 subset, 240,114 cells: ring1-cosine and Moran(PC1) strictly monotone across the whole bracket; PR peaks at λ=4 = the boundary dose) | dossier §8 (c) C2 `[…:delegated]`; pilot `data/study_areas/netherlands/stage2_multimodal/sharpen/pilot/2026-07-19/` |
| Dose upper bound | **λ=8 EXCLUDED from the confirmatory grid** (descriptive explosion exhibit only). FROZEN stability checks, any of which rejects an arm from confirmatory standing: (i) participation ratio non-monotone in dose; (ii) variance inflation > 5× the λ=0 baseline; (iii) per-cell explosion fraction (norm > 5× λ=0 norm) > 1e-4. All three first fire at λ=8 in the pilot; none fire at λ≤4 | dossier §8 (c) `[…:delegated]`; inverse-D-H1 guard; pilot ibid. |
| Exploratory sharpened arms | DoG **c0-g1-a2-4** (B2); A2 smooth-then-sharpen (operand `R10k10e`); unsharp with `S`=`R10k1e`/`R10k5e` (dose-of-smoother) | dossier §8 (a)/(b); task design |
| Existing smoothed arms probed | C10 + R10k1e + R10k5e + R10k10e + R10k10lin + R10k10flat + W10full + W10core + W10sect (INPUTS from `6ff336a`) | dossier §8 (scope addition) `[decided-by-human:2026-07-19]` |
| Probe (confirmatory) | `DNNProbeRegressor` / `DNNClassificationProber`, **`--num-layers 0`** (linear); random k-fold **K=5**; **seed pinned** (see §Statistical-honesty seeds row) | dossier §8 (f) F1a `[…:delegated]` |
| Probe (exploratory robustness) | `--num-layers 1` (one hidden layer) | dossier §8 (f) F1b `[…:delegated]` |
| Regression targets | **leefbaarometer** (2024): lbm, fys, onv, soc, vrz, won; **RUDIFUN** (2022, vintage-aligned): fsi, gsi, mxi | dossier §8 (f) `[…:delegated]`; task design |
| Classification target | **urban_taxonomy** (2025): type_level2 → macro-F1 (+ accuracy) | dossier §8 (f) `[…:delegated]`; task design |
| Stratifier | per-target-column spatial sharpness via `run_spatial_stats.py --target`; **frozen currency = k-ring spatial entropy** (spans continuous+categorical) | dossier §8 (f); task design |
| Sharpness bin rule | **global median split** on the frozen currency across all 10 probed columns; below-median = SHARP, at-or-above = SMOOTH (ties → SMOOTH, conservative for the sharpening hypothesis) | this document (freeze proposal) |
| Stratifier k-ring scale | k=3 rings (windmill-core scale); decile (10-quantile) bins for continuous columns, native classes for categorical | this document (freeze proposal) |
| Metric CRS | EPSG:28992 asserted for any metric spatial op | `.claude/rules/domain-guardrails.md` rider |
| region_id | canonical index; asserted (never silently renamed) on every arm/target load | `.claude/rules/index-contracts.md`, `.claude/rules/no-fallbacks.md` |

**All values are now concrete — no unfrozen placeholders remain.** The λ grid and stability bound were filled from the **declared-exploratory** bracketing pilot (res10 South Holland subset, 240,114 cells, run 2026-07-19 BEFORE freeze; outputs + sidecar at `data/study_areas/netherlands/stage2_multimodal/sharpen/pilot/2026-07-19/`). Next step is the freeze commit, with the sha recorded above.

---

## Statistical-honesty declarations (one row per `specs/preregistration_science_lane.md` §(d) field)

- **Seeds & seed-noise band**: unlike the frozen predecessor (fully deterministic, seeds n/a), THIS campaign's **probes are stochastic** (DNN training init + random fold assignment). Per §(d), every headline probe number is a **mean ± sd over ≥3 seeds**. **FROZEN: confirmatory probes run at seeds `{42, 43, 44}`, K=5 folds each** (science-lane §(d) ≥3-seed compliant; matches the predecessor res9 campaign's seed set); each cell's headline R²/macro-F1 = mean over the 3 seeds, and S-H1/S-H2/S-H3 decision thresholds are expressed in units of the seed sd where measurable (frozen precedent: `2026-07-16` ε=0.0005 band). Seed is EXPLICIT and recorded in every probe sidecar (a null/unrecorded seed is a bug — `[[artifact-provenance]]`).
- **CI on every headline number**: a bootstrap CI (resample over held-out hexes / folds) accompanies every reported probe R²/macro-F1 and every ΔR². A number with no interval is not a headline number. — `scripts/science/stats.py: bootstrap_ci`.
- **Multiple-comparison correction**: CONFIRMATORY family = {S-H1 (3 monotonicity components), S-H2 (interaction slope + G1 floor), S-H3 (smoothing-effect sign on the sharp bin)}. **Holm–Bonferroni** across the confirmatory family; exploratory/robustness/atlas rows excluded (exclusion declared). — `stats.py: holm`.
- **Severity**: per H, below.
- **SESOI** (declared now, before any value exists):
  - **Probe-free sharpness measures** (S-H1, mirroring the frozen campaign): Moran's I **0.02** absolute; ring-1 cosine **0.01** absolute; PR **2%** relative; per-dim entropy **0.05** bits (descriptive 4th).
  - **Probe R²** (S-H2 G1 floor, S-H3): δR² = **0.01** absolute R².
  - **Classification macro-F1**: **0.01** absolute.
  - **Interaction slope β_min** (S-H2 primary): **0.005 ΔR² per unit λ** — FROZEN. Over the frozen grid's span (λ: 0→4) this demands a cumulative sharp-vs-smooth divergence of ≥ 0.02 ΔR², i.e. 2× the G1 δR² floor across the full dose range: an interaction any smaller is real-but-immaterial at this probe scale.
- **Paired tests**: all cross-arm probe comparisons are **paired by hexagon** — every arm shares the C10 frame (3,752,940 hexes) exactly, and folds are assigned once on the shared frame and reused across arms, so ΔR² is a within-fold paired difference. Continuous: paired permutation; classification: McNemar. — `stats.py: paired_permutation`, `mcnemar`.
- **Claim scope**: verdicts generalize over `{ netherlands res10, 2022 embedding vintage; the C10 178D fabric and its unsharp-sharpened variants with S=R10k10e; the λ grid as frozen; targets leefbaarometer (2024) + RUDIFUN (2022) + urban_taxonomy (2025) as enumerated; linear DNN probe (num-layers 0), random K=5 folds, seeds as frozen }`. **NOT covered**: any learned sharpener (none — Wiener/triplet dropped, §8 (e)); other λ grids/surrounds; other regions/vintages/resolutions; spatial-block CV (the `[[domain-guardrails]]` P1 park — random folds only here). Every `N=1` axis (single target per family for a column, single embedding vintage) is named in the verdict.
- **Vintage-mismatch honesty (`[[domain-guardrails]]` gate (i))**: embeddings = **2022**; leefbaarometer = **2024**; urban_taxonomy = **2025**; RUDIFUN = **2022 (vintage-ALIGNED)**. Only RUDIFUN is temporally aligned to the embeddings — the leefbaarometer (+2y) and taxonomy (+3y) probes carry a real temporal mismatch, stated in every result sidecar and reported beside every leef/taxonomy number. This is a claim-scope limit, not a defect to hide: a sharp-target recovery seen on RUDIFUN (aligned) is stronger evidence than the same on leefbaarometer (mismatched).
- **Edge-effects caveat** (prose-only per `[[domain-guardrails]]`): ring/DoG neighbourhoods truncate at the coast/border; identical across arms on the shared C10 frame, cancels in paired comparisons; absolute levels near borders biased. Stated, not gated.

---

## Hypotheses

### S-H1 — Sharpening dose-response (manipulation check) (CONFIRMATORY — confirmatory gate)

- **Background**: unsharp masking is mechanically a high-pass re-amplifier; if it actually sharpens, its signature must appear monotonically in dose λ — the **mirror of the predecessor's D-H1** (which validated that ring-agg actually smooths). This validates the sharpened arms before S-H2/S-H3 probe them. **The dossier's inverse-D-H1 risk is the specific failure mode**: sharpening can *explode* (noise blow-up) instead of moving measures cleanly — the [STABILITY-BOUND] on the λ grid exists to keep the tested doses in the non-exploding regime, and S-H1 is the check that it worked.
- **Prediction (directional, pre-registered)**: along the frozen dose axis λ = 0 (C10) → λ₁ → … → λ₅, in the **sharpening direction** (opposite to smoothing): (i) mean Moran's I over PC1..8 **strictly decreases**; (ii) ring-1 neighbor-cosine **strictly decreases**; (iii) PR (effective dimensionality) **strictly increases** — each step ≥ its SESOI. (Per-dim entropy reported descriptively as a 4th measure; not in the gate, mirroring D-H1's three-component rule.)
- **Decision rule (numeric, frozen)**:

  | Observed outcome | Verdict |
  |---|---|
  | All three components hold monotonically at every step | **fail-to-reject** — the sharpener is validated; S-H2/S-H3 proceed with full confirmatory standing |
  | Exactly one component fails (sub-SESOI step or reversal) | **downgrade** — that measure flagged unreliable, dropped from any measure-based reasoning; reported loud |
  | ≥2 components fail | **reject** — the operator does not track dose (saturation or explosion); S-H2 and S-H3 auto-downgrade to descriptive (**stop rule S2**) |
  | any step shows an *explosion* signature (PR or Moran jumps non-monotonically past a full SESOI in the wrong direction, i.e. the noise-blow-up branch) | **reject + flag EXPLOSION** — the [STABILITY-BOUND] was set too high; reported as the committed inverse-D-H1 headline |
- **Severity**: if unsharp did not actually sharpen (or exploded), consecutive SESOI-exceeding steps in the predicted directions would be very unlikely; a clean pass is a genuine test the operator could have failed.
- **RESULT** (filled 2026-07-20 from `data/verdicts_2026-07-20.md` §D-SH1): all 15 step×component checks pass — Moran-mean strictly ↓ (steps −0.023 to −0.096, each ≥ 0.02 SESOI), ring-1 cosine strictly ↓ (−0.020 to −0.084, each ≥ 0.01), PR strictly ↑ (+2.2% to +10.4% rel, each ≥ 2%); 0 components failing; no explosion signature anywhere on the grid.
- **VERDICT**: **fail-to-reject** — the dose axis is a valid sharpening instrument across the whole frozen grid (no S2 cascade; S-H2/S-H3 keep confirmatory standing).

### S-H2 — Dose × target-sharpness interaction (PRIMARY, CONFIRMATORY)

- **Background**: the load-bearing shape (dossier §8 (f)/(g), user's framing). NOT "sharpening raises mean R²" — that would be a fishing expedition. The claim is that the **sharpening benefit grows with the target's spatial sharpness**: sharp targets recover what smoothing cost; smooth targets are flat-or-hurt. Targets are binned sharp-vs-smooth by the **frozen** stratifier score BEFORE any probe runs.
- **Prediction (directional, pre-registered)**: fit `ΔR²_{c}(λ) ~ β0 + β1·λ + β2·sharp_c + β3·(λ × sharp_c)` over all confirmatory (arm-λ × target-column) cells, where `sharp_c ∈ {0=smooth, 1=sharp}` is the frozen bin and λ is the frozen dose. (a) **interaction**: the dose×sharpness slope `β3` is positive and ≥ **β_min = 0.005 ΔR² per unit λ** — equivalently, mean ΔR²-vs-λ slope over sharp-bin columns exceeds that over smooth-bin columns by ≥ β_min; AND (b) **G1 absolute floor**: the best confirmatory sharpened arm beats the C10 baseline by ≥ **δR² = 0.01** absolute R² on **at least one sharp-bin target family** (leefbaarometer OR RUDIFUN), CI-separated from 0.
- **Decision rule (numeric, frozen)**:

  | Observed outcome | Verdict |
  |---|---|
  | (a) β3 ≥ β_min AND (b) G1 floor met on ≥1 sharp-bin family | **fail-to-reject** — sharpening recovers probe signal on spatially-sharp targets (scoped to this operator + probe + targets) |
  | (a) β3 ≥ β_min AND (b) no arm clears the G1 floor | **downgrade** — interaction present but below practical size: "sharp targets respond to dose, but no arm beats baseline by ≥ δR²"; reported as a real-but-tiny effect |
  | (a) β3 within ±β_min of 0 (no interaction) | **reject the interaction** — sharpening benefit does NOT scale with target sharpness; the motivating shape is not supported by these measures, reported as the headline |
  | (a) β3 ≤ −β_min (reversed interaction) | **reject + flag REVERSAL** — sharpening helps SMOOTH targets more than sharp ones (contradicts the framing); committed reportable |
  | primary × confirmatory | S-H1 must fail-to-reject first; if S-H1 rejects, this verdict is **descriptive** (S2) |
- **Severity**: a false S-H2 (no interaction, or reversed) lands in the β3≈0 or β3≤−β_min branches — reachable, specific failure regions, two of which contradict the motivating belief.
- **RESULT** (filled 2026-07-20 from `data/verdicts_2026-07-20.md` §D-SH2): β3 = −0.0031, 95% CI [−0.0124, +0.0042] (seeded bootstrap; β_min = 0.005) — within ±β_min of 0. Every ΔR²(λ) is NEGATIVE on both bins (sharpening reduces probe R² at every dose; least at λ=0.25). G1 floor not cleared on any sharp-bin family (best deltas −0.010 leefbaarometer:vrz, −0.006 RUDIFUN — CI-separated BELOW zero). The classification column (type_level2) is excluded from the ΔR² OLS (ΔR² undefined for macro-F1) and carried descriptively in D-EXP — prereg-ambiguity note in the provenance appendix. Holm-adjusted p(β3) = 0.53.
- **VERDICT**: **REJECT the interaction** — the sharpening benefit does not scale with target sharpness; sharpening is a monotone cost to linear probes on this fabric.

### S-H3 — Smoothing is non-positive on sharp targets (CONFIRMATORY — the probes-over-smoothed-arms half)

- **Background**: the other half of the story (dossier §8 scope addition — probe the existing smoothed arms). If sharp targets carry high-spatial-frequency signal, then *smoothing* them should **cost** probe performance (or at best do nothing) — the complement of S-H2. This is the empirical grounding for "smoothing hurts sharp targets."
- **Prediction (directional, pre-registered)**: along the exponential smoothing dose C10 → R10k1e → R10k5e → R10k10e (the un-saturated-then-saturated chain), the **mean probe R² on sharp-bin regression targets is non-increasing** — no step *improves* sharp-bin R² by ≥ δR² (0.01). (The linear/flat/windmill smoothers and smooth-bin targets are reported descriptively for the full picture.)
- **Decision rule (numeric, frozen)**:

  | Observed outcome | Verdict |
  |---|---|
  | No smoothing step raises sharp-bin mean R² by ≥ δR² (effect ≤ 0 within SESOI) | **fail-to-reject** — smoothing is non-positive on sharp targets, as predicted |
  | Sharp-bin mean R² is flat within ±δR² across the smoothing chain | **downgrade** — "smoothing is a sub-δR² knob on sharp targets" (smoothing neither helps nor hurts them measurably) |
  | Any smoothing step raises sharp-bin mean R² by ≥ δR² | **reject** — smoothing *improves* sharp-target probing; contradicts the framing, reported as the headline |
  | primary × confirmatory | descriptive if S-H1 rejects (S2) |
- **Severity**: a false S-H3 (smoothing helps sharp targets) shows up directly as a ≥ δR² improving step — a specific, reachable failure region.
- **RESULT** (filled 2026-07-20 from `data/verdicts_2026-07-20.md` §D-SH3): smoothing IMPROVES sharp-bin mean probe R²: C10 0.5556 → k1 0.5715 → k5 0.5802 → k10 0.5808. The C10→k1 step is +0.0159 ≥ δR²=0.01 (paired permutation p = 1e-4; Holm-adjusted 2e-4). The predicted non-positive dose-response is contradicted at the first step.
- **VERDICT**: **REJECT** — smoothing helps even spatially-sharp targets; neighborhood aggregation adds probe-relevant information rather than destroying it. Reported as a committed headline.

### Exploratory hypotheses (EXPLORATORY, descriptive — no verdicts)

- **E-DoG**: the DoG annulus-with-gap arm (c0-g1-a2-4) probe performance vs the unsharp arms — does the band-pass geometry (drops both regional and per-hex content) beat or trail one-knob unsharp on sharp targets? Reported descriptively.
- **E-A2**: the smooth-then-sharpen companion (operand `R10k10e`, sharpened) — does sharpening an already-smoothed arm *recover* HF that smoothing cost, back toward or past C10 on sharp targets? The direct "recovery" test, descriptive.
- **E-1hidden**: F1b one-hidden-layer probe on the confirmatory arms — does the S-H2 interaction survive a nonlinear probe, or is it a linear-probe artifact? Robustness, descriptive.
- **E-windmill**: probe performance of the windmill arms (W10full/W10core/W10sect) — characterizes the predecessor's "strong smoother" arms on the supervised axis (the predecessor's D-H3 was structure-only). Descriptive.
- **E-family**: per-target-family patterns (leefbaarometer sub-domains lbm/fys/onv/soc/vrz/won; RUDIFUN fsi/gsi/mxi; taxonomy type_level2) — which specific targets drive any interaction. Feeds a future pre-registration; no verdict here.

---

## The Full Grid

Arms — **existing (INPUT, from `6ff336a`)**: `C10` (base/anchor), `R10k1e`, `R10k5e`, `R10k10e`, `R10k10lin`, `R10k10flat`, `W10full`, `W10core`, `W10sect`. **New sharpened (this campaign)**: confirmatory unsharp `U10λ1..U10λ5` (S=`R10k10e`, λ ∈ {0.25, 0.5, 1, 2, 4} frozen); exploratory `DoG10` (c0-g1-a2-4), `SU10` (A2 smooth-then-sharpen, operand `R10k10e`), `U10·Sk1`/`U10·Sk5` (dose-of-smoother surrounds), `U10λ8` (descriptive explosion exhibit, EXCLUDED from all confirmatory analyses per the stability bound).

Target columns — **leefbaarometer** (2024, regression R²): lbm, fys, onv, soc, vrz, won. **RUDIFUN** (2022, regression R²): fsi, gsi, mxi. **urban_taxonomy** (2025, classification macro-F1): type_level2. **= 10 confirmatory target columns.** (RUDIFUN l/osr and taxonomy other type_levels available exploratory.)

### Phase A — Producer cells (serial, each gated: sidecar parses; rows = 3,752,940 = C10 frame; NaN/inf 0; region_id-aligned to C10 → fail = stop rule S1)

| Cell | produces | command (frozen; commit `de61161`) |
|---|---|---|
| P-u1 … P-u5 | U10λ1 … U10λ5 | `python scripts/stage2/run_sharpening_operators.py --family unsharp --smoother-arm netherlands_res10_2022_k10_exponential.parquet --lam 0.25,0.5,1,2,4` (one output per λ) |
| P-dog | DoG10 | `... --family dog --dog-center-ring 0 --dog-gap 1 --dog-annulus 2,3,4 --dog-wc 1.0 --dog-ws 1.0` |
| P-a2 | SU10 | `... --family unsharp --operand netherlands_res10_2022_k10_exponential.parquet --recompute-smoother --smoother-K 10 --smoother-weighting exponential --lam 1` |
| P-usk1 | U10·Sk1 | `... --family unsharp --smoother-arm netherlands_res10_2022_k1_exponential.parquet --lam 1` |
| P-usk5 | U10·Sk5 | `... --family unsharp --smoother-arm netherlands_res10_2022_k5_exponential.parquet --lam 1` |

*(The 9 existing smoothed arms are NOT produced here — they are INPUTS from `6ff336a`, consumed under their own provenance per Governance item 2. Producer output filenames resolve by sidecar per `[[artifact-provenance]]` (a).)*

### Phase B — Stratifier cells (run AFTER freeze, BEFORE any probe — the frozen sharpness scores + bins)

| Cell | produces | command (frozen; commit `de61161`) |
|---|---|---|
| ST-leef | leefbaarometer per-column sharpness (Moran/Geary + k3-ring entropy) | `python -u scripts/stage3/run_spatial_stats.py --target leefbaarometer --h3-resolution 10` |
| ST-rudi | RUDIFUN per-column sharpness | `... --target rudifun --h3-resolution 10` |
| ST-tax | urban_taxonomy per-column k3-ring entropy (native classes) | `... --target urban_taxonomy --h3-resolution 10` |
| ST-bin | the global median split → sharp/smooth bin assignment per column | derived from ST-leef/ST-rudi/ST-tax on the frozen currency (k3-ring entropy), median across all 10 columns; ties → SMOOTH |

**The bin assignment (ST-bin) is frozen the moment it is computed and committed — no probe result may move a column between bins.** The stratifier scores are mechanical outputs of the frozen definition; computing them is not a degree of freedom.

### Phase C — Probe cells (confirmatory: `--num-layers 0`, K=5, seeds frozen)

The confirmatory probe grid is **{14 arms} × {10 target columns} × {num-layers 0}**, each cell one probe (mean ± sd over the frozen seeds, with bootstrap CI):

- **14 arms** = C10 + U10λ1..λ5 (6 on the confirmatory sharpening dose axis, carrying S-H2) + R10k1e/R10k5e/R10k10e/R10k10lin/R10k10flat/W10full/W10core/W10sect (8 smoothed, carrying S-H3 + E-windmill).
- **10 target columns** as enumerated.
- **= 140 confirmatory probe cells.** Every cell reported (R²/macro-F1 + CI + seed-sd), nulls included.
- **Exploratory probe extension** (descriptive): + DoG10/SU10/U10·Sk1/U10·Sk5 arms (× 10 columns), + all confirmatory arms re-probed at `--num-layers 1` (E-1hidden), + RUDIFUN l/osr and other taxonomy levels. Reported, verdict-free.

Runner: `python -u scripts/stage3/run_arm_probe_sweep.py --arms-manifest reports/2026-07-19-sharpen-vs-smooth/arms_manifest.txt --targets leefbaarometer,urban_taxonomy,rudifun --resolution 10 --num-layers 0 --seed <frozen> --out-dir data/study_areas/netherlands/stage3_analysis/2026-07-19-sharpen-probes` (resumable, skip-if-exists per cell).

### Phase D — Derived cells (verdicts)

| Cell | derived over | rule | feeds | in correction family? | RESULT |
|---|---|---|---|---|---|
| D-SH1 | probe-free sharpness (Moran/ring-cos/PR) of {C10, U10λ1..λ5} | rule S-H1 | S-H1 | yes | **fail-to-reject** (15/15 SESOI steps, no explosion) — 2026-07-20 |
| D-SH2 | ΔR²(λ) over {C10, U10λ1..λ5} × sharp/smooth bins | rule S-H2 | S-H2 | yes | **REJECT** (β3=−0.003 ∈ ±β_min; all ΔR² negative; G1 not cleared) — 2026-07-20 |
| D-SH3 | probe R² over {C10, R10k1e, R10k5e, R10k10e} × sharp bin | rule S-H3 | S-H3 | yes | **REJECT** (C10→k1 +0.016 ≥ δR², p_adj 2e-4; smoothing helps sharp bin) — 2026-07-20 |
| D-EXP | DoG / A2 / 1-hidden / windmill / family observations | descriptive | exploratory | no | descriptive — see `data/verdicts_2026-07-20.md` §D-EXP + report atlas |

**Cell count**: 9 producer (5 unsharp + DoG + A2 + 2 dose-of-smoother) + 4 stratifier + 140 confirmatory probe (+ exploratory extension) + 4 derived. Every confirmatory cell is a row in the final report.

### Stop rules (halt-branch-continue-queue, per `specs/preregistration_science_lane.md` §(e))

- **S1**: a producer cell failing its gate (row-count ≠ C10 frame, NaN/inf, region_id misalignment) halts only that arm's probe cells; independent arms continue; no improvised replacements.
- **S2**: S-H1 = reject → D-SH2 and D-SH3 verdicts auto-downgrade to descriptive; probe cells still run and report.
- **S3**: any sharpened arm whose frame ≠ C10's 3,752,940 exactly, or whose region_id set differs → halt that arm (frame/index drift = producer bug), diagnose before probing.
- **S4** (dose stability): if the pilot's [STABILITY-BOUND] check flags a λ in the grid as explosion-prone, that λ is NOT frozen into the grid — halt and re-bracket rather than probe an unstable arm.

### Per-example / per-cell persistence

Per-cell probe JSONs (R²/macro-F1, per-fold, per-seed) + provenance sidecars → `data/study_areas/netherlands/stage3_analysis/2026-07-19-sharpen-probes/{arm}/{target}/`; per-column stratifier scores → the target's `diagnostics/{date}/` dir (via `run_spatial_stats.py`); summary CSVs → `reports/2026-07-19-sharpen-vs-smooth/data/`.

---

## What Would Change Our Minds

*To be reproduced verbatim at the top of the final report (house precedent — the predecessor README §3 reproduces its block; every verdict quotes the rule it answers to).*

- **S-H1 is rejected** if ≥2 of {Moran-mean, ring-1 cosine, PR} fail strict SESOI-monotonicity in the sharpening direction across the λ dose axis — the operator does not track dose (saturation or **explosion**); S-H2/S-H3 become descriptive (S2). An explosion signature (a non-monotone SESOI-crossing jump) is the committed inverse-D-H1 headline: sharpening blew up rather than helped.
- **S-H2 (the primary interaction) is rejected** if the dose×sharpness slope β3 is within ±β_min of 0 (no interaction) — the sharpening benefit does NOT scale with target sharpness, the motivating shape unsupported; we commit now to reporting that. It **flags REVERSAL** if β3 ≤ −β_min (sharpening helps smooth targets more than sharp ones), and **downgrades** if the interaction is present but no arm clears the δR²=0.01 G1 floor (real but practically tiny).
- **S-H3 is rejected** if any exponential smoothing step *raises* sharp-bin mean probe R² by ≥ δR²=0.01 — smoothing improves sharp-target probing, contradicting "smoothing hurts sharp targets"; committed reportable. It **downgrades** to "sub-δR² knob" if smoothing is flat within ±δR² on sharp targets.
- **The exploratory rows have no gate** — DoG/A2/1-hidden/windmill/family observations feed future pre-registrations, never verdicts here.
- **Vintage caveat on every verdict**: leefbaarometer (2024) and taxonomy (2025) results carry a temporal mismatch to the 2022 embeddings; only RUDIFUN (2022) is aligned. A verdict resting on a mismatched target names that limit.

---

## Cross-references

- **Foundations (STAMPED)**: `specs/sharpening_operators_dossier.md` §8 (`[decided-by-human:2026-07-19]` + delegated) — every governed choice above cites its stamp; §3.1 (unsharp math), §3.2 (DoG), §1 (D-H1 saturation grounding).
- **Governance contract**: `specs/preregistration_science_lane.md` (RATIFIED 2026-07-18) — report-form §(a), order gate §(c), statistical-honesty §(d), halt-branch-continue-queue §(e), report layer §(g).
- **Frozen predecessor (READ-ONLY substrate / INPUT arms)**: `reports/2026-07-18-smooth-vs-learn-res10/PREREGISTRATION.md` + `README.md` (freeze `6ff336a`) — D-H1 REJECT / exponential saturation; the 9 smoothed arms this campaign probes.
- **Producers (commit `de61161`)**: `scripts/stage2/run_sharpening_operators.py` (unsharp + DoG); `scripts/stage3/run_arm_probe_sweep.py` (arm×target DNN probe sweep); `scripts/stage3/run_spatial_stats.py --target` (the frozen stratifier). Smoother: `stage2_fusion/models/ring_aggregation.py`. Concat: `stage2_fusion/concat.py`.
- **Probes**: `stage3_analysis/dnn_probe.py` (`--num-layers 0` = linear), `dnn_classification_probe.py`; targets `leefbaarometer_target.py`, `urban_taxonomy_target.py`, RUDIFUN loader.
- **Order-gate reporter**: `scripts/science/order_gate.py` (Mode A). **Stats helpers**: `scripts/science/stats.py` (`bootstrap_ci`, `holm`, `paired_permutation`, `mcnemar`, `seed_band`, `sesoi_check`).
- **Registry**: `stage1_modalities/__init__.py::MODALITY_REGISTRY` (`sharpen`/`H` row, `in_prefix_map=False`, zeropad3 — dossier §8 registry stamp; AD-1 column-naming note above).
- **Rules**: `.claude/rules/artifact-provenance.md` (commit-then-run; sidecar-read), `.claude/rules/domain-guardrails.md` (CRS/region_id/vintage gates), `.claude/rules/poi-filter-discipline.md` (C10's POI block = filter_v2), `.claude/rules/viz-discipline.md` (report figures), `.claude/rules/training-discipline.md` (serial-CPU — sharpening + probes queue behind any live peer heavy job).
