# MPC-MNIST Fidelity Audit — PAPER GROUND TRUTH (W1a)

**Role**: single reference for the join-audit agent (who HAS read `scripts/mpc/mnist/`). This document is written **blind to our implementation** — it describes only what the paper prescribes, so the join-audit's diff is between *this* and *our code*, uncontaminated.

**Source paper**: Ororbia, Friston, Rao — *Meta-Representational Predictive Coding: Biomimetic Self-Supervised Learning*, arXiv:2503.21796. The paper self-names the method **MPC** (single "R", folded into "Meta-representational"); "MRPC" is our internal shorthand.

**Versions read for this document**:
- **v1** (submitted 22 Mar 2025) — the on-disk PDF `deepresearch/predictive coding/META-REPRESENTATIONAL PREDICTIVE CODING biomimetic self-supervised learning.pdf`, 24 pp, extracted verbatim via PyMuPDF this session and read directly (main text pp.1–17, references pp.17–22, **appendix/supplement pp.23–24**). **Our implementation targets v1 semantics.**
- **v2** (submitted 2 Jul 2026) — `https://arxiv.org/html/2503.21796v2`, fetched this session. Mined for the human's "any more items" ask.

**Locator convention**: `v1 p.X` = page of the on-disk PDF; `v1 Eq.N` = equation number in v1; `v2 Eq.N` = equation number in v2 (shifted; see §4). Where v1 and v2 disagree, both are shown. Where the paper is silent, **SILENT** is written literally — gaps are NOT filled with ML convention.

**Prior verbatim eq-check reused (not re-derived)**: `specs/stage2_roster/07_notes_2026-07-03_generative_model_em.md` §1–§3. This document EXTENDS it with details that eq-check did not cover (init/persistence, batch semantics, NWTA timing, self-prediction, per-patch centering, the E-step feedback-index subtlety) and CORRECTS two claims about the appendix (see §4).

---

## 0. Executive orientation

MPC is an **encoder-only** predictive-coding scheme. It never reconstructs pixels. `V=6` parallel encoder streams look at the SAME image through 6 different windows (4 foveal 8×8, 1 parafoveal 16×16, 1 peripheral 24×24), and each stream's job is to **predict the latent activity of the other streams (and, per the paper, itself)** — "meta" = representations predicting representations. Learning is **local Hebbian, no backprop** (Eqs.9–11), with a genuine **E-step latent-settling** inner loop (Eq.8) before each **M-step** weight update. A downstream **log-linear probe** (v1) reads the frozen latents. Target: MNIST **97.50%** (st1) / **97.81%** (st2, best).

The whole scheme is `v1 §2.2, p.4–7`.

---

## 1. Model mechanics, exact

### 1.1 Stream geometry + glimpse encoding — v1 Eq.1 (v2 ~Eq.1–3); v1 §2.1 pp.3–4, appendix p.23

**The 6 streams** (`v1 p.3`, appendix p.23): `V = C + F + P = 4 + 1 + 1 = 6`.
- `C=4` **foveal** views: **8×8** px patches, "four overlapping foveal views, arranged in a 2×2 grid" (`v1 p.3`). Appendix p.23: the 4 foveal views "overlap with one another by **1–2 pixels**", centered on the gaze center-point.
- `F=1` **parafoveal** view: **16×16** px, single patch centered on the gaze center.
- `P=1` **peripheral** view: **24×24** px, single patch centered on the gaze center.
- MNIST is grayscale; the C/F/P split is a **spatial-extent hierarchy, not color channels** (`v1 §2.1`).
- Stream index assignment (`v1 Eq.1, p.4`; appendix Eq.16 p.23): `v=1,2,3,4` → foveal, `v=5` → parafoveal, `v=6` → peripheral.

**Glimpse construction — Eq.1 (v1 p.4) + appendix Eqs.15–16 (p.23)**, VERBATIM:
> "all foveal, parafoveal, and peripheral views (centered around the same center-point) are first **average pooled to always be the shape of S × S pixels**, vectorized (i.e., flattened), and concatenated"
>
> `g(k) = (< g_1(k), g_2(k), …, g_V(k) >)^T ∈ R^{((C+F+P)·(S·S))×1}` , with `g_v(k) = Flat(Pool(p_v))`.

So each stream `v`'s clamped sensory input is `g^v(k)`, an **`S×S`-pooled, flattened** view — NOT the raw 8/16/24-px patch. This is the **extent×resolution non-redundancy**: foveal = narrow-extent fine detail, peripheral = wide-extent coarse gist, all pooled to the same `S²` size.

**Pool size S — STATED (correction to our spec)**. `v1 appendix p.23`, VERBATIM:
> "Foveal views are generally shaped such that `S_C = S` whereas parafoveal views are shaped such that `S_C > S`… in this work, we specifically choose for foveal views **S = S_C = 8 pixels**, for parafoveal views `S_F = 16 pixels`, and for peripheral views `S_P = 24 pixels`."

Therefore **`S = 8`, each stream input dim = `S² = 64`**. (Our spec `mpc_mnist_vanilla.md:48,258` marked S "NOT stated → AMBIGUOUS D5"; that is **inaccurate — v1's appendix states S=8 explicitly**. The chosen value 8 is nonetheless correct.)
- **Internal inconsistency to be aware of**: `v1 Fig.8 caption p.12` describes the layer-`ℓ=1` receptive fields as "**12×12 fields**" (would imply S=12, dim 144). This conflicts with the appendix's S=8. The explicit appendix hyperparameter (S=8) is authoritative; the "12×12" is most likely a display-resolution artifact. Flag only.

**Action vector `a(k)` — exact formula, v1 appendix p.23**, VERBATIM:
> `a(k) = 2([x, y]^T / D) − 1` for a `D×D` pixel image, where x,y are the raw Cartesian coordinates of the glimpse center.

For MNIST `D=28`: `a(k) = 2·[x,y]/28 − 1 ∈ [−1,1]²`. Random policy selects (x,y) each step.

**Per-glimpse `u` (what feeds each stream)**: stream `v` at saccade `k` receives the `64`-dim vector `g^v(k)` clamped to its layer 0, plus the shared 2-vector `a(k)` used ONLY in cross-stream means (Eq.4). Foveal streams (v=1..4) each receive a DIFFERENT one of the 4 overlapping foveal patches (pooled to 8×8); v=5 the parafoveal (16→8 pooled); v=6 the peripheral (24→8 pooled).

### 1.2 Per-stream architecture + the NWTA nonlinearity — v1 Eq.5 (v2 Eq.7); §2.2 p.6

- Each stream has `L=3` hidden layers above the clamped sensory layer `ℓ=0` (`v1 p.9`: "several circuit models (with L = 3 layers)"). Layer 0 has identity nonlinearity and is clamped: `z^{0,v}(t) = g^v(k)` (`v1 Eq.3 note, p.5`).
- **Per-layer hidden dim `H_ℓ` (= `J_ℓ`): SILENT.** v1 gives only the notation `z^ℓ ∈ R^{J_ℓ×1}` (`v1 p.2`) and never states `J_ℓ` for MNIST. **v2 is also SILENT** (confirmed by fetch — "never specifies actual J_ℓ values"). → our **D7 (H_ℓ=128) is unresolved by the paper.**

**NWTA (Eq.5) — EXACT semantics.** VERBATIM (`v1 p.6`):
> `NWTA(z^{ℓ,v})_j = z^{ℓ,v}_j` if `z^{ℓ,v}_j ∈ {N_w largest elements of z^{ℓ,v}}`, else `0`
> "which is the N-winners-take-all (NWTA) function … where only the `N_w` neurons with highest values within the layer/group `z^{ℓ,v}` emit a non-zero firing rate (the rest that lose this cross-neuron competition will emit a zero)."

Operational semantics, point by point (these answer the task's explicit NWTA questions):
- **Hard or soft?** **HARD.** Top-`N_w` by value; strict cutoff; no softmax/temperature. (The paper cites SoftHebb [67] as related but chose the *hard* NWTA; a *soft* WTA is named only as possible future work, `v1 p.6`.)
- **How many winners `N_w`?** **`N_w = 15`** (tuned per-model in range `[10,20]`, "often finding that the value of `N_w=15` yielded good results in general", `v1 p.10`).
- **Applied to `z` (state) or `φ(z)` (rate)?** NWTA **IS** `φ()`. It is applied to the latent state `z^{ℓ,v}` to produce the firing rate `φ(z^{ℓ,v})`. **Winners keep their actual value `z_j` (not binarized to 1); losers are set to 0.** So `φ(z)` is a top-`N_w`-masked copy of `z`.
- **Per E-step iteration or once at settle?** **Per iteration.** `φ()` appears inside the mean predictions (Eq.3 `φ(z^{ℓ-1,v})`, Eq.4 `φ(z^{ℓ,v})`) and its derivative in Eq.8. Since `z` changes every Euler step, `φ(z)` (and thus the winner set) is **recomputed every E-step iteration**. It is not frozen after a warm-up.
- **What happens to non-winners?** Zeroed in the rate `φ(z)` (they contribute nothing to any prediction), but their underlying state `z_j` continues to evolve under Eq.8 (a non-winner can become a winner next iteration).
- **Derivative `∂φ/∂z` for Eq.8**: `v1 Eq.8` needs `∂φ(z^{ℓ,v})/∂z^{ℓ,v}`. For a hard value-preserving top-`N_w` mask, `φ(z)_j = z_j·m_j` with `m_j = 1` iff `j ∈ top-N_w`, so the elementwise derivative is the **binary winner mask `m`** (straight-through). The paper does not spell this out (SILENT on the exact subgradient); the binary-mask reading is the standard one.

**Covariances**: intra `Σ^{ℓ,v} = σI` and inter `Σ_C^{ℓ,v,q} = σI`, scaled identity, `σ>0` (`v1 p.5–6`). With `σ=1` every Gaussian term reduces to `½‖error‖²` and covariances drop out of the errors/updates. Exact `σ` value: **SILENT** (paper only says scaled identity).

### 1.3 Error neurons — v1 Eqs.6/7 (v2 Eqs.8/9); §2.2 p.6

Two error-neuron families per layer `ℓ>0`, VERBATIM (`v1 p.6`):
- **Eq.6 (intra):** `e^{ℓ,v} = z^{ℓ,v}(t) − μ^{ℓ,v}` — stream `v`'s own layer-ℓ activity minus its within-stream prediction.
- **Eq.7 (inter):** `e_C^{ℓ,v,q} = z^{ℓ,q}(t) − μ_C^{ℓ,v,q}` — **stream `q`'s activity MINUS stream `v`'s prediction of it.** Note the target is `q` (the OTHER stream's latent), the predictor is `v`. Sign: target − prediction.

Who predicts whom:
- **Intra (residual, Eq.3):** `μ^{ℓ,v} = W^{ℓ,v}·φ_{ℓ-1}(z^{ℓ-1,v})` — layer ℓ's mean is generated **from the layer BELOW (ℓ-1)** of the SAME stream (`v1 Eq.3, p.5`). ⚠️ The prose on p.4 ("every layer ℓ … minimize the prediction error between its activity and the prediction of this activity by layer ℓ+1") reads top-down, but **the equation is ground truth: the predictor input is `z^{ℓ-1,v}` (below).** This is the classic PC top-down/bottom-up prose ambiguity; use the equation. (Prior `07_notes §7 item 1` flagged the same.)
- **Cross (Eq.4):** `μ_C^{ℓ,v,q} = R^{ℓ,v,q}·φ_ℓ(z^{ℓ,v}) + A^{ℓ,v,q}·a(t)` — stream `v`'s layer-ℓ latent (through `φ`) predicts stream `q`'s layer-ℓ latent, PLUS an action-conditional term. **`a(t)` enters the model ONLY here** (and its update Eq.11); it does NOT enter the intra-mean Eq.3.

### 1.4 E-step (latent settling) — v1 Eq.8 (v2 Eq.10); §2.2 p.6–7

VERBATIM (`v1 Eq.8, p.6`):
> `∂F(Θ)/∂z^ℓ(t) = τ_z · ∂z^{ℓ,v}(t)/∂t = −e^{ℓ,v} + ( E_W^{ℓ,v}·e^{ℓ+1,v} + E_R^{ℓ,v,q}·e_C^{ℓ,v,q} ) ⊙ ∂φ(z^{ℓ,v})/∂z^{ℓ,v}`

Operational semantics: `z^{ℓ,v}` is pushed by (i) its own intra-error `−e^{ℓ,v}` (target-side pull), (ii) top-down feedback of the next layer's intra-error `E_W^{ℓ,v}·e^{ℓ+1,v}`, (iii) cross-stream predictor feedback `E_R^{ℓ,v,q}·e_C^{ℓ,v,q}`, with (ii)+(iii) gated by the NWTA derivative-mask `φ'`.

**Fixed-transpose feedback** (`v1 p.6`, VERBATIM): "set its feedback connection matrices to `E_W^{ℓ,v} = (W^{ℓ,v})^T` and `E_R^{ℓ,v,q} = (R^{ℓ,v,q})^T`; note that these can, alternatively, be learned with Hebbian rules." → fixed transpose is a **stated simplification** (adopt it → no backprop; credit assignment is local via transposes).

**Two subtle ambiguities in Eq.8 the join-audit MUST check (potential fundamental loci):**
1. **Top-down transpose index.** The feedback into layer `ℓ` from the layer-above error `e^{ℓ+1,v}` (dim `H_{ℓ+1}`) requires a matrix `H_{ℓ+1}→H_ℓ`, i.e. **`(W^{ℓ+1,v})^T`**. But the paper writes `E_W^{ℓ,v} = (W^{ℓ,v})^T` (which is `H_ℓ→H_{ℓ-1}`) and multiplies `e^{ℓ+1,v}`. As literally written these dims do not match — the intended matrix is the transpose of the forward weight **connecting ℓ and ℓ+1** (`W^{ℓ+1,v}`). **With flat hidden dims (all `H_ℓ` equal) both matrices are square `[H×H]`, so a wrong choice RUNS SILENTLY but computes the wrong feedback.** This is a real silent-bug surface.
2. **Missing cross-stream target-side term.** In the ensemble free energy `F=Σ_v F_v`, `z^{ℓ,v}` is ALSO a *target* of every stream `q'` that predicts it (error `e_C^{ℓ,q',v} = z^{ℓ,v} − μ_C^{ℓ,q',v}`). A full-ensemble gradient would add a direct `−Σ_{q'} e_C^{ℓ,q',v}` pull. **The literal Eq.8 does NOT include it** — it shows only the predictor-side feedback `E_R^{ℓ,v,q}·e_C^{ℓ,v,q}` (v predicting q), gated by `φ'`. Reading A (each stream descends its own `F_v`; cross-stream coupling arises because each stream's predictor-error references the OTHER stream's current state as target) is self-consistent and is what the equation literally states. Reading B (symmetric, add the target-side pull) is what "descend the ensemble `F`" (the LHS `∂F(Θ)`) would give. **The paper's LHS (`∂F` ensemble) and RHS (only `F_v` terms) are mutually inconsistent; the paper does not resolve which.** The literal/most-supported reading is A. The join-audit should identify which our code does — either is a defensible reading of an underspecified equation, but they produce materially different settling.

**Euler integration** (`v1 p.6–7`): "inference (E-step) is carried out … by applying Equation 8 using Euler integration, for all layers ℓ>0, over a stimulus window length **`E = T/Δt`**." Update: `z ← z + (Δt/τ_z)·[RHS]`. Top layer `L` has no `ℓ+1` term. Layer 0 never updated.
- **`T`, `Δt`, `τ_z`: SILENT in v1 AND v2** ("both are in milliseconds (ms)", no values; `v1 footnote 7 p.7`). → our **D8 (T_infer=20, Δt/τ_z=0.1) is unresolved by the paper.**

**INITIALIZATION + PERSISTENCE of latents — the key question, answered:**
- **The paper is essentially SILENT.** There is **no algorithm box** in v1 or v2. The only statements bearing on it:
  - `v1 Fig.3 caption p.7`, VERBATIM: "all MPC streams are conditioned by the actions … taken by a saccade over the sensory input **as well as possibly their prior expectation (at time t−1)**." (v2 fetch returned the identical sentence.)
  - `v1 Fig.3 label p.7`: "Neural states driven by glimpse decisions (**and possibly prior states**)."
- **Verdict**: whether latents are (a) reset to zero each glimpse, (b) warm-started from the previous saccade's settled state, or (c) persisted across the K saccades of an image is **NOT prescribed**. The hedged "**possibly** their prior expectation (at t−1)" signals warm-start is an *allowed option*, not a mandate. Reset-per-glimpse and warm-start are BOTH consistent with the paper. **SILENT — do not treat either as "the paper's choice."** (This is a place our impl necessarily made an undocumented choice; that choice is not a paper-fidelity violation either way, but it IS a design lever the join-audit should surface.)

### 1.5 M-step (Hebbian learning) — v1 Eqs.9–11 (v2 Eqs.11–13); §2.2 p.6–7

VERBATIM (`v1 p.6`):
- **Eq.9 (intra W):** `τ_w · ∂W^{ℓ,v}/∂t = −λ_w W^{ℓ,v} + e^{ℓ,v} · (φ(z^{ℓ-1,v}))^T`
- **Eq.10 (inter R):** `τ_w · ∂R^{ℓ,v,q}/∂t = −λ_w R^{ℓ,v,q} + e_C^{ℓ,v,q} · (φ(z^{ℓ,v}))^T`
- **Eq.11 (action A):** `τ_w · ∂A^{ℓ,v,q}/∂t = −λ_w A^{ℓ,v,q} + e_C^{ℓ,v,q} · (a^{ℓ,v})^T`

Operational semantics:
- **Local outer-product Hebbian**: `post_error · (pre_activity)^T`. For W: post = `e^{ℓ,v}` (dim `H_ℓ`), pre = `φ(z^{ℓ-1,v})` (the NWTA-activated layer BELOW, dim `H_{ℓ-1}`) → outer product `[H_ℓ × H_{ℓ-1}]` matching `W^{ℓ,v}`. For R: post = `e_C^{ℓ,v,q}`, pre = `φ(z^{ℓ,v})` (this stream's OWN activity). For A: post = `e_C^{ℓ,v,q}`, pre = `a(t)` (the 2-vector action). **Signs: `+ error·pre^T`, `− λ_w·Weight` (decay).** (Sign check: this equals gradient DESCENT of `½‖z−Wφ‖²` on W, `∂F/∂W = −e·φ^T`, so `W ← W + η e·φ^T` reduces error — consistent. The paper's "gradient ascent" phrasing, `v1 p.10`, refers to ascending negative free energy / evidence; the *mechanism* is these local plasticity ODEs, NOT backprop-Adam.)
- **One M-step, after settle**: `v1 p.7`, "then synaptic learning (M-step) is performed by applying, via Euler integration, Equations 9, 10, and 11 for all layers ℓ>0 **once**." Weight update: `Θ ← Θ + (Δt/τ_w)·[−λ_w Θ + outer(post, pre)]`.
- **Unit-column-norm constraint** (`v1 p.7`, VERBATIM): "After the M-step is performed, updated synaptic matrices are constrained to have **unit column Euclidean norms**." Applied EVERY M-step, to `W, R, A`. (Note the paper says **column** norm; the ngc-learn Olshausen-SC ancestor normalizes per-atom/row — an orientation the join-audit should check, but the paper text says column.)
- **`τ_w`, `λ_w`, learning rate `Δt/τ_w`: SILENT in v1 AND v2.** v1 p.10: "The learning rate … was **tuned for each model using the validation subset** … saccade-driven models preferred higher rates." `λ_w` is defined as the std of the zero-mean Gaussian synaptic prior (`v1 Eq.2 term Ω`, p.4) but **no numeric value is given**. → our **D9 (lr=0.02, λ_w=1e-3) is unresolved by the paper.**
- **Batch semantics — summed or averaged? SILENT, but the paper LEANS online/averaged.** The M-step equations are written per-example (single `z`, single `e`). v1 does NOT state the batch reduction. The load-bearing context (`v1 §Limitations p.13`, VERBATIM): "our MPC framework does **not rely on large batches** or batch statistics/normalization …"; and footnote 11 p.13: "**We used a batch size greater than one solely to speed up simulation; MPC is inherently an online learning framework.**" → the intended update is the **online (per-sample) update**; a mini-batch is a speed device. That framing implies the batch update should be **batch-size-invariant** (i.e., **AVERAGE** over the batch, or sum-with-lr/B), so that batch=100 ≈ 100 online steps' worth of direction, NOT 100× the step magnitude. **The exact reduction is SILENT; the "inherently online" framing is the only guidance and it points away from a raw sum-without-rescale.** (This directly bears on the effective learning rate — see §5.)

### 1.6 Cross-stream structure + objective — v1 Eqs.2–4 (v2 Eqs.4–6); §2.2 p.4–6

**Objective (Eq.2), per-stream variational free energy**, VERBATIM structure (`v1 p.4`):
> `F_v(Θ^v) = Σ_{ℓ=1..L} Σ_q N(z^{ℓ,q}(t); μ_C^{ℓ,v,q}, Σ_C^{ℓ,v,q})` [Cross-Representation — the SSL signal]
> `        + Σ_{ℓ=1..L} N(z^{ℓ,v}(t); μ^{ℓ,v}, Σ^{ℓ,v})` [Residual Energy — within-stream hierarchy]
> `        + Ω(Θ^v)` [Synaptic prior — `Σ_{p,i,j} N(Θ^v[p]_ij; 0, λ_w)`, Gaussian weight decay]

Ensemble: `F(Θ) = Σ_{v=1..V} F_v(Θ^v)` (`v1 p.5`). With `Σ=σI`, each `N(·)` term = `½‖error‖²`. **Predictive/Gaussian, explicitly NOT contrastive** (`v1 §contributions p.2`, §1: "obviates the need for common SSL mechanisms such as the production of positive/negative examples as in contrastive learning"). No InfoNCE, no pos/neg pairs.

**Does a stream predict its OWN latent (self-prediction)? — YES, the sum over `q` includes `q=v`.** VERBATIM (`v1 p.4`, describing the free-energy functional): "the v-th stream — which tries to predict the latent representations of any other stream (`q ≠ v`) **as well as possibly itself (`v = q`)**." Whether self is actually wired is set by the **topology** (§below).

**Intra-stream mean (Eq.3) vs cross-stream mean (Eq.4) — how they combine:** the objective sums BOTH the residual (within-stream, generated from the layer below via `W`, Eq.3) AND the cross-representation (across-stream, generated from the same layer's own activity via `R` + action via `A`, Eq.4) for every layer `ℓ>0`. They are additive terms in `F_v`; the E-step (Eq.8) descends their sum, the M-step (Eqs.9–11) updates `W` from the residual error and `R,A` from the cross error.

**The four v1 topologies (Fig.5, §3 p.9–10)** — which `(v→q)` pairs are wired:
- **st1 — all-to-all**: VERBATIM `v1 p.10`: "the simplest — an all-to-all structure (**every column predicts all other columns and themselves**)." → **st1 INCLUDES self-prediction (`v=q`)**: 6×6 = **36 ordered predictor→target pairs**, of which 6 are self (`R^{ℓ,v,v}` = a same-layer lateral self-prediction). MNIST **97.50%**.
- **st2** — chained local one-to-one foveal + all para/peripheral predict all foveal & themselves. Best: **97.81%**.
- **st3** — two-neighbor foveal + para/peripheral → all foveal & themselves. **97.74%**.
- **st4** — all-foveal→all-foveal + para/peripheral → all foveal & themselves. **97.80%**.
- (`v2` replaces these named topologies with a **power-kernel** `γ_{v,q}`, `γ_{v,v}=1` — self-prediction retained with weight 1 — see §4.)

**Prediction is latent-to-latent, same-glimpse, cross-stream** (`v1 §2.2`): stream `v`'s layer-ℓ latent predicts stream `q`'s layer-ℓ latent, at the SAME glimpse. It is NEVER a prediction of raw input, NEVER across glimpses (saccades supply data coverage, not a temporal target), and NEVER a downsampled copy of the predictor's own input (that was the deleted-v0 tautology).

---

## 2. Training protocol (v1 §3, §3.1, p.9–10)

| Item | v1 value | Locator |
|---|---|---|
| Dataset | MNIST 28×28 grayscale, 10 classes, standard split | p.9 |
| **Image preprocessing** | **normalize pixel intensities to [0,1]** | p.9 |
| **Patch preprocessing** | **per-patch mean-centering: "we only center it (i.e., subtract the mean value of patch from the patch group of pixels)"** — applied to EACH extracted view for patch-level models (GPC-fov + all MPC) | **p.9 (VERBATIM)** |
| Epochs | **5** | p.10 |
| Batch size | **100** ("mini-batches of length 100"; batch>1 only for speed, inherently online) | p.10, footnote 11 p.13 |
| K saccades | **10** ("all saccade-driven models … used K = 10 saccades") | p.10 |
| Saccade policy | **random / involuntary**: "a fixed K-length saccade sequence is produced by **randomly jumping across the sensory space**" (Fig.3 p.7); coordinates by "a random policy … at each step" (appendix p.23). **No stated distribution** (uniform vs gaussian vs bounded) beyond "randomly jumping"/"randomly generated saccades" (p.3). → the exact sampling distribution is **SILENT**; "randomly jumping across the sensory space" most naturally reads uniform-over-valid-positions but is not pinned. |
| Optimizer wording | "gradient ascent", lr tuned per-model on validation | p.10 |
| L layers | 3 | p.9 |
| N_w | 15 (range [10,20]) | p.10 |
| Weight init distribution | **SILENT** (v1 and v2 give no init distribution for `W,R,A`) | — |
| Latent init per glimpse | **SILENT** (see §1.4) | — |

**Note the per-patch mean-centering is a genuine, easily-missed preprocessing step** — it is stated in v1 p.9 AND re-stated verbatim in v2. It means each stream's clamped input is **zero-mean per patch**, not raw [0,1]. (Our spec `mpc_mnist_vanilla.md:175` says only "Pixels normalized to [0,1]" and does not mention per-patch centering — flagged in §5.)

---

## 3. Evaluation / probe protocol (v1 §3, §3.1, p.10)

VERBATIM (`v1 p.10`):
> "Once a model has completed its unsupervised/self-supervised phase, we **fix its synaptic connection strengths (disable its plasticity)** and then allow it to process the training data, validation data, and test data **once**, extracting its latent representation for each data sample. If a model iteratively processes one input, we **concatenated the sub-representations it produces across the K-length saccade trajectory**."
>
> "A **log-linear classifier** is fit to the latent codes of the training set (with validation latent codes used for hyperparameter-tuning) and then evaluated on the test-set latent codes."

Point by point:
- **Classifier type (v1):** **log-linear** (= multinomial logistic regression). Trained with its own optimizer (a supervised readout — backprop here is fine; it is NOT part of MPC). Hyperparameters tuned on validation latents. (v2 switches to **KNN** — see §4.)
- **Which representation feeds it:** the **concatenated sub-representations across the K=10 saccade trajectory**. The paper does **not** state which layer(s) constitute the per-saccade "sub-representation" — top layer only vs all-layers concatenated vs which streams. → **SILENT (our D10, top-layer concat, is a defensible-but-unpinned reading).** The concatenation is **across saccades** (K sub-reps stacked), and implicitly across streams (each saccade yields a per-stream latent); it is a CONCAT, **not** an average.
- **Settled after how many iterations:** the same E-step settling as training (T_infer, SILENT value), run once per image at eval.
- **K at eval:** **same K=10 as train** (v1). (v2 uses K=15 train / **60 test** — see §4.)
- **The 97.5% number's exact context:** `v1 Table 1, p.10`, MNIST column, row **MPC-st1 = 97.50 ± 0.15%** (10 trials, log-linear probe). Best row MPC-st2 = 97.81 ± 0.02%. Supervised BP-FNN ref = 98.04%; generative GPC = 91.97%; GPC-fov = 96.83%; JEPA = 95.40%. Sample-efficiency (Fig.7, p.11): MPC-st4 stays within ~2pp down to 100 labeled samples.
- **A SECOND probe exists** (`v1 p.11`): a single-hidden-layer (1024 ReLU) MLP **decoder** retro-fit to the frozen latents, measuring reconstruction MSE (`Dec-MSE` column). This is a **diagnostic external readout, NOT part of MPC training** — do not mistake it for MPC having a decoder. The appendix (p.23–24) also reports a **nonlinear attentive probe** (Adam+dropout) in Table 2 — this is why Table 2's "Attn-ACC" (98.80% for C=4) exceeds Table 1's log-linear numbers; the **headline 97.5%/97.81% are the LOG-LINEAR probe**, which is what v1-fidelity should match.

---

## 4. v1 → v2 diff — the "more items" section (the human's explicit ask)

**Version metadata**: v1 = 22 Mar 2025; v2 = 2 Jul 2026 (arXiv abs page). Equation FORMS for the core MPC machinery are **identical** across versions; only numbering shifts and new mechanisms are added around them.

### (a) What v2 CHANGES (new mechanisms — all OUT of scope for our v1-vanilla build, but named here so the join-audit can confirm none leaked in)

| Aspect | v1 | v2 | v2 locator |
|---|---|---|---|
| Saccade policy | random/involuntary only | adds **MPC-epi** (epistemic: next fixation = **argmax of a "focus map"** = bottom-up saliency + top-down accumulated prediction-error/epistemic demand) and **MPC-stoch** (focus map perturbed with **i.i.d. standard Gumbel noise**, then sampled) | v2 saccade-planning § |
| LGN units | none | **LGN layer at `ℓ=0`** of each stream: **Difference/ratio-of-Gaussians (DoG) spatial filtering + Naka-Rushton shunting contrast normalization / dynamic gain control**, producing ON/OFF channels + a "signed contrast-invariant feature map" | v2 LGN § |
| Topology | named st1–st4 | **power-kernel `γ_{v,q}`**: "`γ_{v,q}` is one for `q=v` and `0 ≤ γ_q < 1` for `q≠v` (determined … based on a power kernel applied to the index coordinates of each stream)" → **self-prediction retained at weight 1**, cross-stream down-weighted by distance | v2 Fig.5 + topology § |
| K saccades | 10 (train=test) | **15 train / 60 test** ("training could make use of smaller budgets, e.g., K=15, whereas test-time processing benefited from larger budgets, e.g., K=60") | v2 training § |
| Epochs / iters | 5 epochs, batch 100 | **up to 100 epochs / 20,000 iterations, batch 200** | v2 training § |
| Probe | log-linear | **KNN** classifier (specific K value **not stated** in the fetched text) | v2 eval §, Table 2 |
| Results (MNIST) | st1 97.50 / st2 97.81 (log-linear) | MPC-epi **98.10 ± 0.10**, MPC-stoch **97.57 ± 0.13** (KNN); GPC-fov 98.04; I-JEPA 90.82 | v2 Table |
| Equation numbers | obj Eq.2, intra-mean Eq.3, cross-mean Eq.4, NWTA Eq.5, errors Eq.6/7, E-step Eq.8, M-step Eq.9–11 | **+2 shift**: obj **Eq.4**, intra-mean Eq.5, cross-mean Eq.6, NWTA **Eq.7**, errors **Eq.8/9**, E-step **Eq.10**, M-step **Eq.11–13** (the two inserted equations are the LGN DoG/Naka-Rushton pair) | v2 (fetch-confirmed) |

### (b) What v2 CLARIFIES that v1 left ambiguous — hunt of v2 text + appendix

**Verdict: v2 clarifies almost NOTHING of the continuous-time / dimensional ambiguities. The two things it DOES confirm are already in v1.** Row-by-row against our AMBIGUOUS D-rows:

| D-row | Our spec's value | Does v2 (or a closer v1 reading) settle it? | Settled value / status |
|---|---|---|---|
| **D5** (pool S) | S=8, dim 64 | **YES — but by v1's APPENDIX (p.23), not v2.** v1 appendix states `S = S_C = 8` explicitly. v2 main text still says "S×S, value not stated." | **S = 8, input dim 64. SETTLED (v1 appendix). Our spec's "not stated" label was inaccurate; the chosen value is correct.** |
| **D6** (foveal offset/overlap) | offset o=2px → 4px overlap | **YES — by v1 appendix (p.23):** foveal 2×2 grid "**overlap … by 1–2 pixels**." | **overlap 1–2 px (offset ≈ 6–7 px). SETTLED (v1 appendix) — differs from our spec's 4px overlap.** |
| **D7** (hidden dim H_ℓ) | 128 flat | **NO.** SILENT in v1 AND v2 (v2 fetch: "never specifies actual J_ℓ values"). | **SILENT.** |
| **D8** (T_infer, Δt/τ_z) | 20 iters, step 0.1 | **NO.** SILENT in v1 AND v2 (both: "T and Δt in ms", no values). | **SILENT.** |
| **D9** (lr Δt/τ_w, λ_w) | lr 0.02, λ_w 1e-3 | **NO.** SILENT in v1 AND v2 ("tuned per model on validation"; λ_w no numeric). | **SILENT.** |
| **D9-batch** (sum vs avg) | mean /B | **NO explicit statement**, but v1 §Limitations + footnote 11 ("inherently online learning framework; batch>1 solely to speed up simulation") **lean toward batch-size-invariant = average / online-equivalent.** | **SILENT on the exact reduction; paper framing points to online/averaged, away from a raw un-rescaled sum.** |
| **D10** (probe sub-rep layer) | top-layer concat | **NO.** SILENT in v1 AND v2 ("extract its latent representation … concatenate sub-representations", no layer named). | **SILENT.** |

**The one big clarification hiding in plain sight (both versions):** **per-patch mean-centering** — v1 p.9 AND v2 both state "we only center it (i.e., subtract the mean value of patch from the … pixels)." This is a preprocessing step our spec's §7 preprocessing line omits. NOT a new-in-v2 item — it was in v1 all along and the spec missed it. (See §5.)

### (c) v2 related-work / discussion reinterpretation of v1's mechanism

- v2 keeps the v1 self-loop **explicitly** via `γ_{v,v}=1` — this is v2 CORROBORATING that self-prediction is core to the model (not an accident of st1's phrasing). Any v1 build that DROPS self-prediction is departing from a feature v2 makes structurally explicit.
- v2's LGN front-end (contrast-invariant ON/OFF) suggests the authors found the **raw-input distribution mattered enough to add explicit normalization** — indirect support that input conditioning (of which v1's per-patch mean-centering is the minimal form) is load-bearing, not incidental.
- Nothing in v2's discussion reinterprets the E-step feedback structure or resolves the Eq.8 ambiguities from §1.4.

---

## 5. Fundamental-suspect checklist — verdicts (PAPER side)

One line each: what the paper ACTUALLY prescribes, with locator, or SILENT. (No claims about our code — the join-audit owns that side.)

| Suspect | Paper's prescription | Locator | Fidelity-risk note |
|---|---|---|---|
| **Latent persistence / init** | **SILENT.** No algorithm box. Only "**possibly** their prior expectation (t−1)" / "possibly prior states" — warm-start is an allowed option, not mandated; reset also consistent. | v1 Fig.3 p.7; v2 same sentence | Not a fidelity violation either way; but it is an undocumented lever. |
| **NWTA semantics** | **HARD** top-`N_w`; **winners keep value `z_j`, losers → 0**; `N_w=15`; applied to state `z` to make rate `φ(z)`; **recomputed every E-step iteration**; derivative = binary winner mask. | v1 Eq.5 + p.6, p.10 | If code binarizes winners to 1, or applies NWTA once/at-settle instead of per-iteration, or uses soft-WTA → infidelity. |
| **Prediction direction** | Intra: `μ^{ℓ,v}=W·φ(z^{ℓ-1,v})` (from layer BELOW). Cross: `μ_C^{ℓ,v,q}=R·φ(z^{ℓ,v})+A·a` (v predicts q's latent). Error sign: **target − prediction** (Eq.6/7). `a(t)` ONLY in cross-mean. | v1 Eq.3/4/6/7 p.5–6 | Prose on p.4 is top-down and misleading; the equation is bottom-up (from ℓ-1). |
| **M-step batching** | **SILENT** on sum-vs-average; equations written per-example; "**inherently an online learning framework; batch>1 solely to speed up simulation**" → intended batch-size-invariant (average / lr÷B), NOT a raw sum. | v1 Eq.9–11 p.6; footnote 11 p.13 | A raw un-rescaled sum over batch=100 inflates the effective step ~100× vs online — a plausible train-instability driver. |
| **Probe representation** | Log-linear (v1) on **frozen** latents, **concatenated across K=10 saccades**; which layer(s) = **SILENT**; concat not average; disable plasticity first. | v1 p.10 | Which layer feeds the probe is unpinned; a wrong choice degrades the readout but is paper-consistent. |
| **Glimpse encoding** | 6 views (4 fov 8×8 + 1 para 16×16 + 1 periph 24×24), each **average-pooled to S×S with S=8** (dim 64), **flattened, concatenated**; foveal 2×2 grid overlapping **1–2 px**; `a(k)=2[x,y]/D−1`. | v1 Eq.1 p.4 + appendix p.23 | S=8 and overlap 1–2px are STATED in the appendix (our spec marked S "ambiguous", chose 8 = right; chose overlap 4px = too much). |
| **Image/patch preprocessing** | Whole image → **[0,1]**; then **EACH extracted patch mean-centered (subtract patch mean)** before pooling. | v1 p.9 (VERBATIM); v2 same | **Our spec omits the per-patch mean-centering.** If code feeds raw [0,1] (all-positive) patches, the input distribution — and thus every NWTA winner set and every prediction — differs fundamentally from the paper. **Top-tier suspect.** |
| **Sign / transpose conventions** | M-step `+error·pre^T − λ_w·W`; unit **column** L2-norm every M-step; E-step fixed transpose `E_W=(W)^T`, `E_R=(R)^T`; **BUT top-down feedback into layer ℓ needs `(W^{ℓ+1})^T`** (paper writes `(W^{ℓ})^T` — dimensionally inconsistent, silent under flat dims); **cross-stream target-side term `−e_C^{ℓ,q,v}` absent from literal Eq.8** (predictor-side only). | v1 Eq.8–11 p.6–7 | Two silent-under-flat-dims loci in Eq.8 (§1.4) are high-blast-radius if mis-wired; column-vs-row norm orientation is a stated-column-vs-SC-ancestor-row question. |
| **Init (weights)** | **SILENT** (no init distribution for W,R,A in v1 or v2). | — | Undocumented; paper-consistent whatever chosen, but affects settling/stability. |
| **Self-prediction (topology)** | st1 = "every column predicts all others **and themselves**"; objective sum over `q` includes `v=q`; v2 makes it explicit `γ_{v,v}=1`. | v1 p.10, p.4; v2 topology | **Paper's st1 INCLUDES self-prediction (36 pairs). Our D13 excludes it (30 pairs).** The objective is structurally different. Suspect. |
| **E-step under-settling (consequence, not a paper claim)** | Paper: E-step settles over `E=T/Δt` (values SILENT) **to equilibrium** BEFORE the M-step; the Hebbian M-step is only a valid free-energy-descent step at the E-step fixed point. | v1 p.6–7 | If settling depth is too shallow, `ΔW` is computed off-equilibrium and need not descend F — this is structural, though driven by the SILENT T/Δt values. (Already our spec's W6b top suspect.) |

---

## Appendix A — exact v1→v2 equation-number crosswalk

| Concept | v1 | v2 |
|---|---|---|
| Glimpse vector | Eq.1 | ~Eq.1–3 (LGN eqs inserted) |
| Objective (VFE) | Eq.2 | Eq.4 |
| Intra-stream mean | Eq.3 | Eq.5 |
| Cross-stream mean | Eq.4 | Eq.6 |
| NWTA | Eq.5 | Eq.7 |
| Intra error | Eq.6 | Eq.8 |
| Inter error | Eq.7 | Eq.9 |
| E-step (latent ODE) | Eq.8 | Eq.10 |
| M-step W | Eq.9 | Eq.11 |
| M-step R | Eq.10 | Eq.12 |
| M-step A | Eq.11 | Eq.13 |
| (GPC objective / GPC E-step / GPC M-step) | Eq.12 / 13 / 14 | shifted +2 |

Uniform **+2** shift from the objective onward; the two inserted equations are the v2 LGN (DoG + Naka-Rushton) front-end. **Core MPC equation FORMS are identical between versions.**

---

*Written blind to `scripts/mpc/mnist/`. All v1 claims trace to the on-disk PDF (PyMuPDF extraction this session); all v2 claims to `arxiv.org/html/2503.21796v2` (WebFetch this session). Where the paper is silent, SILENT is stated — no ML-convention gap-filling.*
