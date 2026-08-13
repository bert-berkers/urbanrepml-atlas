# Vanilla MPC-MNIST — AS-BUILT MECHANICS

**Author**: stage2-fusion-architect (W1b, T2 MPC-fidelity lane, teal-watching-bay) · 2026-07-14
**Scope**: describe what `scripts/mpc/mnist/{model,glimpse,train,probe}.py` ACTUALLY does as a
mechanism — math + words, every claim anchored `file:line`. **No paper is read here; no
paper-fidelity judgment is made.** This document is the code-side input for a separate join-audit
that will diff it against an independently-produced paper description.

**Method discipline** (per `reports/2026-06-14-distant-gap-mpc-implementation-audit/README.md`): describe
what the code DOES — who predicts whom, what updates what, what persists — never what the variable
names or docstrings *claim*. Where a behavior is emergent/implicit (a tensor persists because it is
never reassigned; a geometry choice makes "independent" streams near-duplicates), it is called out
explicitly as a prime fundamental-discrepancy candidate.

**The two runs this describes** (flag sets that ACTUALLY fired, from the sidecars):

| | `mpc_st1_full` (62.48%) | `mpc_st1_corr_T100_20k` (53.51%) |
|---|---|---|
| commit | `7c5977d` | `3dab93d` |
| N images / epochs | 60000 / 5 | 20000 / 5 (`limit_train`) |
| `t_infer` | 20 | 100 |
| `dt_tau_z` | 0.1 | 0.1 |
| `lr` (Δt/τ_w) | 0.02 | 0.01 |
| `weight_decay` λ_w | 1e-3 | 0.0 |
| `norm_axis` | 0 (unit-column) | 1 (unit-row / per-atom) |
| topology / self-pred | st1 / False (30 pairs) | st1 / False (30 pairs) |
| epoch-mean FE | **RISES** 49.5→91.3 | falls 593.7→505.4 |
| probe test acc | 0.6248 | 0.5351 |

Both runs: 6 streams, 3 layers, H=128, S=8 (input dim 64), N_w=15, K=10, batch 100, seed 42, CPU.
FE `total` = intra+cross (prior excluded, see §6). Probe latent dim = K·6·H = 7680 both runs.

---

## 1. Glimpse pipeline (`glimpse.py`)

### 1.1 Image preprocessing
- MNIST loaded via `transforms.ToTensor()` → float32, shape `[1,28,28]`, range **[0,1]** (÷255).
  `train.py:174-177` (`load_mnist_train`), `probe.py:39-45` (`load_mnist`). **No binarization, no
  mean/std normalization, no per-image or per-patch normalization anywhere.** Pixel values enter the
  streams as raw [0,1] intensities.
- Zero-pad each side by `PAD=12`: `[B,1,28,28] → [B,1,52,52]`, `mode="constant", value=0.0`
  (`glimpse.py:26,34-36`). 12 = half the largest (24×24) patch, so any fixation in [0,27] with the
  foveal ±2 offset yields fully in-bounds patches (verified: worst case fixation 27 + offset 2 +
  half 12 = 41 padded ≤ 51).

### 1.2 Saccade position sampling
- `sample_fixations` (`glimpse.py:39-49`): `fx, fy = torch.randint(0, 28, (B,), generator=gen)` —
  **integer** coordinates uniform over the FULL `[0,27]×[0,27]` grid, one independent draw per image.
- Generator is **seeded** and **persistent across the whole run**: `sacc_gen =
  Generator().manual_seed(seed + 12345)` created once (`train.py:283`), advanced by every
  `sample_fixations` call. So a given image gets **different** fixations every epoch (the stream is
  not reset per image/batch/epoch). Probe uses a fresh `Generator().manual_seed(seed)` per
  `extract_latents` call (`probe.py:57`).
- **Statistics note**: MNIST digits are centered on a large black border, but fixations are uniform
  over the whole 28×28 → a large fraction of glimpses (especially the wide peripheral view) are
  centered on near-black border/pad and carry ~zero signal. This is emergent from uniform sampling
  over centered data, prescribed by no line.

### 1.3 Patch extraction (per stream)
- `extract_patch` (`glimpse.py:52-72`): patch centered at `(cx,cy)` in original coords; `half =
  size//2`; padded center `px=cx+PAD`; rows/cols span `[center-half, center-half+size-1]`.
  **Asymmetric for even sizes**: 8×8 spans `[c-4, c+3]` (4 left / 3 right of center); 16×16 spans
  `[c-8,c+7]`; 24×24 `[c-12,c+11]`. Vectorized over batch by advanced indexing (per-image centers).
- `build_glimpse` (`glimpse.py:89-125`), 6 patches:
  - **4 foveal** 8×8 at offsets `(-o,-o),(-o,o),(o,-o),(o,o)` with `o=fov_offset=2`
    (`glimpse.py:109-111`). Each foveal patch center = `(fx±2, fy±2)`, each spanning 8px → the 4
    patches tile a ~12×12 region around the fixation, **overlapping ~75%** (a shift of 4 between
    adjacent 8×8 windows).
  - **1 parafoveal** 16×16 centered (`glimpse.py:114-115`).
  - **1 peripheral** 24×24 centered (`glimpse.py:118-119`).

### 1.4 Pooling → common S×S
- `pool_to_s` (`glimpse.py:75-86`): `avg_pool2d(kernel = p//s)`, requires `p % s == 0`. With S=8:
  - foveal 8→8: `p==s` → **identity, returned unchanged** (`glimpse.py:81-82`) — foveal inputs are
    **RAW pixels**.
  - parafoveal 16→8: `avg_pool2d(k=2)` — each output = mean of a 2×2 block.
  - peripheral 24→8: `avg_pool2d(k=3)` — each output = mean of a 3×3 block.
- Each pooled view `.reshape(B, -1)` → **per-stream input vector dim = S² = 64** (`glimpse.py:111,
  115, 119`).

**Input-statistics consequence (IMPLICIT, load-bearing).** Average-pooling reduces variance. The 4
foveal streams keep raw-pixel variance; parafoveal is 2×2-averaged (≈¼ variance for uncorrelated
pixels), peripheral 3×3-averaged (≈1/9). So the 6 streams do **not** have matched input statistics,
and — combined with §1.3's ±2 overlap — the "6 parallel streams" are in fact **4 near-duplicate
raw-pixel windows + 2 blurred wide views**, not 6 independent content channels. NWTA and the
cross-stream prediction (§3, §5) operate downstream of this asymmetry. Prescribed by no spec line as
"these should be near-identical"; it is an emergent property of the S=8 / o=2 geometry.

### 1.5 Action vector
- `action = stack([2·fx/27 − 1, 2·fy/27 − 1])` → `[B,2]` in **[−1,1]²** (`glimpse.py:121-124`). It is
  the fixation coordinate; it enters ONLY the cross-stream prediction (§5) and its Hebbian update
  (§6), never the intra-stream path.

**Equation-as-implemented (glimpse):**
```
g^v(k) = flatten( avgpool_{p_v/8}( patch_{p_v}( pad(img), center_v(fx,fy) ) ) ) ∈ R^64,  v=1..6
   p_1..4 = 8 (identity pool), center_v = (fx±2, fy±2);  p_5 = 16 (k=2);  p_6 = 24 (k=3)
a(k) = (2·fx/27 − 1, 2·fy/27 − 1) ∈ [−1,1]^2
```

---

## 2. Latent state lifecycle (CRITICAL)

**The single most important mechanistic fact: latents carry NO state across saccades. Only the
weights (W,R,A) persist.**

- **Creation / init**: every call to `model.infer` allocates a **fresh** `z = [[None]*V for _ in
  range(L)]` (`model.py:230`) and initializes it by a **feedforward pass through the current
  weights**: `z[l][v] = below[v] @ W[l][v].t()`, where `below` starts as the glimpse `phi0` and
  becomes `nwta(z[l][v])` for the next layer (`model.py:229-235`). There is no persistent `z`
  buffer, no carried hidden state, no zero-init-then-accumulate.
- **Across E-step iterations (within one glimpse)**: `z` IS mutated in place across the `t_infer`
  Euler steps (`model.py:241-245`, `_euler_latent_step` writes `z[l][v] = z[l][v] + step*dz` at
  `:320`). This is the only place `z` persists.
- **Across saccades within one image**: **NOT persisted.** The training loop calls `model.infer`
  fresh for each `k in range(n_saccades)` (`train.py:310-314`); the probe does the same
  (`probe.py:65-68`). Each of the K=10 saccades is an **independent settle from feedforward init** —
  the 10 glimpses of an image share nothing but the (slowly-updating) weights.
- **Across images within a batch**: the batch dimension `B` is vectorized; each image is its own row
  `[B,H]`. No per-image persistence beyond the batch tensor's lifetime.
- **Across batches / epochs**: **NOT persisted.** Only `self.W`, `self.R`, `self.A` (updated by
  `learn`) survive. The saccade generator state also persists (§1.2).
- **Error-neuron state** (`e`, `eC`, `mask`, `phi`): recomputed from scratch every
  `_compute_errors` call (`model.py:259-288`), returned in the `state` dict, never persisted beyond
  the current glimpse's settle + M-step. No error-neuron buffer accumulates.

**Emergent consequence**: the "saccade loop" (spec mechanism #1) and "action conditioning"
(mechanism #2) create **no temporal memory**. The latents are i.i.d. across fixations. The only
integrator in the whole system is the weight matrices. The probe (§8) then concatenates 10
**independent random-fixation snapshots** of the top latent. No spec line prescribes latent
persistence across saccades — but that a fixation "trajectory" carries no state IS a candidate
fundamental discrepancy (a trajectory implies memory; here it is a bag of independent samples).

---

## 3. E-step as implemented (`model.infer`, `_compute_errors`, `_euler_latent_step`)

**Order of operations per glimpse** (`model.py:227-257`):
1. `phi0 = glimpses` (layer-0 identity nonlinearity; glimpses are NOT NWTA'd — `model.py:227`).
2. Feedforward init of `z` (`model.py:232-235`).
3. Loop `t_infer` times (`model.py:241-245`): **(a)** `_compute_errors` → `{phi, mask, e, eC}`;
   **(b)** optional FE trace append; **(c)** `_euler_latent_step` mutates `z`.
4. **Final** `_compute_errors` on the settled `z` (`model.py:248`) — these errors/activities are
   what the M-step consumes.

So per iteration the order is **error computation → Euler state update**. NWTA is computed inside
`_compute_errors` (to build `phi` and `mask`), recomputed every iteration.

**`_compute_errors`** (`model.py:259-288`):
- `phi[l][v] = nwta(z[l][v], N_w)` and `mask[l][v] = nwta_mask(z[l][v], N_w)` for all l=0..L-1, all v
  (`model.py:270-271`).
- intra: `below = phi0[v] if l==0 else phi[l-1][v]`; `mu = below @ W[l][v].t()`; `e[l][v] = z[l][v]
  − mu` (`model.py:275-279`). So the prediction of latent layer ℓ is made **from the layer below**
  (encoder/forward-generative direction): μ^{ℓ,v} = W^{ℓ,v} φ(z^{ℓ-1,v}).
- cross: `mu_c = phi[l][v] @ R[l][(v,q)].t() + action @ A[l][(v,q)].t()`; `eC[l][(v,q)] = z[l][q] −
  mu_c` (`model.py:284-286`), for every ordered pair `(v,q)` in `self.pairs`.

**`_euler_latent_step`** (`model.py:290-320`), the settling update — for every l, v:
- top-down feedback: `fb = e[l+1][v] @ W[l+1][v]` if `l < L-1` else `zeros`
  (`model.py:306-309`). This is `(W^{ℓ+1,v})ᵀ e^{ℓ+1,v}` (fixed-transpose, D12). Top layer: fb=0.
- cross feedback, **v-as-PREDICTOR only**: `cross = Σ_q e_C[l][(v,q)] @ R[l][(v,q)]` over pairs whose
  first element == v (`model.py:313-316`, the `if vv == v:` filter). This is `Σ_q (R^{ℓ,v,q})ᵀ
  e_C^{ℓ,v,q}`.
- update: `dz = −e[l][v] + (fb + cross) * mask[l][v]`; `z[l][v] += step·dz` (`model.py:319-320`),
  `step = dt_tau_z` (0.1 both runs).

**Equation-as-implemented (E-step):**
```
z^{ℓ,v} ← z^{ℓ,v} + step · [ −e^{ℓ,v} + ( (W^{ℓ+1,v})ᵀ e^{ℓ+1,v}  +  Σ_q (R^{ℓ,v,q})ᵀ e_C^{ℓ,v,q} ) ⊙ m^{ℓ,v} ]
   with  e^{ℓ,v}  = z^{ℓ,v} − W^{ℓ,v} φ(z^{ℓ-1,v})
         e_C^{ℓ,v,q} = z^{ℓ,q} − ( R^{ℓ,v,q} φ(z^{ℓ,v}) + A^{ℓ,v,q} a )
         m^{ℓ,v} = binary top-N_w mask of z^{ℓ,v} (φ' for NWTA)
   top layer (ℓ=L): drop the (W^{ℓ+1})ᵀ term.  layer 0: never updated (clamped glimpse).
```

**Key mechanistic facts:**
- **No z clipping, no z normalization, no leak beyond −e.** `z` is unbounded. The `−e` term is the
  only "pull toward prediction"; there is no separate `−z` decay (consistent with leak=0). Step size
  fixed 0.1. Iterations: 20 (full) / 100 (corr).
- **NWTA gates the feedback but NOT `−e`.** `z` itself is never masked/zeroed; only `φ(z)=nwta(z)` is
  sparse and only `φ`/`mask` are used in predictions/gating. The settled `z` read by the probe (§8)
  is the **dense** latent.
- **The cross-feedback is predictor-role ONLY** (`if vv==v`, `model.py:314-316`). A stream's latent
  `z^{ℓ,v}` receives a signal to be a better *predictor* of the other streams (Σ_q (R^{v,q})ᵀ e_C),
  but receives **no** direct "v-as-target" term `Σ_p e_C^{p,v}` pulling it toward what other streams
  predict it to be. The v-as-target term is absent from the code AND from the transcribed Eq.8 (spec
  §4) — i.e. code is faithful to spec-Eq.8, but the omission is a prime **paper**-fidelity candidate
  for the join-audit (a symmetric cross-consistency objective would include both roles).
- **Cross target is the raw, dense, unbounded `z^{ℓ,q}`** (`model.py:286`), predicted from the
  sparse `φ(z^{ℓ,v})`. Since `z` has no magnitude bound, `‖e_C‖²` can grow as latent magnitudes grow
  — a mechanistic route to the FE-rise seen in `mpc_st1_full` (49.5→91.3).

---

## 4. NWTA implementation (`model.py:116-133`)

- `nwta_mask(z, n_w)` (`:116-128`): `idx = z.topk(n_w, dim=-1).indices`; `mask =
  zeros_like(z).scatter_(-1, idx, 1.0)`. Binary `[B,H]` mask, **exactly N_w ones per row**.
  Short-circuit: if `n_w >= H` return all-ones (`:123-124`) — never triggers (15 < 128).
- `nwta(z, n_w) = z * nwta_mask(z, n_w)` (`:131-133`) — keeps the N_w largest, zeros the rest.
- **Ties**: broken by `topk`'s index order (deterministic; lower index wins) → always exactly N_w
  kept.
- **Gradient-free**: hand-built by `topk`+`scatter`; the mask IS used as `φ'` in the E-step
  (`model.py:271, 319`).
- **Applied per-row (per-image), per-layer, per-stream** — the loops at `model.py:270-271` cover all
  l=0..L-1 and all v; each `[B,H]` tensor is masked independently per row. N_w=15 of H=128 fire.

---

## 5. Prediction wiring (st1, the config actually run)

`build_topology_pairs` (`model.py:88-109`), `topology="st1"`, `include_self_pred=False` (both runs;
sidecars confirm `pairs: 30`):

```
pairs = [(v,q) for v in 0..5 for q in 0..5 if v != q]   →  30 ordered pairs  (self-pairs EXCLUDED, D13)
```

- **Every stream predicts every OTHER stream; self-prediction (v==q) is excluded** (`model.py:99-104`,
  the `(v != q) or cfg.include_self_pred` guard). D13 flag state in BOTH runs: `include_self_pred =
  False`. 30 pairs.
- Prediction is **same-layer, cross-stream, same-glimpse**: predictor v's activity `φ(z^{ℓ,v})` (plus
  action) predicts target q's raw latent `z^{ℓ,q}` at the **same** layer ℓ (`model.py:284-286`). It
  happens at **all 3 hidden layers** (`for l in range(L)`, `:283`).
- **Top-down within-stream** (intra): `μ^{ℓ,v} = W^{ℓ,v} φ(z^{ℓ-1,v})` — layer ℓ predicted from the
  layer below (`model.py:277-278`). The E-step feedback across layers uses the **fixed transpose**
  `(W^{ℓ+1,v})ᵀ` (`model.py:307`, D12) — no separately-learned feedback weights.
- **Cross feedback transpose**: `(R^{ℓ,v,q})ᵀ` via `eC @ R` (`model.py:316`, D12).
- **Signs**: both errors are activity − prediction: `e = z − μ` (`:279`), `e_C = z_q − μ_c` (`:286`).
- **Action** enters every cross-prediction: `+ A^{ℓ,v,q} a` (`model.py:285`). Since a single fixation
  gives one `a` shared by all 6 streams, `A@a` is a per-pair, position-driven additive bias on the
  cross-prediction.

**Prediction pair count actually wired**: 30 pairs × 3 layers = **90 R matrices [128,128] + 90 A
matrices [128,2]**, plus 6 streams × 3 layers = 18 W matrices ([128,64] at l=0, [128,128] at l≥1).

---

## 6. M-step as implemented (`model.learn`, `model.py:324-354`)

Fires **once per saccade** (`train.py:318`, inside the K-loop) → 10 M-steps per batch. Uses the
**settled-state** errors/activities from `infer`'s final recompute (`model.py:248`).

`B = action.shape[0]` (batch size, `model.py:335`). For each layer l:

**Eq.9 — intra forward weights** (`model.py:339-343`), per stream v:
```
pre  = phi0[v]  if l==0  else phi[l-1][v]            # NWTA activity below (glimpse at l=0)
hebb = e[l][v].t() @ pre / B                          # [H, in]  = MEAN outer product over batch
W[l][v] = W[l][v] + lr · ( −λ_w · W[l][v] + hebb )
W[l][v] = W[l][v] / (‖W[l][v]‖_axis + eps)            # renormalize, axis = norm_axis
```

**Eq.10/11 — cross + action weights** (`model.py:345-354`), per pair (v,q):
```
post   = eC[l][(v,q)]                                 # [B,H], dim of z^q
hebb_r = post.t() @ phi[l][v] / B                     # [H,H]
R[l][(v,q)] = R + lr · ( −λ_w · R + hebb_r );   R = R / (‖R‖_axis + eps)
hebb_a = post.t() @ action / B                        # [H,2]
A[l][(v,q)] = A + lr · ( −λ_w · A + hebb_a );   A = A / (‖A‖_axis + eps)
```

**The D9 `×B / /B` question — RESOLVED: the code uses `/B` (batch MEAN).** `e.t() @ pre` sums the
per-sample outer products; the explicit `/ B` (`model.py:341, 348, 352`) converts sum→mean. (Spec §5
also prescribes `/B` mean; the W6b addendum flags that the ngc-learn SC ancestor uses SUM, so the
code is spec-faithful but the *spec's* mean vs the ancestor's sum is a separate reference question —
not a code defect.)

**Learning-rate application**: `Θ += lr·(−λ_w·Θ + hebb)`, so the update is `ΔΘ = lr·hebb −
lr·λ_w·Θ`. lr = 0.02 (full) / 0.01 (corr). Weight decay λ_w = 1e-3 (full) / 0.0 (corr) — and decay
IS scaled by lr (matches spec §5 Euler form).

**Weight normalization (`_col_norm`, `model.py:356-360`)**: `w / (w.norm(dim=axis, keepdim=True) +
eps)`, applied to W, R, AND A after every update, every M-step. `norm_axis` semantics:
- `axis=0` (full run): normalize along dim 0 → each **column** unit L2 (a column of `[H,in]` runs
  over the H output units).
- `axis=1` (corr run): normalize along dim 1 → each **row** unit L2 (SC per-atom: each of the H rows
  of `[H,in]` is a unit vector). Applied identically to A `[H,2]` (axis=1 makes each row a unit
  2-vector — a very different geometry than column-norm).
- Same axis is used at init (`model.py:173`) and at every M-step.
- **Order: update → normalize** (`model.py:342-343, 349-350, 353-354`).

**Emergent consequence (load-bearing)**: because every weight is renormalized to unit norm along
`axis` after each small (lr=0.01–0.02) increment, the **raw magnitude of the Hebbian outer product
is almost entirely discarded** — the weights live on the unit sphere and learning is a slow rotation
on that sphere. This means the effective learning signal is the *direction* of `hebb` relative to
the current weight, scaled by lr, then reprojected. It also means weight decay λ_w has little effect
(the renorm dominates magnitude), consistent with the corr run setting λ_w=0.

**Signs / transposes**: post-synaptic = error (`e` for W, `e_C` for R/A); pre-synaptic = activity
(`φ(z^{below})` for W, `φ(z^v)` for R, `a` for A). Outer product = `post.t() @ pre` → `[dim_post,
dim_pre]`. All signs positive on `hebb`, negative on the decay term.

---

## 7. Training loop (`train.py:229-385`)

- **Epochs**: 5 (`cfg.epochs`, `:294`). **Batching**: manual index slicing, NOT a `DataLoader`. All
  images pre-stacked into one `[N,1,28,28]` tensor in memory (`load_mnist_train`, `:246`); N=60000
  (full) / 20000 (corr, `--limit-train`).
- **Shuffle**: per-epoch deterministic permutation `perm =
  randperm(n, generator=Generator().manual_seed(seed+epoch))` (`:296`). `batch = imgs[perm[bi·bs :
  (bi+1)·bs]]` (`:303-304`).
- **drop_last**: `batches_per_epoch = ceil(N/bs)` (`:285`) — last partial batch is kept and handled
  via `B=batch.shape[0]`. Both runs divide evenly (600 / 200 batches), so no partial batch fired.
- **Per-batch sequence** (`:299-328`): for each batch, loop `k in range(K=10)`: `sample_fixations →
  build_glimpse → infer (E-step settle) → free_energy (logged) → learn (M-step)`. So the order is
  **one E-step then one M-step, per saccade, K times** — NOT "E over all saccades then one M." 10
  independent (settle, learn) events per batch.
- **State carried between batches**: only the weights (in `model`) and the saccade generator
  (`sacc_gen`, continuously advanced). No latent/error state.
- **FE logged**: per batch, mean over the K saccades (`batch_fe.intra/cross` accumulated over
  saccades, `/k` at `:325`); `fe.prior` snapshotted after the last learn (`:321`). Note `fe =
  model.free_energy(state)` is taken on the settled state **before** `model.learn` (`:317-318`).
- **`FreeEnergy.total = intra + cross`** (`model.py:147-150`) — the synaptic **prior Ω is EXCLUDED**
  from the logged/settled "total" (tracked separately as `fe_prior`). So the "FE total" curve is the
  cross+residual energy, not the full Eq.2 objective incl. the weight-prior term.
- **Device/dtype**: CPU, float32, `set_num_threads(8)` (`:233`).
- **Seeding** (`:234, 283, 296, 167, 189`): `seed_everything(42)` = `random.seed / np.random.seed /
  torch.manual_seed / use_deterministic_algorithms(True)`. Model-init gen = `seed` (42,
  `model.py:167`); saccade gen = `seed+12345`; per-epoch shuffle gen = `seed+epoch`; self-check gen =
  `seed+99999`.
- **wandb metrics** (`:342-345, 363-364`): `fe_total, fe_intra, fe_cross, fe_prior, batch_wall_s` per
  `log_every` batch; `epoch_mean_fe` per epoch. A permanent startup self-check (`:185-222`) asserts
  (a) zero autograd edges over ~324 weight/latent/error tensors, and (b) within-glimpse FE decreases
  (`feN <= fe0 + 1e-6`).
- **Autograd-free proof**: entire train step under `torch.no_grad()` (`:293`); weights are plain
  tensors (`model.py:178-189`), `_assert_no_grad` at init/load (`:191, 203-207`).

---

## 8. Probe as implemented (`probe.py`)

- **Representation extracted** (`extract_latents`, `:48-73`): freeze W,R,A (never call `learn`), run
  each image once through K=10 saccades. Per saccade, `infer` settles with the **same `cfg.t_infer`
  as training** (20 full / 100 corr), then take the **settled TOP-layer latent** `state["z"][L-1][v]`
  — the **raw dense** z (NOT NWTA'd) — concatenated across the 6 streams → `[B, 6·H=768]`
  (`:69-70`). Concat across K saccades → `[B, K·6·H = 7680]` (`:72`).
- **Concatenation order**: stream-minor within a saccade (`cat` over 6 streams, `:70`), saccade-major
  across the trajectory (`cat` over K, `:72`): `[s0v0…s0v5, s1v0…s1v5, …]`.
- **Saccade policy at probe time**: **uniform RANDOM**, fresh `Generator().manual_seed(seed=42)` per
  `extract_latents` call (`:57`) — same random policy as training, NOT centre/fixed-grid.
- **Classifier** (`train_log_linear`, `:76-111`): standardize features on **train** mean/std
  (`:86-91`, test uses train stats — no leakage); `clf = nn.Linear(7680, 10)`; `Adam(lr=1e-2,
  weight_decay=1e-4)` (`:94`); `CrossEntropyLoss`; **full-batch** gradient descent for
  `epochs=probe_epochs` (default 200, `:122`; the fixation-arms run confirms epochs=200, lr=0.01).
  This is the ONE sanctioned autograd path (D11) — it never touches MPC weights.
- **Train/test split**: standard MNIST 60000 train / 10000 test (`:136-137`). Note: even the corr run
  (model trained on 20k) fits the probe head on the FULL 60k train latents and evaluates on 10k test.
- **How 62.48% was produced**: `mpc_st1_full/checkpoint_final.pt` (t_infer=20, lr=0.02, λ=1e-3,
  axis=0, 60k/5ep) → probe (200 ep, lr 1e-2) → **test 0.6248 / train 0.7484** (`probe_result.json`;
  latent 7680).
- **How 53.51% was produced**: `mpc_st1_corr_T100_20k/checkpoint_final.pt` (t_infer=100, lr=0.01,
  λ=0, axis=1, 20k/5ep) → probe → **test 0.5351 / train 0.6512** (`probe_result_final_full.json`).
- **Mechanistically decisive side-result** (`fixation_arms_result.json`, corr checkpoint, 2000-train
  / 1000-test subsample): **`centre_single`** (ONE forced fixation at pixel (14,14), 768-dim single
  top latent) → **test 0.731**; **`random_k10`** (the standard probe policy) → **test 0.214**
  (train 1.0 — overfits the 7680-dim rep on 2k samples); `centre_biased_k10` → 0.497. A single
  centered glimpse dramatically **out-classifies** the 10-random-saccade concatenation. This says the
  class signal lives in a well-placed glimpse, and the random-saccade bag both dilutes it and inflates
  dimensionality — a property of the input geometry + memoryless-latent design (§1, §2), not a probe
  bug.

---

## 9. Mechanism summary (the DistantGap test — NO predictive-coding vocabulary)

1. From one MNIST image, at a random point, six 64-number vectors are cut out: four nearly-identical
   raw 8×8 pixel windows (shifted by 2px), one 2×-shrunk 16×16 window, one 3×-shrunk 24×24 window.
2. Each vector is multiplied through three 128-wide matrix layers; at each layer only the 15 largest
   numbers are treated as "on" when that layer is used to predict things.
3. A fixed-point iteration nudges the 128-numbers of each layer until two mismatch quantities shrink:
   (a) how badly each layer is reconstructed by a matrix times the layer beneath it, and (b) how
   badly each window's 128-numbers (times a per-pair matrix, plus a matrix times the 2-number
   fixation coordinate) match every OTHER window's 128-numbers.
4. After the iteration settles, each matrix is nudged by (mismatch ⊗ input)/batch and then rescaled
   to unit length; no classification loss, no gradient of any loss, no labels are used here.
5. Nothing about digit identity is ever fed in during this phase; the coordinate is the only "where"
   signal and it is position, not class.
6. Ten such random-window settles per image are stacked into one 7680-number vector, and a plain
   linear layer is trained (with labels, with backprop) to map that vector to a digit.
7. The only thing that could make the 7680-vector predict the digit is if the mutual predictability
   among the six windows — how well one window's code linearly predicts another's — differs by digit.
8. But four of the six windows are near-copies, many windows are near-black border, the settle keeps
   no memory between the ten fixations, and each window's code is only pushed to PREDICT others (never
   pulled toward being predicted), so the class-discriminating structure the linear head can exploit
   is weak and indirect.
9. Empirically the classifier lands at 62% (accidentally, from raw-pixel window statistics leaking
   through the untrained-to-classify codes), and a single centered window beats the ten-window bag —
   consistent with "the codes carry the pixels of a lucky window, not a learned class-separating
   representation."
10. What is optimized: window-to-window (and layer-to-layer) reconstruction consistency on the unit
    weight-sphere; the signal that drives it is prediction mismatch, not any digit target.

---

## 10. Spec-code drift (loud list — code vs `specs/mpc_mnist_vanilla.md` §11 D1–D13)

**D-table faithfully realized** (no drift): D1 (v1), D2 (st1), D3 (MNIST), D4 (CPU + 20k subset used
in corr run), D5 (S=8), D6 (o=2), D7 (H=128), D8 (t_infer/step — full-run values 20/0.1 as
tabled; corr-run 100/0.1 is the W6b-addendum re-run), D10 (top-layer concat), D11 (nn.Linear+CE
backprop probe), D12 (fixed transpose), D13 (self-pred excluded, 30 pairs, `include_self_pred=False`
both runs). D9 `/B` matches spec §5 (mean).

**Drift / undocumented additions found:**

1. **`norm_axis` knob is a spec addition not in the frozen D-table** (`model.py:68-71, 356-360`;
   CLI `train.py:76-78`). Spec §5 and the §7 table say *"unit L2 per column, every M-step"* (i.e.
   axis=0). The code adds a `norm_axis` param and the **`mpc_st1_corr_T100_20k` run used `axis=1`**
   (unit-row / SC per-atom) — a departure from the literal spec-§5 constraint. Introduced via the
   W6b diagnostic addendum (spec lines 297, 309), but the frozen D1–D13 table was never given a
   D-row for it. **Minor–moderate**: it changes the weight-normalization geometry (columns→rows,
   including on A `[H,2]`) and is one of the four corr-run levers, so it is load-bearing for
   interpreting the 53.51% result, yet it rides under the sidecar's blanket `deviations_ratified:
   D1-D12` string.

2. **Sidecar provenance string omits D13** (`train.py:151`, hardcoded `"D1-D12
   (specs/mpc_mnist_vanilla.md, 2026-07-12)"`, appears in both run sidecars). D13 (self-pred
   excluded) IS applied and IS the reason `pairs=30`, but the ratification string names only D1–D12.
   The D-additions in #1 (norm_axis) and the corr-run's t_infer=100/lr=0.01/λ=0 re-run values are
   likewise not reflected in that string. **Minor** (provenance-hygiene, not behavior) — but per
   [[artifact-provenance]] the sidecar is the identity, so a downstream consumer reading
   `deviations_ratified: D1-D12` would mis-state the config's deviation set.

3. **`FreeEnergy.total` excludes the synaptic-prior term** (`model.py:147-150`). Spec §6 Eq.2 writes
   `F = cross + residual + Ω(Θ)`; the code's logged/settled "total" is `intra+cross` only. Spec §6
   also says "prior is tracked but not part of the latent-settling target," so this is
   **spec-consistent for the E-step**, but the quantity plotted as "FE total" (and used in the
   FE-rise diagnosis) is **not** the full Eq.2 objective — a reader comparing "the free energy" to
   Eq.2 must know Ω is excluded. **Minor**, flag for clarity.

**Code-faithful-to-spec but candidate PAPER discrepancies** (NOT code-vs-spec drift — surfaced for
the join-audit, per the mandate):
- **E-step omits the v-as-target cross term** (§3): both code (`model.py:314-316`) and the
  transcribed spec-Eq.8 (spec §4) push `z^{ℓ,v}` only as a *predictor* (Σ_q (R^{v,q})ᵀ e_C^{v,q}),
  never as a *target* (Σ_p e_C^{p,v}). Code == spec; whether Eq.8 itself should carry the target
  term is a paper question.
- **`/B` mean M-step** (§6) is spec-faithful but the W6b addendum notes the ngc-learn SC ancestor
  uses SUM — a spec-vs-reference gap, surfaced for the join-audit.
- **Cross target is raw unbounded dense `z^q`** (§3): spec §6 Eq.4 says the target is `z^{ℓ,q}`, so
  code matches; the unboundedness (no z normalization) is a mechanistic route to FE-rise, worth the
  paper-side check on whether targets should be sparse/normalized.

---

## Appendix — the three most suspicious IMPLICIT / EMERGENT behaviors (no spec line prescribes them)

1. **Latents are fully memoryless across saccades** (`model.py:230` fresh alloc + feedforward init
   every `infer`; `train.py:310-314` / `probe.py:65-68` call it per-saccade). The K=10 fixations
   share nothing but the slowly-updating weights; the "saccade trajectory" is a bag of independent
   i.i.d. settles. The only integrator in the system is the weight matrices. A trajectory that
   carries no state is the strongest fundamental-discrepancy candidate.

2. **The six "parallel streams" are 4 near-duplicate raw windows + 2 blurred views**
   (`glimpse.py:81-82` foveal identity-pool, `:109` ±2 offset → ~75% overlap; para/periph
   variance-reduced by 2×2 / 3×3 pooling). The cross-stream SSL signal presumes independent-content
   channels; by construction the channels are near-redundant and statistically mismatched. Emergent
   from the S=8 / o=2 geometry, prescribed by no line as "identical."

3. **The E-step z-update omits the v-as-target cross-error** and the **weights live on a unit
   sphere** (`model.py:314-316` predictor-only `if vv==v`; `:342-343,349-350,353-354` renorm-after-
   update). A latent is only ever pushed to predict others, never pulled toward being predicted; and
   because every weight is renormalized to unit norm after each tiny lr-scaled increment, the raw
   Hebbian magnitude is discarded and learning is a slow rotation on the sphere (making λ_w nearly
   inert). Neither the asymmetric objective nor the magnitude-discarding is called out by any spec
   line as an intended property.

**Bonus implicit facts**: peripheral glimpses are frequently near-black (uniform fixation sampling
over centered digits, §1.2); the action term is position-only and shared across all 6 streams within
a glimpse (§5); the probe's single-centre-fixation arm (73.1%) beating the 10-random-saccade arm
(21.4% on the same subsample) shows the class signal is in a lucky window, not a learned
class-separating code (§8).
