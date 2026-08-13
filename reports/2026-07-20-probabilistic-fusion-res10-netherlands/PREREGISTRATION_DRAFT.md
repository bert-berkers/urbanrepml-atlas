# PRE-REGISTRATION DRAFT — geo-MPC vs full-area fusion on identical probes

> **DRAFT for morning review — NOT registered. No order-gate receipts. Campaign cells NOT authorized
> tonight. Every foundational choice below is `[blocked:human-decision]`.** This document lays out
> options + evidence for the human to decide; it closes nothing. Produced by stage2-fusion-architect,
> session stone-gathering-storm, 2026-07-20, as item 6e. Per
> `.claude/rules/novel-research-escalate-dont-default.md`: an agent may *inform* a foundation, never
> *ratify* it. No `[decided-by-human]` tag appears here because no decision has been made.

---

## 1. Purpose + attribution scope (read first — this bounds every claim)

**Question**: Does cone-sampled probabilistic fusion (geo-MPC) produce embeddings that predict urban
outcomes **better than, worse than, or the same as** the full-area fusion baselines, when both are
probed on identical targets with identical folds?

**Attribution scope — honest**: This campaign tests the **architecture class as it stands today**, not
a tuned contender. The geo-MPC's current best arm (a7) is **statistically zero against its own
untrained init**: Δ = +0.0040 mean OOF R², p ≈ 0.57
(`reports/2026-07-20-geo-mpc-windmill-hillclimb/README.md`). So the pre-registered expectation is
**not** "geo-MPC wins" — it is "measure where this class actually sits vs full-area fusion, on a fair
probe, with the honest null clearly in view." A result of "geo-MPC ties or loses" is a **publishable,
expected** outcome under this framing, not a failure.

**Out of scope (named as follow-up, NOT part of this campaign)**: a **cone-VAE port** (a
variational-autoencoder fusion over the cone hierarchy) is a plausible stronger contender but is
*unbuilt* and *unspecified*. It is explicitly follow-up work — do not fold it into this campaign, and
do not let "geo-MPC lost, but cone-VAE might win" leak into this campaign's conclusions.

---

## 2. Foundational choices — ALL `[blocked:human-decision]`

Each row lays out options + evidence and stops. None is closed. (`.claude/rules/novel-research-escalate-dont-default.md`.)

| # | Foundational choice | Options | Evidence to weigh | Tag |
|---|---|---|---|---|
| F1 | **Trainer architecture for res10** | (a) full-resident; (b) cone-lazy `LazyConeBatcher`; (c) res9-only, defer res10 | Full-resident **OOMs at res10** (`scaling_analysis.md` VRAM table: ~2.4M anchors → tens of GB > 24 GB). Cone cache exists (284 cones, 12 GB) but no geo-MPC consumes it. | `[blocked:human-decision]` |
| F2 | **What "the cone / glimpse IS" at res10 scale** | (a) keep the res9 windmill (9/10 streams, image_radius 8) tiled per-anchor; (b) redefine the glimpse as a res5-rooted cone subtree (~1,500 hexes, res5→res10); (c) hybrid | This is the *paper-mapping* question the novel-research gate exists for — the windmill geometry was human-ratified for res9 (2026-07-04); res10 scale may change what a glimpse should be. **Do not default.** | `[blocked:human-decision]` |
| F3 | **Anchor-set definition for NL res10 (`is_anchor` semantics)** | (a) all cells anchors; (b) a density/coverage-filtered subset (as South Holland N6); (c) cone-root-derived anchors | NL res10 regions_gdf has **no `is_anchor` column** (verified); South Holland's was carved by the N6 rule. The semantics at national scale are undefined. | `[blocked:human-decision]` |
| F4 | **Readout convention** | (a) `average` (fixation-averaged core-disc latents — what the trainer emits today); (b) `anchor` (single central fixation) | Worth ~0.04–0.06 R² difference on identical weights: untrained twin scores 0.4310 (average) vs 0.4741 (anchor) (`hillclimb README` Warning 2). The comparison to any historical bar depends on this. | `[blocked:human-decision]` |
| F5 | **Does geo-MPC enter as res9-only or res10?** | (a) res9-only this campaign (feasible today after minor fixes); (b) block the campaign on the res10 cone-lazy build | Res9 South Holland geo-MPC checkpoints **exist and are probeable today** (a1–a7 on disk). Res10 is blocked on F1–F3. A res9-only campaign is runnable much sooner. | `[blocked:human-decision]` |

**Recommendation-as-decision is forbidden here.** The rows above deliberately do not pick a winner.
The one *engineering* observation (res9-only is runnable sooner) is a fact about readiness, not a
ruling on scope.

---

## 3. Design (conditional on the F-choices above)

Stated as a mechanistic square so the human can see exactly what would be registered once foundations
are ratified. **Not authorized tonight.**

### 3.1 Arms (embeddings compared)

| Arm | Embedding | Status today |
|---|---|---|
| E1 | geo-MPC best arm (a7 geometry: smr5 + k0-fovea + ns40/dt0.05), **res9 South Holland** | checkpoint on disk |
| E2 | full-area res9 fusion baseline — raw 178D concat | buildable |
| E3 | full-area res9 fusion — ring-agg k=10 (current best full-area performer, CLAUDE.md Stage 2) | buildable; tonight's re-baseline lane produced comparators |
| E4 *(optional)* | geo-MPC untrained twin (the honest floor) | inference-only |

Scale of comparison (res9 vs res10) is **F1/F5-dependent**. If res9-only (F5-a), all arms are res9 and
runnable after the `infer_south_holland.py` fix. If res10 (F5-b), the campaign is blocked on F1–F3.

### 3.2 Probe battery + CV scheme (declared choices)

| Element | Choice | Note |
|---|---|---|
| Targets | **leefbaarometer** (liveability composite) + **RUDIFUN** (land-use/density) | both are established probe targets; RUDIFUN + leefbaarometer are the two named in the task |
| Probe model | DNN probe, linear architecture (`--num-layers 0`) = true linear regression (CLAUDE.md Stage 3) | preferred probe; re-baseline OLS with DNN before comparing |
| Seeds | **42, 43, 44** | three seeds per arm × target |
| Metric | out-of-fold R² | reported per seed + mean |
| **CV scheme** | **`[declared fork]`** | see below |

**CV-scheme fork (a declared choice, not an accident)**: two fold regimes exist in the codebase and
they are not interchangeable —
- **campaign-harness random k-fold** (what tonight's ring-agg re-baseline used, for comparability
  across the comparator matrix), vs
- **`dnn_probe` spatial 10 km folds** (geographically blocked, guards against spatial leakage;
  `.claude/rules/domain-guardrails.md` P1 parks spatial-CV-leakage as prose-note, human-skeptical it
  bites at our n).

The campaign **must declare which** at freeze time and **use it for every arm** — mixing regimes across
arms would confound the comparison. Recommendation to the human (a *methods* note, not a foundational
ruling): register **random k-fold** for like-for-like comparability with the existing comparator matrix
tonight's ring-agg re-baseline extends, and optionally add spatial 10 km folds as a declared secondary
robustness pass. This fork is itself surfaced, not silently chosen.

### 3.3 Hypotheses (illustrative — to be frozen only post-ratification)

| ID | Hypothesis | Test |
|---|---|---|
| H1 | geo-MPC (E1) vs raw-concat (E2): sign + magnitude of ΔR² per target | paired across seeds |
| H2 | geo-MPC (E1) vs ring-agg k=10 (E3): does the learned fusion beat the zero-parameter smoother? | paired across seeds |
| H3 | geo-MPC trained (E1) vs untrained twin (E4): does training add value? (the hillclimb's open question, on a fresh probe) | paired across seeds |

SESOI (smallest effect size of interest), significance correction, and any gate thresholds are
**deliberately left unspecified** in this draft — they are set at freeze time under `/science`, once
foundations are ratified.

---

## 4. Governance (what registration WOULD require — not done tonight)

- [ ] Every F1–F5 choice carries `[decided-by-human:<date>]` (novel-research hard gate).
- [ ] Frozen `PREREGISTRATION.md` under a dated campaign dir via `/science register` (freeze-then-run).
- [ ] Order-gate receipt clean.
- [ ] `infer_south_holland.py` geometry-reconstruction bug fixed (else E1 cannot be read out).
- [ ] CV scheme declared and applied uniformly across arms.
- [ ] Commit-then-run; sidecars green; POI = filter_v2 asserted.

**None of these boxes is checked. This is a draft.**

---

## 5. Pointers

- Scaling verdict + memory model: `reports/2026-07-20-probabilistic-fusion-res10-netherlands/scaling_analysis.md`
- geo-MPC current state (a7 = statistically zero): `reports/2026-07-20-geo-mpc-windmill-hillclimb/README.md`
- Trainer: `scripts/mpc/train_south_holland.py`; infer bug: `scripts/mpc/infer_south_holland.py:139-144`
- Cone infra: `stage2_fusion/data/hierarchical_cone_masking.py`; cache `data/study_areas/netherlands/cones/cone_cache_res5_to_10/` (284 cones, 12 GB)
- Probe infra: `stage3_analysis/dnn_probe.py`; targets `leefbaarometer_target.py`
- Discipline: `.claude/rules/novel-research-escalate-dont-default.md` (hard gate), `.claude/rules/domain-guardrails.md` (CV/spatial-leakage park)
