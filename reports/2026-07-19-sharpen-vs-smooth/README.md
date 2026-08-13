# Sharpen-vs-Smooth on Urban Embeddings at Res10 — a Supervised-Probe Campaign

**Status: COMPLETE** — pre-registration frozen `da40124`, results committed `fb1543f`, verdicts independently re-derived, report built.

📄 **[report.pdf](report.pdf)** — the full write-up (methods, related work, supervised probes, maps).
This is **Part 2** of the smoothing story; Part 1 is [`../2026-07-18-smooth-vs-learn-res10/`](../2026-07-18-smooth-vs-learn-res10/) (read-only, frozen — this campaign consumes its artifacts and never amends it).

## The question

Part 1 measured what k-ring *smoothing* does to the 178-dimensional embedding fabric and found the exponential kernel saturates by k=5. It never probed. This campaign inverts the operator — **unsharp masking**, `H(x) = x + λ(x − S(x))`, which subtracts the local mean and adds the high-frequency remainder back with gain λ — and probes **both directions of the dose axis** against real targets for the first time.

The motivating intuition: many urban targets are spatially *sharp* (a block differs from the next block), so a blurred embedding should track them poorly, and re-amplifying the surviving high frequencies should help.

## What we found

**It doesn't. The intuition is falsified in the exact way the pre-registration named in advance.**

| Hypothesis | Verdict | Basis |
|---|---|---|
| **S-H1** — sharpening dose moves probe-free sharpness measures monotonically (manipulation check) | **fail-to-reject** | 15/15 step×component checks pass SESOI; Moran's I ↓, ring-1 cosine ↓, participation ratio ↑ at every step; no explosion |
| **S-H2** — the sharpening benefit grows with target sharpness (primary) | **REJECT the interaction** | β₃ = −0.003, 95% CI [−0.012, +0.004] (β_min 0.005); **every** ΔR² negative on both bins; no arm clears the +0.01 floor |
| **S-H3** — smoothing is non-positive on sharp targets | **REJECT** | sharp-bin mean R² *rises* with smoothing: 0.556 → 0.572 → 0.580 → 0.581; first step +0.016, p_adj 2e-4 |

The instrument worked — S-H1 confirms the operator genuinely sharpens across the whole grid — so the null is about the world, not the tool. Read together: **sharpening monotonically costs linear-probe R², including on the targets selected for being spatially sharp, while smoothing monotonically helps, even on those same sharp targets.**

The interpretation: for liveability, density/function-mix, and morphology, a hexagon's neighbourhood is not noise obscuring its signal — **the neighbourhood *is* signal**. These properties are partly properties of the surroundings, so high-pass filtering discards exactly the contextual information the probes rely on. The dose axis is also asymmetric: smoothing's gains saturate by k=5 (Part 1), while sharpening's costs keep growing out to λ=4.

## Design

- **Fabric**: canonical res10 concat, 3,752,940 hexagons × 178 dims (AlphaEarth 64 + hex2vec 50 + highway2vec 64, per-block z-scored), Netherlands, 2022 vintage.
- **Arms**: 14 confirmatory — C10 baseline, 5 unsharp doses (λ ∈ {0.25, 0.5, 1, 2, 4}), 5 ring-aggregation smoothers, 3 windmill smoothers. Plus exploratory: difference-of-Gaussians (annulus-with-gap), smooth-then-sharpen, dose-of-smoother variants.
- **Targets**: leefbaarometer (liveability, 2024), RUDIFUN (PBL density/function-mix, 2022 — the only vintage-aligned one), urban taxonomy (morphology, 2025). Vintage mismatches are declared, not hidden.
- **Stratifier**: targets binned SHARP vs SMOOTH by k=3 ring spatial entropy, **frozen before any probe ran**.
- **Probes**: linear DNN probe (zero hidden layers), K=5 random folds, seeds {42, 43, 44} — **648 probe cells**, all reported.
- **Discipline**: freeze-then-run; the λ grid and stability bound came from a declared-exploratory bracketing pilot *before* freeze; the order gate confirms the freeze precedes every result; QAQC independently re-derived all three verdicts with its own implementation and different random seeds — confirmed, zero discrepancies.

## Contents

| Path | What |
|---|---|
| [`PREREGISTRATION.md`](PREREGISTRATION.md) | The frozen contract: hypotheses, SESOIs, stop rules, full cell grid, filled RESULT/VERDICT slots |
| [`report.pdf`](report.pdf) | The write-up |
| [`latex/`](latex/) | Report source + build instructions |
| [`data/`](data/) | `verdicts_2026-07-20.md` (per-cell basis tables), probe-cell summary, stratifier scores + frozen bins, run logs |
| [`figures/`](figures/) | Methodology figures (kernel geometry on real hexagons, effective-kernel profiles, the subtract-the-low-pass story, dose strip, target gallery) |
| [`figures/maps_2026-07-20/`](figures/maps_2026-07-20/) | Map battery: 7 arms × 2 views, each national map paired with a South Holland zoom |

## One governance note (recorded, awaiting a human tick)

Three foundational choices were decided by the human explicitly in chat (operator families, resolution, probing the existing smoothed arms). The rest were decided by the coordinator under an explicit blanket delegation and were **originally mis-tagged** as `[decided-by-human:…:delegated]` — a suffixed form that still puts the human's signature on choices they did not see. An audit caught it; the dossier now uses the agent-decision class `[decided:2026-07-19:under-blanket-delegation]`. The frozen pre-registration still quotes the original form because **a frozen artifact is not edited after freeze** — that is the point of freezing; this note is the correction of record.

Two of those delegated choices deserve a human tick: the λ grid, and β_min = 0.005 / the +0.01 floor — i.e. *what counts as a real effect*. The verdicts are robust to them (β₃ = −0.003 with a CI spanning zero, and every ΔR² negative — no plausible threshold flips the outcome), so this is a precedent exposure rather than a correctness one.

## Limitations

Linear probes only (the one-hidden-layer robustness arm was **not** run); random folds rather than spatial cross-validation; res10 and the Netherlands only; two of three target families carry a vintage mismatch against the 2022 embeddings. Verdicts are scoped accordingly — see the claim-scope paragraph in the pre-registration and the report's discussion.
