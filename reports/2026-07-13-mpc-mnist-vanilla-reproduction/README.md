# Vanilla MPC on MNIST — a faithful reproduction attempt that does not (yet) reproduce

**A backprop-free predictive-coding model, built to the paper's own recipe, reaches 62.5% on MNIST where the paper reports 97.5% — and a drift investigation localises *why*.**

Session `swift-waving-rock`, overnight 2026-07-12 → 13. CPU-only (the GPU belonged to a concurrent VAE session); no geospatial artifacts touched. Implementation commit `5da9fd3`; training run `mpc_st1_full` (wandb `274n6d0z`).

---

## Why this report exists (the one-paragraph overview)

The project has a forward-looking research idea: a **cone-batching** geospatial model that would process the H3 hexagon hierarchy the way the brain processes an image through saccades — using a **backprop-free** learning rule instead of the usual gradient descent. Before betting on that idea in our own hard-to-debug spatial domain, the sane move is to first reproduce the paper it comes from *on the paper's own easy domain* — handwritten-digit classification (MNIST) — where the answer is known (97.5% accuracy) and any shortfall is a defect in **our** build, not a mystery of geography. This report is that reproduction attempt. The short version: **the build runs cleanly and every mechanism behaves as specified, but it classifies digits at 62.5% where the paper reports 97.5%. That gap means tonight does *not* settle the geospatial question — it *narrows* it.** The learning core, as we implemented it with the hyperparameters the paper leaves unspecified, underperforms even on digits. We show the shortfall is a *low, flat representation ceiling* (not a training-time collapse), and we point precisely at the next fix.

---

## Glossary (every term defined once, in plain language)

This report is meant to be read cold — by the user over coffee, by a future session, and eventually as raw material for a paper writeup. So the load-bearing vocabulary is defined up front.

- **MPC (Meta-representational Predictive Coding).** The model from the paper (Ororbia, Friston, Rao; arXiv:2503.21796). "Predictive coding" = a network that learns by trying to *predict* its own internal signals and adjusting to reduce the prediction error. "Meta-representational" = it does this across several parallel views (**streams**) of the same input, each stream predicting the others.
- **Backprop-free / Hebbian.** Standard neural networks learn by **backpropagation** — a global error signal is chained backwards through every layer. MPC does *not* do this. It uses a **Hebbian** rule instead: a local "neurons that fire together, wire together" weight update that needs only the activity of the two units it connects, no global backward pass. This locality is the whole appeal — it is more biologically plausible and parallelisable. Verifying our build is *actually* 100% backprop-free (no hidden autograd) is one of tonight's checks.
- **Stream / glimpse / saccade.** The model never sees the whole digit at once. It takes a **glimpse**: a small patch around a fixation point, decomposed into 6 parallel **streams** (4 fine foveal patches + 1 medium "parafoveal" + 1 coarse "peripheral" — mimicking how the eye samples sharp detail at the centre and blur at the edges). A **saccade** is one such fixation; the model makes **K=10** saccades per image, jumping the fixation around, and accumulates evidence across them.
- **E-step / M-step.** MPC learns in two alternating phases (an "Expectation–Maximisation" loop). The **E-step** ("inference / settling") holds the weights fixed and lets the internal latent activities relax until the prediction error stops shrinking — like letting a physical system settle to equilibrium. The **M-step** ("learning") then holds the settled activities fixed and nudges the *weights* by the Hebbian rule. E-step = think; M-step = learn.
- **Free energy (FE).** The single scalar the model is trying to *minimise* — essentially its total prediction error (how badly each stream predicts the others, plus small regularisation terms). Lower free energy = better internal model. **A healthy training run drives free energy down.** (When it goes *up*, something is wrong — a central finding below.)
- **NWTA (N-Winners-Take-All).** The sparsity mechanism. In each layer only the **N_w = 15** most-active units are allowed to stay on; the rest are zeroed every step. This forces a sparse, competitive code. "NWTA collapse" would mean the same 15 units always win regardless of input (a dead, non-informative code) — we check for this.
- **Receptive field.** What a first-layer filter has learned to detect, visualised by reshaping its weights back into an 8×8 image patch. Healthy filters look like diverse little edge/blob detectors; degenerate ones look identical or blank.
- **Probe / log-linear probe / accuracy.** The learning above is **self-supervised** — it never sees the digit *labels*, only the pixels. To measure whether the learned representation is *useful*, we freeze the model, run every image through it to extract its latent representation, and fit one simple classifier on top: a **log-linear probe** = a single linear layer + softmax (multinomial logistic regression, no hidden layer). Its **test accuracy** (% of held-out digits classified correctly; 10% = random guessing) tells us how much digit-identity information the frozen representation carries. The probe *is* allowed to use backprop — it is a downstream readout, entirely separate from the Hebbian core.
- **Latent dimension 7680.** The probe's input width. For each of K=10 saccades we take the top-layer latent of all 6 streams (6 × 128 = 768) and concatenate across saccades: 10 × 768 = **7680** numbers per image.
- **st1.** The simplest of the paper's four wiring **topologies** — "all-to-all", every stream predicts every other stream. The paper's reported st1 accuracy is our target: **97.50% ± 0.15**.
- **Representation drift.** The failure mode where a model's representation *degrades* as training continues — good early, worse later. We test for it by probing checkpoints from different epochs.

---

## TL;DR — five findings

**The build is mechanically sound; the accuracy is not there; the shortfall is a low flat ceiling, not a collapse — and the most likely cause is the handful of hyperparameters the paper never states.**

1. **Headline: 62.48% test accuracy, versus a 97.50% paper target** ([`figures/08_confusion_matrix.png`](figures/08_confusion_matrix.png)). The spec's own success bands put ≥95% = reproduction, 90–95% = partial, and **<90% = "implementation defect, re-frames the diagnostic."** 62.5% is well inside the defect band. **So tonight does not settle whether backprop-free cone-batching will work on geography — it tells us the core underperforms on the paper's *own* domain, which must be fixed first.**
2. **Free energy *rises* over training** (49.5 → 54.9 → 70.5 → 89.5 → 91.3 across the 5 epochs — [`figures/00_drift_probe_and_fe_trajectory.png`](figures/00_drift_probe_and_fe_trajectory.png), panel B). The model gets *worse* at its own predictive objective after a good first epoch. This is a genuine M-step stability pathology.
3. **But the representation does *not* drift.** Probing epoch 0, 2, and 4 checkpoints on an identical held-out set gives **44.7% → 43.5% → 46.9%** — essentially flat (panel A). Early-stopping would *not* have rescued the result. So the rising free energy is **decoupled** from probe quality; the problem is a low ceiling, not a decay.
4. **No collapse anywhere in the mechanism.** NWTA fires exactly 15 units/layer; the E-step settles monotonically every glimpse; first-layer filters stay unit-norm, zero dead, and near-orthogonal (mean pairwise cosine ≤ 0.14) from epoch 0 to the end. The infrastructure is healthy — the shortfall is *calibration/faithfulness*, not a broken part.
5. **The prime suspects are the paper's unspecified constants.** The paper omits the pool size, hidden width, E-step integration steps, and — most suspect — the **M-step learning rate and weight decay** (our deviations D5, D7, D8, **D9**). We guessed small stable defaults; the rising free energy says our M-step guess is mis-calibrated. **The highest-value next step is to pull the authors' `ngc-learn` reference defaults and re-run.**

**Honesty caveat stated loudly and not softened below:** a 62.5% reproduction is a *negative* result about our current build, not evidence against the MPC idea or against cone-batching. It narrows the question; it does not answer it.

---

## What was built

A paper-faithful implementation of **MPC v1, topology st1**, entirely in `scripts/mpc/mnist/` (commit `5da9fd3`, committed *before* the run per the commit-then-run provenance rule so the sidecar names the exact code that ran):

- `glimpse.py` — the 6-stream foveated glimpse (4 foveal 8×8 + parafoveal + peripheral, average-pooled to 8×8, stream input dim 64).
- `model.py` — the MPC core: the E-step settling loop (Eq. 8), the Hebbian M-step (Eqs. 9–11), NWTA sparsity, fixed-transpose feedback, per-column unit-norm constraint. **100% autograd-free** — the weights are plain tensors, never `requires_grad`; verified tonight (see mechanism checks).
- `train.py` — the self-supervised training loop, wandb logging, checkpointing per epoch, `*.run.yaml` provenance sidecar.
- `probe.py` — the frozen-latent extractor + log-linear probe (the one sanctioned backprop use).
- `viz.py` — a 9-panel mechanistic-diagnostic renderer (built earlier in the session).

**Run**: 60,000 MNIST train images, 5 epochs, batch 100, K=10 saccades, on CPU with 8 threads. 3,000 batches at 3.23 s/batch ≈ 2.7 h wall. Seed 42, deterministic. Status `success`, no NaN, resume-capable.

---

## The headline result

| Metric | This build | Paper target (v1 st1) | Verdict |
|---|---|---|---|
| MNIST test accuracy (log-linear probe, full 60k/10k) | **62.48%** | **97.50% ± 0.15** | 35 pp short |
| Train accuracy (full) | 74.84% | — | modest overfit |
| Latent dim | 7680 | 7680 (K·6·H) | matches spec |

62.48% is **below the spec's <90% "implementation-defect" threshold** (§8). Per the spec's own framing, this "signals an implementation defect and re-frames the whole diagnostic." The confusion matrix ([`figures/08_confusion_matrix.png`](figures/08_confusion_matrix.png)) shows the errors are broadly spread — the model has learned *something* (every digit is recognised well above the 10% chance line; digit 1 is the strongest at 92.3% recall (F1 88.3%)), but confusable pairs (4/9, 3/8, 5/3) bleed heavily. This is the signature of a representation that is *coarsely* right but lacks the fine discriminative structure the paper's version has.

---

## The core investigation: is this drift, or a low ceiling?

The two facts in tension are (i) test accuracy is low, and (ii) free energy — the training objective — *rose* over the run instead of falling. A rising objective usually means the representation is being actively damaged as training proceeds. If so, an early checkpoint should probe *better* than a late one. We tested this directly.

**Method.** We probed three checkpoints — epoch 0, epoch 2, epoch 4 — under **identical limited conditions** (10,000 train / 2,000 test images, same seed) so the numbers are comparable to each other. (These limited numbers are *not* comparable to the 62.48% full-probe headline, which used all 60k/10k; the limited probe overfits its smaller train set, inflating train accuracy and depressing test accuracy. They are an internal apples-to-apples *trajectory*, not an absolute.)

| Checkpoint | TEST acc (2k held-out) | TRAIN acc (10k) | Epoch-mean free energy |
|---|---|---|---|
| epoch 0 | **44.65%** | 96.00% | 49.5 |
| epoch 2 | **43.50%** | 98.88% | 70.5 |
| epoch 4 | **46.85%** | 99.58% | 91.3 |

See [`figures/00_drift_probe_and_fe_trajectory.png`](figures/00_drift_probe_and_fe_trajectory.png).

**Reading it.** Test accuracy is **flat** across training (44.7 → 43.5 → 46.9, a ~3 pp spread with no downward trend — if anything the *last* epoch is marginally best). Meanwhile train accuracy climbs monotonically (96 → 99.6%) and free energy rises steeply. So:

- **The representation is not drifting *down*.** Hypothesis (a) — "epoch-0 latents are much better than epoch-4, so training destroys them; fix the M-step decay" — is **refuted**: epoch 4 ≥ epoch 0. Early-stopping buys nothing.
- **The free-energy rise is real but *decoupled* from probe quality.** The M-step is genuinely anti-optimising its own objective after epoch 0 (free energy nearly doubles), yet the linear-readout accuracy is unmoved. The rich 7680-dim latent stays *linearly decodable* at ~45% even as the generative model degrades — the climbing train accuracy (memorising the 10k subset ever more easily) is consistent with weights rotating into higher-variance, more idiosyncratic directions without gaining held-out signal.

### Verdict: hypothesis (b) — flat-but-low — with a distinct co-finding

**The evidence supports (b): the representation ceiling is low and roughly constant across training (44.7 ≈ 43.5 ≈ 46.9), pointing at capacity/geometry/faithfulness (D5/D7/D8) or a structural implementation gap — not drift.** The one-line reasoning: *epoch-0 and epoch-4 probe within 2 pp of each other, so the shortfall is present from the first epoch and is not caused by training-time decay.*

The **co-finding** (which would read as hypothesis (a) if taken alone) is that free energy rises steeply over training — a real **M-step stability pathology**, most plausibly the D9 learning-rate/decay guess. It matters for the *next run's health* (a diverging objective is not acceptable) but it is **not** the cause of the low accuracy, because early checkpoints — before the divergence — are no better. Both problems likely share one root: the unspecified continuous-time constants (learning rate, decay, integration step) that live in the authors' code, not the paper text.

### Hypothesis (c) — collapse / degeneracy — is refuted

We checked the mechanism directly for the collapse modes the panels might reveal:

- **NWTA**: exactly `[15, 15]` active units per layer ([`figures/09_nwta_sparsity.png`](figures/09_nwta_sparsity.png)) — the sparsity mechanism is exact, no dead-unit or all-active pathology.
- **First-layer filters (receptive fields)**, epoch 0 vs final, computed numerically across all 6 streams and 3 layers: every column stays **unit-norm** (the norm constraint holds), **zero dead columns**, and mean pairwise |cosine| stays low — L0 0.096 → 0.136, L1 0.071 → 0.097, L2 0.071 → 0.073. Filters remain **diverse and near-orthogonal** from start to finish; they do not converge to redundancy or blankness ([`figures/05_receptive_fields.png`](figures/05_receptive_fields.png)).
- **E-step settling**: free energy decreases monotonically within every glimpse ([`figures/02_estep_settling.png`](figures/02_estep_settling.png)) — inference is working exactly as specified.

So there is no collapse. The parts work; the whole underperforms.

---

## What worked mechanically (the infrastructure is sound)

Tonight's run cleanly validated everything *except* the final accuracy — which is why the calibration/faithfulness gap is localised rather than diffuse:

- **E-step monotone settling** — free energy falls every inference iteration, validating the Eq. 8 integration.
- **NWTA exact sparsity** — precisely N_w=15 winners per layer, every step.
- **100% autograd-free core proven** — the Hebbian weights are plain tensors; no gradient ever touches the representation learner. The only backprop is the downstream probe (sanctioned, D11).
- **Numerical stability** — no NaN/inf across 3,000 batches; per-column norm constraint holds; run status `success`.
- **Reproducibility + resume** — seed 42, deterministic, per-epoch checkpoints, provenance sidecar with commit-then-run honoured.
- **Diagnostics** — all 9 mechanistic-viz panels render against the final checkpoint; the drift probes preserved all four probe-result JSONs on disk (nothing overwritten).

The message: **the calibration/faithfulness gap is a *localised* problem sitting on a healthy foundation**, not a rewrite.

---

## Deviations recap, with suspects ranked

The build carries 13 documented deviations from the paper (full table: `specs/mpc_mnist_vanilla.md` §11, private repo). Most are deliberate scoping choices (v1 not v2, MNIST only, CPU-only, st1 topology, fixed-transpose feedback). **Six are AMBIGUOUS — values the paper simply omits and we had to guess.** These are the accuracy suspects, ranked by how strongly tonight's evidence implicates them:

| Rank | Dev | Guessed value | Why suspect | Evidence tonight |
|---|---|---|---|---|
| **1** | **D9** M-step lr / decay | lr Δt/τ_w = 0.02, λ_w = 1e-3 | "learning rate tuned per model" — paper omits; the exact constants live in `ngc-learn` code | **Free energy rises over training → the M-step is mis-calibrated.** Strongest signal. |
| **2** | **D8** E-step integration | T_infer=20, Δt/τ_z=0.1 | paper omits iterations + step size | E-step *does* settle, but 20 iters may under-settle for a 3-layer stack; interacts with D9. |
| **3** | **D7** hidden dim H_ℓ | 128 all layers | paper omits per-layer width | A 128-wide code may cap the ceiling; the flat-but-low signature is consistent with a capacity limit. |
| **4** | **D5** pool size S | 8 (stream input dim 64) | "S×S", value omitted | Sets the input granularity each stream sees; wrong S changes what is learnable. |
| 5 | D6 foveal offset | 2 px | "overlapping 2×2 grid", offset unspecified | Low blast radius; unlikely primary. |
| 6 | D10 probe sub-rep | top-layer only | "concatenate sub-representations", layer unspecified | Alternative (all-layer concat) is a cheap thing to try if the ceiling persists. |

**The single highest-value follow-up: pull the authors' `ngc-learn` / repository defaults for the time constants τ_z, τ_w, Δt, the decay λ_w, the hidden widths H_ℓ, and pool S, replace our guesses, and re-run.** The paper omits these precisely because they are implementation constants that live in code — which is exactly where a reproduction is most likely to diverge. This is a targeted fix, not a redesign.

---

## D13 — retro-ratification ask for the human

One deviation (**D13**) was decided *mid-build*, under delegated overnight autonomy, by the implementer + coordinator — and is flagged as **not yet human-ratified** (spec §11 D13; ops plan AD-6):

> **st1 self-prediction is excluded.** The paper's st1 is "all-to-all *and itself*", but this spec's own CPU cost model counts 6×5 = 30 *ordered* pairs, i.e. streams predicting *other* streams only (v≠q), excluding a stream predicting its own latent. We implemented the exclude-self reading (a `--include-self-pred` flag restores the literal "and itself" version). Rationale: spec-internal consistency with the cost model, plus anti-tautology — a stream predicting its own latent is the degenerate self-referential signal this whole diagnostic exists to avoid.

**The ask:** please ratify (or reject) excluding self-prediction. It is a foundational wiring choice on a novel-method reproduction, and the novel-research hard gate says such choices are the human's to sign, not an agent's. If you want the literal paper reading, the `--include-self-pred` flag re-runs it as a one-line change — and it is worth trying as part of the ngc-learn alignment re-run regardless, since restoring self-prediction slightly changes the free-energy landscape.

---

## v2-ablation deferral note (AD-5)

The current arXiv version of the paper is **v2** (adds epistemic saccade *planning*, LGN units, K=15/60 saccades, 100 epochs, a KNN probe — reported 97.57–98.10%). We deliberately reproduced **v1** (random saccades, log-linear probe) as the simpler foundational baseline. The v2 machinery is **deferred** (ops plan AD-5): it is a set of ablations to layer *on top of* a working v1, and there is no point adding planning to a core that does not yet clear its own baseline. v2 stays parked until the v1 ceiling is understood.

---

## Next steps (ranked)

1. **ngc-learn hyperparameter alignment re-run** *(highest value)* — pull the authors' reference defaults for τ_z, τ_w, Δt, λ_w, H_ℓ, S; replace the D5/D7/D8/D9 guesses; re-run. Directly targets the rising-free-energy pathology (D9) and the low ceiling. Include the `--include-self-pred` variant in the same sweep (D13).
2. **Per-pair cross-error logging in `train.py`** — the trainer currently logs only aggregate cross-stream free energy, not per-(stream v, stream q) error. Adding per-pair logging would let a future run see *which* stream predictions are failing over training (the §9.3 "cross-error over training" panel is currently only available as a checkpoint snapshot, not a trajectory).
3. **Early-stopping / epoch-0 probe follow-up** — tonight showed epoch 0 ≈ epoch 4, so *once D9 is fixed* and free energy actually falls, re-check whether a longer/healthier run finally lifts the ceiling. (Under the current mis-calibration, early-stopping buys nothing — this step is contingent on step 1.)
4. **Capacity sweep (D7/D5)** — if the ceiling survives the ngc-learn alignment, widen H_ℓ (256/360) and re-check S — a targeted test of the capacity hypothesis.
5. **v2 ablations** *(deferred, AD-5)* — planning + LGN + KNN probe, only after v1 reproduces.

---

## Cluster maps and probe maps

The seven figures above are *mechanistic* — they check that each moving part behaves as specified. This section adds seven **analysis** figures (numbered `10`–`16`) that ask a different question: *what does the frozen representation actually look like, and where does its digit signal live?* They are the MNIST analogue of this project's geospatial Stage-3 outputs — **cluster maps** (unsupervised structure of the latents) and **probe maps** (supervised readouts sliced by class, by evidence, by stream, and by fixation position). Every figure below is computed from the **same final checkpoint** that gives the 62.48% headline, via a new builder `scripts/mpc/mnist/report_maps.py` (each figure carries a `*.provenance.yaml` sidecar). Because these run on CPU alongside a concurrent job, they use stated subsamples of the 10,000-image test set rather than the full set; the subsample is named in each figure title and below. **The one-line takeaway: the unsupervised structure is weak (the latents do not clump into digit-shaped clusters on their own), the digit signal is spread roughly evenly across all six streams, and — the sharpest new finding — it is strongly concentrated at the *centre* of the image, which the paper's random-saccade policy under-exploits.**

### Cluster maps — is the representation digit-shaped on its own?

Two vocabulary items first, since they carry the section. **Cluster purity** = if we group the latents into 10 clusters with k-means (an unsupervised algorithm that never sees the labels) and then, *after the fact*, label each cluster by the digit that is most common in it, purity is the fraction of all points that fall in their cluster's majority digit. 100% = every cluster is a pure single digit; 10% = clusters are random with respect to digit. **Adjusted Rand Index (ARI)** = a chance-corrected agreement score between the cluster assignment and the true digit labels: 1.0 = perfect agreement, 0.0 = no better than random, negative = worse than random.

- **[`figures/10_cluster_projection.png`](figures/10_cluster_projection.png) — k-means clusters vs true digits in 2D.** k-means (k=10) is run on the full 7680-dim concat latents of 2,000 test images, then the points are projected to 2D (t-SNE — the UMAP library is not installed in this environment, so the same UMAP→t-SNE→PCA fallback chain as the mechanistic panel 6 lands on t-SNE) and shown twice: left coloured by the k-means cluster each point was assigned, right coloured by its true digit. If the representation were digit-shaped, the two panels would look alike. They do not: the right panel shows only loose, heavily overlapping digit territories, and the left panel's clusters cut across them rather than tracking them.
- **[`figures/11_cluster_digit_composition.png`](figures/11_cluster_digit_composition.png) — cluster×digit heatmap, with purity and ARI.** The row-normalised heatmap (each row a cluster, showing how its members split across the 10 digits) quantifies the picture: **cluster purity = 16.9%** and **ARI = 0.017**. Both are barely above the random-assignment floor. This is the *unsupervised confirmation* of the supervised headline: the 62.48% probe accuracy is not evidence of digit-shaped clumps in the latent geometry — it is a **linear boundary the probe learns** through a cloud that, left to its own structure, does not separate the digits. A healthy 97.5% representation would show high-purity clusters here; ours does not.
- **[`figures/12_cluster_mean_images.png`](figures/12_cluster_mean_images.png) — the literal cluster map.** For each of the 10 k-means clusters, this is the pixel-mean of the 28×28 MNIST images assigned to it — what each latent cluster "looks like" back in image space. Several cluster means are blurry, similar-looking superpositions of multiple digits rather than one crisp prototype, and several clusters share a near-identical dominant digit while others mix two or three. That visual redundancy is exactly what a 16.9% purity predicts: the clusters are not carving the digits apart.

### Probe maps — where does the digit signal live?

- **[`figures/13_probe_per_class_metrics.png`](figures/13_probe_per_class_metrics.png) — per-digit precision / recall / F1.** Computed directly from the headline probe's confusion matrix (so this reuses the exact full-60k/10k 62.48% result, no refit). The per-class F1 spread is wide: **digit 1 is easiest (F1 = 88.3%)** — its distinctive thin vertical stroke is hard to confuse — while **digit 9 is hardest (F1 = 48.9%)**, dragged down by the 4/9/7 confusions. Digits 8 (53.3%) and 5 (53.9%) are the next weakest. The pattern — one very strong digit, a mid-tier majority, and a cluster of confusable digits below 55% — is the per-class signature of a representation that is *coarsely* right but lacks fine discriminative structure, the same reading the headline confusion matrix gave, now resolved digit by digit.
- **[`figures/14_probe_accuracy_vs_saccades.png`](figures/14_probe_accuracy_vs_saccades.png) — evidence accumulation.** The log-linear probe is refit on the first *s* saccades only (the first *s*×768 columns of the concat), for *s* = 1…10, on a 6,000-train / 2,000-test subsample. Accuracy climbs **monotonically from 23.6% at a single saccade to 39.5% at all ten**, with visibly diminishing returns after ~6 saccades. (These numbers sit below the 62.48% headline because the probe is trained on 6k images, not 60k — this figure is an internal *shape*, not an absolute.) The clean monotone rise confirms the multi-saccade accumulation mechanism is doing real work: each extra fixation adds decodable evidence. So the low ceiling is **not** a failure to integrate across saccades.
- **[`figures/15_probe_per_stream_accuracy.png`](figures/15_probe_per_stream_accuracy.png) — which of the six streams carries the signal.** Each stream is probed on *its own* 1,280-dim latents (its 128-wide top layer across all 10 saccades). The accuracies are strikingly flat: **foveal streams 27.0 / 28.2 / 29.0 / 29.2%, parafoveal 30.5%, peripheral 30.3%** — a span of only ~3.5 points, with the two *coarse* streams (parafoveal and peripheral) narrowly on top and the *fine* foveal streams slightly behind. No single stream dominates, so the digit information is distributed; but the fact that the coarse, low-detail streams edge out the sharp foveal ones is a quiet warning sign — in a healthy model the high-resolution foveal pathway should carry the *fine* discriminative detail that separates 4 from 9. Here it does not out-carry the blur, consistent with the coarse-but-not-fine ceiling.
- **[`figures/16_probe_fixation_position_map.png`](figures/16_probe_fixation_position_map.png) — the fixation-position map (the sharpest new finding).** This is the literal spatial probe map: the model is forced to make a *single* fixation at each cell of a 5×5 grid of positions over the 28×28 canvas (fixation centres at pixels 4/9/14/18/23 in x and y), and a probe is fit on that single-fixation latent (2,000 train / 1,000 test per cell). Accuracy is a strong function of *where the eye is forced to look*: **52.3% at the corners rising to 79.6% dead centre** (peak at pixel (14,14)). Two things follow. First, the digit signal is heavily **centre-concentrated** — unsurprising, since MNIST digits are centred, so a central fixation catches the whole digit (especially in the 24×24 peripheral stream) while a corner fixation lands on blank background. Second and more pointed: **a single well-placed (central) fixation at 79.6% beats the ten random-saccade concat at 39.5%** on the same-size training set. The paper's *random* ("involuntary") saccade policy therefore spends most of its ten glimpses on low-information off-centre positions, diluting the very signal it is trying to accumulate. This does not contradict the paper (their healthy model reaches 97.5% *with* random saccades, so a correctly-calibrated core can overcome the dilution), but it surfaces a **concrete, testable lever that is orthogonal to the six unspecified constants**: a centre-biased or planned fixation policy. It is exactly the capability the paper's own **v2** adds — *epistemic saccade planning*, deliberately fixating informative regions — and this map is direct evidence for why that matters. Worth a cheap ablation (centre-biased sampling) alongside the ngc-learn hyperparameter re-run.

### What these maps add to the central finding

The mechanistic panels showed the machine is healthy; these analysis maps show *the shape of its shortfall*. The low-flat-ceiling verdict is now triangulated from three independent directions: (i) **unsupervised** — the latents do not cluster by digit (purity 16.9%, ARI 0.017), so the 62.5% is a learned linear boundary, not intrinsic geometry; (ii) **per-stream** — the signal is evenly spread and the fine foveal pathway fails to out-carry the coarse streams, consistent with missing fine detail; and (iii) **spatial** — the signal is centre-localised and the random saccade policy under-exploits it. Directions (i) and (ii) reinforce the existing suspect ranking (capacity/faithfulness, D5/D7/D8, plus the D9 M-step). Direction (iii) adds a *new* lever the six-constants list did not contain — the fixation policy — cheap to test and complementary to the ngc-learn alignment re-run.

**New-figure index (builder: `scripts/mpc/mnist/report_maps.py`; each with a `*.provenance.yaml` sidecar):** `10_cluster_projection.png`, `11_cluster_digit_composition.png`, `12_cluster_mean_images.png`, `13_probe_per_class_metrics.png`, `14_probe_accuracy_vs_saccades.png`, `15_probe_per_stream_accuracy.png`, `16_probe_fixation_position_map.png` — all in `reports/2026-07-13-mpc-mnist-vanilla-reproduction/figures/`.

---

## Prediction vs actual — what does the model actually predict?

The maps above measure the representation with a downstream probe. This section instead asks the model to *show* what it predicts, in the input's own visual language, so the shortfall can be **seen** rather than only scored. One architectural fact governs what is decodable here, and the figures are honest about it. This is an **encoder-only** MPC: its generative weights run *bottom-up*, `μ^{ℓ,v} = W^{ℓ,v} φ(z^{ℓ-1,v})`, so layer 1 is *predicted from* the clamped input patch `g^v` — **the model never generates the input patch itself, and there is no trained input decoder.** The only route back to input (pixel) space is the **fixed-transpose readout** `(W^{0,v})ᵀ φ(z^{1,v})` — the very same `Wᵀ` operator the E-step already uses for top-down feedback. It is a *linear approximation* from the layer-1 latent, not a learned reconstruction, and these figures label it as such. Builder: `scripts/mpc/mnist/report_pred_viz.py` (each figure carries a `*.provenance.yaml` sidecar).

- **[`figures/18_glimpse_stream_content.png`](figures/18_glimpse_stream_content.png) — what *is* a glimpse (read this one first).** For three test digits (2, 5, 7) and one random ("involuntary") saccade each, the left panel shows the full 28×28 image with the fixation point (red +) and the six stream **footprints** overlaid — the four foveal 8×8 windows (offset ±2px around the fixation), the parafoveal 16×16, and the peripheral 24×24. The right six panels are the actual patches each stream feeds the model: every window is average-pooled to a common 8×8 `g^v`. The figure makes the multi-scale design legible — and honestly captures the random-policy problem the fixation map (fig 16) quantified: digit 2's saccade lands bottom-right, so its foveal streams see **blank** while only the coarse parafoveal/peripheral windows catch the stroke. A glimpse is often mostly background.
- **[`figures/17_prediction_vs_actual.png`](figures/17_prediction_vs_actual.png) — actual patch vs reconstruction, per stream.** Four test digits (0, 3, 6, 8), fixated at the image **centre** (so every stream sees digit content — best case for reconstruction). For each, three rows across the six streams: **Row 1** the actual pooled 8×8 patch `g^v`; **Row 2** the self-reconstruction via the fixed transpose `(W^{0,v})ᵀ φ(z^{1,v})`; **Row 3** the cross-reconstruction — stream *q*'s patch as predicted from the *other* streams' latents through the cross-predictor `R φ(z^{1,v}) + A·a` then the same transpose readout, meaned over the five predictors `v≠q`. **Honest verdict: input-space decoding is only weakly faithful.** The reconstructions recover the coarse light/dark layout of the patch but not the sharp digit stroke; per-stream Pearson correlation between self-reconstruction and the actual patch is only **fov0 +0.16, fov1 +0.26, fov2 +0.34, fov3 +0.39, para +0.34, periph +0.39** — positive everywhere (the latent does carry patch information) but far from a crisp reconstruction. The cross-reconstruction row is visibly close to the self-reconstruction row, meaning the streams' latents are mutually predictive (the cross-predictor works), but they are predicting a *blurry* target. This is the same coarse-but-not-fine ceiling the probe maps found, now shown directly in pixels — with the honest caveat that a linear transpose readout, not a trained decoder, sets the visual quality bar.
- **[`figures/19_prediction_error_over_saccades.png`](figures/19_prediction_error_over_saccades.png) — does error shrink as saccades accumulate?** Per-stream settled intra prediction error `Σ_ℓ ½‖e^{ℓ,v}‖²` (left) and total intra + cross error (right) at settling-end for each of the K=10 **real seeded** saccades, on 500 test images, with three individual per-digit traces. **Honest verdict: error does NOT shrink across saccades — it resets per glimpse.** Batch-mean total intra is essentially flat (k1 = 21.7 → k10 = 23.5, a +3.4% trend), total cross likewise flat (61.9 → 67.8), and the per-digit grey traces jump around wildly by saccade — because each jump tracks *fixation content* (blank vs. digit), not accumulating evidence. This is expected and correct given the architecture: `infer()` re-initialises the latents feed-forward for every glimpse (train.py settles each saccade **independently**, no latent carryover), so there is nothing to accumulate *inside the settling dynamics*. The evidence accumulation that fig 14 showed (23.6% → 39.5% as more saccades are added) happens **downstream, in the concatenation of per-saccade latents the probe reads**, not in the free-energy minimisation. The figure pins down where the multi-saccade benefit does and does not live.

**New-figure index (builder: `scripts/mpc/mnist/report_pred_viz.py`; each with a `*.provenance.yaml` sidecar):** `17_prediction_vs_actual.png`, `18_glimpse_stream_content.png`, `19_prediction_error_over_saccades.png` — all in `reports/2026-07-13-mpc-mnist-vanilla-reproduction/figures/`.

---

## Corrected re-run (D8): before/after

**Overview — what was changed, and why.** The original run's single most-suspect deviation was **D8 under-settling**. Recall the two-phase learning loop (see Glossary): the **E-step** ("settling / inference") holds the weights fixed and lets the internal latents relax toward equilibrium, and the **M-step** ("learning") then nudges the weights with the local Hebbian rule. A foundational fact of predictive coding is that *the Hebbian M-step is only a valid free-energy-descent step at the E-step equilibrium* — if the latents are still on their settling transient when the weights update, the update `ΔW ∝ e·φ(z)` is computed off-equilibrium and does not actually descend the free energy *F*. The original run settled to a depth of only **T·β = 2.0** (T = 20 iterations × β = 0.1 step size), where the closest public reference ancestor — Olshausen sparse coding (SC) in the authors' `ngc-learn` framework — settles to **T·β = 10.0** before its single M-step. Learning off the transient is the cleanest mechanistic account of the original run's central pathology (free energy *rose* over training while the linear-probe readout stayed flat). The corrective run, **human-ratified per the novel-research hard gate** (spec `specs/mpc_mnist_vanilla.md`, D8 section), aligns the settling depth to the ancestor (T = 100, β = 0.1 → T·β = 10.0) and drops three compounding mis-calibrations: it removes the weight decay (`λ_w 1e-3 → 0`, since the unit-norm constraint already regularizes), lowers the M-step learning rate to match SC (`η 0.02 → 0.01`), and flips the unit-norm axis to per-atom (`norm_axis 0 → 1`, SC's per-latent convention). To keep the ~5× more expensive E-step inside an overnight CPU window it trains on the **D4 20,000-image subset** rather than the full 60,000 — a confound that governs how the two runs may and may not be compared (stated loudly under Finding 2).

**Config table — baseline vs corrected.**

| Setting | Baseline `mpc_st1_full` | Corrected `mpc_st1_corr_T100_20k` |
|---|---|---|---|
| E-step iterations `T_infer` | 20 | **100** |
| Step size `β = Δt/τ_z` | 0.1 | 0.1 |
| **Settling depth `T·β`** | **2.0** | **10.0** (matches SC ancestor) |
| Weight decay `λ_w` | 1e-3 | **0** |
| M-step learning rate `η` | 0.02 | **0.01** |
| Unit-norm axis | 0 (per-column) | **1** (per-atom) |
| Training images | 60,000 (full) | **20,000** (D4 subset) |
| Epochs / batch / K saccades / seed | 5 / 100 / 10 / 42 | 5 / 100 / 10 / 42 |
| Self-prediction (D13) | excluded | excluded |
| Run `git_commit` (sidecar) | `7c5977d` | `3dab93d` (code `39ad0bd`) |

**Finding 1 — the correction succeeded at its mechanistic target: free energy now descends.** The headline pathology is fixed. Where the baseline's epoch-mean free energy **rose +84%** across the five epochs (49.5 → 54.9 → 70.5 → 89.5 → 91.3), the corrected run's free energy **falls −15% net** (593.7 → 531.5 → 559.5 → 512.9 → 505.4). The trajectory is no longer monotone-up; it descends with one mid-run bump. **Scale caveat (load-bearing): the two FE columns are *not* comparable in absolute magnitude** — the corrected run changed the settling depth, removed weight decay, and flipped the norm axis, all of which rescale the free-energy units. Only the *sign of the trend* is comparable across runs, and on that axis the result is unambiguous: deeper settling converted a rising objective into a falling one, exactly as the D8-under-settling account predicted. The M-step is now, at least directionally, descending *F*.

**Finding 2 — but it did NOT close the probe gap.** The mechanistic fix did not translate into accuracy. The full-grid log-linear probe (the same 60k-train / 10k-test readout that gave the 62.48% headline) on the corrected final checkpoint reaches **53.51% test / 65.12% train** — *below* the baseline's 62.48%. **Confound (stated loudly): this full-probe comparison is confounded by training-set size** — the baseline model trained on 60,000 images, the corrected model on only 20,000. A model that saw one-third of the data is expected to carry a weaker representation regardless of the settling fix, so **53.51% vs 62.48% is not a clean before/after and must not be read as "the D8 fix made accuracy worse."** The probe *readout* uses the full 60k/10k in both cases; it is the underlying *model* that trained on different amounts of data. The un-confounded comparison — a 60k corrected re-run — has not been done (it is a candidate next step, below). What *is* un-confounded is the within-run trajectory, which is the more revealing finding.

**Finding 3 — the inversion: within the corrected run, probe accuracy DECLINES as free energy falls.** Probing the corrected run's own epoch-0/2/4 checkpoints under identical limited conditions (10k-train / 2k-test, apples-to-apples *within this run*) gives **43.0% → 37.9% → 38.9%** test accuracy — a downward trajectory. This is the striking result: **as the corrected model successfully drives its free energy down, its linear-readout digit structure gets *worse*, not better.** Free-energy descent and probe quality are not merely decoupled here (as they were in the baseline) — they are *anti-correlated*. The near-init latents probe best; settling the model deeper toward its own generative objective actively erodes the features a linear classifier can use. Contrast the baseline, where FE *rose* while the probe stayed *flat* (44.7 → 43.5 → 46.9, no trend). The corrected run trades the baseline's "objective diverges, readout flat" pathology for a new "objective converges, readout degrades" one. Mechanistically this says the free-energy minimum this configuration descends toward is *not* a digit-discriminative representation — deeper faithfulness to the paper's objective, under our remaining (still-inferred) hyperparameters, points the representation *away from* linear separability. This is the sharpest single result of the corrective lane, and it is what most complicates the "just settle deeper" story.

**Finding 4 — the fixation-policy axis: centre-local signal is preserved; random-saccade readout collapses.** Three fixation-policy "arms" were probed on the corrected checkpoint (2k-train / 1k-test per arm, matched subsample), isolating *where the model looks* from *how well it learned*:

| Fixation policy | Corrected test acc | Latent dim | Swift baseline comparator (911eb3a) |
|---|---|---|---|---|
| `centre_single` — one forced central fixation | **73.1%** | 768 | 79.6% |
| `random_k10` — 10 uniform-random saccades (paper's policy) | **21.4%** (train 1.00) | 7680 | 39.5% |
| `centre_biased_k10` — 10 saccades from a σ=3px gaussian around centre | 49.7% (train 1.00) | 7680 | (corners 52.3) |

Two things stand out. First, the **single centred fixation largely survives the correction** (73.1% corrected vs 79.6% baseline) — the centre-localised digit signal that fig 16 identified is still mostly there. Second, the **random-saccade readout collapses** (21.4% corrected vs 39.5% baseline), and its train accuracy is a perfect 1.00 against 21.4% test — **flagged: severe overfit** on a small (2k) train set with a 7680-dim input. The gap between the two arms (73.1% from 768 centred dims vs 21.4% from 7680 random dims) says the **sampling policy dominates the outcome far more than the settling fix does** — a well-placed low-dimensional fixation beats ten poorly-placed high-dimensional ones by 50+ points. This is the corrected-run echo of the baseline's fig-16 finding (a single central fixation at 79.6% beat the ten-random concat at 39.5%), and it points at the same lever: the paper's **v2** *epistemic saccade planning*, which deliberately fixates informative regions rather than sampling uniformly. **Caveats:** the arm accuracies are on a matched but small 2k/1k subsample (internal shapes, not absolutes); `centre_single` reads a **768-dim** latent while the K=10 arms read **7680 dims** (a dimensionality gap that itself favours the low-dim arm on a small train set); and both K=10 arms overfit (train 1.00). Swift's baseline comparators were computed at commit 911eb3a on the pre-correction checkpoint and are **not recomputed** here — they are directional, not a controlled A/B.

**Honest verdict, and what it means for Tuesday's day run.** The corrective run is a **clean mechanistic success and an accuracy non-result.** D8 under-settling was real — fixing it converted the rising free energy into a falling one, confirming the diagnosis at the level it was made. But **D8 alone is refuted as the *sufficient* cause of the 62.5%-vs-97.5% gap**: deeper settling did not lift accuracy, and within the corrected run it *lowered* the probe (the inversion, Finding 3). The corrected-config family — deeper settling, no decay, per-atom norm — is **not a free win**; it fixes the objective's health while leaving (or worsening) the representation's linear-readout quality, which means one or more *additional* mis-calibrations remain between "free energy descends" and "digits become linearly separable." The full-probe number (53.51%) is confounded by the 20k-vs-60k training-set difference and should not anchor the verdict; the within-run inversion and the fixation-policy dominance are the un-confounded signals.

Candidate next levers — **presented as options for the human to choose among, not a recommendation** (foundational choices on a novel-method reproduction are human-gated per the novel-research hard gate):

- **D9 M-step calibration** — the remaining un-aligned suspects: the batch-reduction convention (our **mean-over-batch** vs the reference **sum-over-batch**, an effective ×B learning-rate difference), and the SGD-vs-Adam M-step form. The inversion (Finding 3) is consistent with an M-step that descends *F* toward a non-discriminative minimum, which is what a mis-scaled or wrong-form update would do.
- **The sampling-policy lever** — a centre-biased or planned (v2-style epistemic) saccade policy, motivated by the fixation-arm dominance (Finding 4) and fig 16. Orthogonal to the hyperparameter suspects.
- **A 60k un-confounding re-run** of the corrected config — to remove the 20k-vs-60k confound from the full-probe comparison and get a clean before/after on accuracy at fixed data size.

These are not mutually exclusive, and none is picked here; the corrected run's role was to *test D8 in isolation*, and it did.

**Corrected-run artifact index:** run dir `data/mpc_mnist/runs/mpc_st1_corr_T100_20k/` — `run_info.json` (epoch-mean FE trajectory), `probe_result_final_full.json` (53.51%), `probe_result_epoch{0,2,4}_limited.json` (drift 43.0/37.9/38.9%), `fixation_arms_result.json` (three arms + swift's 911eb3a comparators). Fixation-arm builder: `scripts/one_off/2026-07-14-fixation-policy-arms.py`. wandb run `dlybsj95`, project `mpc-mnist-vanilla`.

---

## Artifact index (all paths absolute-from-repo-root)

- **Run dir**: `data/mpc_mnist/runs/mpc_st1_full/` — checkpoints epoch0–4 + final + latest, `run_info.json`, `mpc_mnist.run.yaml` sidecar.
- **Training log**: `data/mpc_mnist/runs/mpc_st1_full.log` (sibling of the run dir).
- **Probe results** (all preserved, nothing overwritten — data-lifecycle):
  - `probe_result.json` and `probe_result_final_full.json` — the full 60k/10k probe (62.48%).
  - `probe_result_epoch0_limited.json` / `_epoch2_limited.json` / `_epoch4_limited.json` — the drift-probe trajectory (10k/2k).
- **Viz panels** (9): `data/mpc_mnist/runs/mpc_st1_full/viz/2026-07-13/` (each with a `*.provenance.yaml` sidecar).
- **Report figures**: `reports/2026-07-13-mpc-mnist-vanilla-reproduction/figures/` — `00_drift_probe_and_fe_trajectory.png` (drift + FE), `01_free_energy_training.png`, `02_estep_settling.png`, `05_receptive_fields.png`, `06_latent_projection.png`, `08_confusion_matrix.png`, `09_nwta_sparsity.png`.
- **Spec**: `specs/mpc_mnist_vanilla.md` (deviations D1–D13). **Ops plan**: `.claude/plans/2026-07-12-implement-the-vanilla-mrpc-paper-arxiv-2503-21796.ops.md`.
- **wandb**: run `274n6d0z`, project `mpc-mnist-vanilla`.

---

*Backprop-free predictive coding, built to the paper's recipe, running clean on CPU — and 35 points short. The gap is a low flat ceiling on a healthy foundation, and it points at the six constants the paper never printed. That is a good place for a reproduction to be stuck: not "something is broken and we don't know what," but "the mechanism works and the missing numbers live in the authors' code." Next session pulls those numbers.*
