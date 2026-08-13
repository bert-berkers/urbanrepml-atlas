# MPC v2 — Epistemic-Saccades Foundations Digest

**A study-before-build digest of the epistemic-saccade mechanism arXiv:2503.21796 v2 adds to vanilla (v1) MPC, mapped onto tonight's fidelity evidence.** Thread C of the vanilla-MPC fundamentals lane (`green-shining-sky`, 2026-07-16). Sibling to [`paper_ground_truth.md`](paper_ground_truth.md), [`findings_ranked.md`](findings_ranked.md), [`hypothesis_ledger.md`](hypothesis_ledger.md), [`README.md`](README.md). Model terms (glimpse, saccade, stream, NWTA, DC/common-mode, E-step/M-step, free energy, probe) are defined once in the README's **"Terms used here"** table and not repeated. Code paths are relative to `scripts/mpc/mnist/`.

**Why this thread exists NOW.** Tonight F1 (per-patch mean-centering) was **REFUTED as a sufficient cause** under a pre-registered rule (H1: adding mean-centering made the linear digit-probe *worse* — epoch-0 34.2% → epoch-2/final 29.9%; control-final 33.5% — while free energy fell 20%). The working mechanistic synthesis (hypothesis ledger H8–H11) is that the binding constraint is the **glimpse policy + a location-unbound readout**, not input preprocessing: fixations are uniform-random over [0,28)² (`glimpse.py:39-49`), the probe feature excludes the fixation coordinates (`probe.py:71-73`), and latents carry no memory across saccades (F5). The single sharpest established clue is **clue (b)**: one central fixation reads **73.1%** where ten random fixations read **21.4%**. The v2 paper's epistemic saccades are the authors' own principled answer to *where to look* — which is exactly the open question the refutation of F1 routed us to.

---

## ⚠️ Provenance honesty box — read before trusting any v2 claim here

The traceability of v1 and v2 content is **asymmetric**, and this digest is only as good as its worst-traced claim. Stated plainly:

| Content | On-disk, re-readable? | Trace path | This digest's confidence |
|---|---|---|---|
| **v1** MPC mechanics (glimpse geometry, action vector `a(k)`, cross-mean Eq.4, E-step Eq.8, random saccade policy) | **YES** — `deepresearch/predictive coding/META-REPRESENTATIONAL PREDICTIVE CODING biomimetic self-supervised learning.pdf` (24pp, PyMuPDF-extracted 2026-07-03/15) | `paper_ground_truth.md` §1–§3, verbatim-anchored to v1 PDF pages | **HIGH** — equation forms verbatim-checked |
| **v2** epistemic-saccade mechanism (focus map, MPC-epi, MPC-stoch, Gumbel, K=15/60, results) | **NO v2 PDF on disk** — v2 exists only as a **WebFetch extraction** of `arxiv.org/html/2503.21796v2` done 2026-07-15 | `paper_ground_truth.md` §4(a) — a second-hand extraction, **not** a re-readable artifact | **MEDIUM (verbal) / LOW (equation)** — see below |
| **v2** focus-map **equation** (the exact mathematical form of the fixation-selection quantity) | **NO** | **not transcribed anywhere traceable** — `paper_ground_truth.md` §4(a) gives only the *verbal* description and locates it as "v2 saccade-planning §" with no equation number | **NONE — flagged THIN throughout** |

**Consequences that bind this whole digest:**
1. Every v2 claim below cites `paper_ground_truth.md` §4(a). That document is honest and careful, but it is a WebFetch summary, not the paper. A future session that wants to *build* on v2 epistemic saccades **must re-fetch or obtain the v2 PDF** and verbatim-check the focus-map equation before any foundational choice is ratified. This digest cannot substitute for that.
2. The 2026-06-11 HTML digest (`deepresearch/2026-06-11-mrpc-arxiv-2503-21796-digest.md`) is **v1-only** — it explicitly says the paper "defers learned motor-control / active saccade policies to future work" (line 42). It contributes **nothing** on v2 epistemic saccades and is not cited as a v2 source here.
3. Where v2 is thin, this digest writes **THIN** literally rather than filling the gap with active-inference convention. Friston is a co-author and "epistemic" is active-inference vocabulary, so the temptation to assume the focus map *is* formal expected free energy (EFE) is strong — that assumption is flagged as an **interpretive association, not a transcribed fact**, everywhere it appears.

---

## Section 1 — What v2 adds, mechanically

### 1.1 The five-row v2 delta (all rows trace to `paper_ground_truth.md` §4(a))

| Aspect | v1 (on-disk PDF) | v2 (WebFetch extraction) | Trace |
|---|---|---|---|
| **Saccade policy** | random / involuntary only — fixations uniformly sampled, not learned | adds **MPC-epi** (epistemic: next fixation = **argmax of a "focus map"**) and **MPC-stoch** (focus map + i.i.d. Gumbel noise, then sampled) | §4(a) saccade-policy row |
| **Focus map = what?** | (none) | **bottom-up saliency + top-down accumulated prediction-error / epistemic demand** | §4(a) — *verbal only, no equation* |
| **Saccade budget K** | 10 (train = test) | **15 train / 60 test** ("test-time processing benefited from larger budgets, e.g. K=60") | §4(a) K-saccades row |
| **Probe** | log-linear (multinomial logistic) | **KNN** (K value not stated in the fetch) | §4(a) probe row |
| **MNIST result** | st1 97.50 / st2 97.81 (log-linear) | **MPC-epi 98.10 ± 0.10**, **MPC-stoch 97.57 ± 0.13** (KNN) | §4(a) results row |

**Bundled-but-separable v2 additions** (named so a future build does not conflate them with the saccade mechanism): a **LGN front-end** at ℓ=0 (Difference-of-Gaussians + Naka-Rushton contrast normalization → ON/OFF contrast-invariant map — this is the pair of inserted equations that shifts v2's numbering +2), a **power-kernel topology** `γ_{v,q}` (self-loops retained at `γ_{v,v}=1`, cross-stream down-weighted by index distance) replacing v1's named st1–st4, and a scaled-up training regime (**100 epochs / 20k iters, batch 200**). None of these four is the epistemic-saccade mechanism; they co-arrived in the same revision. This digest scopes to the **saccade policy** only.

### 1.2 The fixation-selection quantity — what drives *where to look*

The traceable description, verbatim from `paper_ground_truth.md` §4(a):

> next fixation = **argmax of a "focus map"** = **bottom-up saliency + top-down accumulated prediction-error / epistemic demand**

Decomposed into the two additive terms (this decomposition is the digest's reading of the verbal description; the split into two named terms is stated in the source, the *form* of each term is not):

- **Bottom-up saliency** — an input-driven "this region is visually interesting" signal, computed from the image (in v2 most plausibly off the new LGN ON/OFF contrast map, though the source does not say so — **THIN**).
- **Top-down accumulated prediction-error / epistemic demand** — a model-driven "I don't yet understand this region / looking here would resolve the most uncertainty" signal. The word **accumulated** is load-bearing: it means the term integrates something *over the saccades already taken*. You cannot accumulate prediction error across fixations without carrying state across fixations. (This is the F5 coupling — Section 2.3.)

**Candidate set, cadence, latent interaction — THIN.** The source does not state: over what candidate set the argmax runs (every valid pixel position? a coarse grid? the LGN map's resolution?); at what cadence the focus map is recomputed (implied every saccade by "accumulated", but not stated); or the precise coupling to the latent state `z`. All flagged THIN.

### 1.3 MPC-epi vs MPC-stoch — the one semi-formal piece

The only part of the v2 selection rule that admits a near-formal statement is the **stochastic** variant, because "focus map + i.i.d. standard Gumbel noise, then sampled" is the **Gumbel-max trick**: adding i.i.d. standard-Gumbel noise to a set of scores and taking the argmax draws an exact categorical sample with probability proportional to `exp(score)`. So:

```
MPC-epi   (deterministic):  fixation(k+1)  =  argmax_x  Φ_k(x)
MPC-stoch (stochastic):     fixation(k+1)  =  argmax_x  ( Φ_k(x) + G_x ),   G_x ~ Gumbel(0,1) i.i.d.
                                          ≡  sample  x  ∝  exp( Φ_k(x) )        # Gumbel-max identity
```

| Symbol | Meaning | Traceable? |
|---|---|---|
| `Φ_k(x)` | the **focus map** at saccade step `k`, evaluated at candidate fixation `x` = bottom-up saliency(x) + top-down accumulated-prediction-error/epistemic-demand(x) | verbal only (§4(a)); **the functional form of `Φ` is NOT transcribed — THIN** |
| `x` | a candidate fixation coordinate (pixel position on the 28×28 image) | candidate set not stated — THIN |
| `argmax_x` | greedy selection of the peak focus location (MPC-epi) | §4(a) "argmax of a focus map" |
| `G_x` | i.i.d. standard Gumbel(0,1) perturbation, one per candidate (MPC-stoch) | §4(a) "i.i.d. standard Gumbel noise" |
| `k` | saccade index, `k = 0 … K−1`; K=15 train, K=60 test | §4(a) K row |

The MPC-stoch verbal description ("perturbed with Gumbel noise, then sampled") **matches the Gumbel-max identity exactly**, which is why the `≡ sample ∝ exp(Φ)` line can be written with confidence *given the verbal description*. What cannot be written with confidence is `Φ` itself.

### 1.4 The v1 substrate the v2 policy sits on (on-disk-traceable — variable sidebars)

The v2 saccade policy changes only *how `a(k)` is chosen*. Everything the action feeds into is the **unchanged v1 machinery**, and these ARE verbatim-traceable to the on-disk v1 PDF (via `paper_ground_truth.md` §1). Transcribed here so the v2 mechanism is grounded in what actually consumes its output:

**Action vector (v1 Eq., appendix p.23):**
```
a(k) = 2·([x, y]ᵀ / D) − 1                    # fixation (x,y) rescaled to [−1,1]²
```
| Symbol | Meaning |
|---|---|
| `x, y` | raw Cartesian pixel coordinates of the glimpse center (the fixation the policy chose) |
| `D` | image side length; `D = 28` for MNIST |
| `a(k)` | the 2-vector action ∈ [−1,1]² — the model's **only** signal of "where am I looking" |

**Cross-stream mean — where the action enters the model (v1 Eq.4):**
```
μ_C^{ℓ,v,q} = R^{ℓ,v,q}·φ(z^{ℓ,v})  +  A^{ℓ,v,q}·a(k)
```
| Symbol | Meaning |
|---|---|
| `μ_C^{ℓ,v,q}` | stream `v`'s prediction, at layer ℓ, of stream `q`'s latent |
| `R^{ℓ,v,q}` | cross-prediction weight (the SSL wiring) |
| `φ(z^{ℓ,v})` | NWTA-activated latent of the predicting stream `v` |
| `A^{ℓ,v,q}·a(k)` | the **action-conditional** term — the fixation coordinate is the ONLY place `a(k)` enters (it does NOT enter the within-stream mean Eq.3) |

**Key consequence for v2:** the model already *conditions its predictions on where it is looking* through `A·a(k)`. What v2 adds is a **closed loop** on top: the model now also *chooses* `a(k+1)` from its own accumulated prediction-error state, instead of drawing it at random. The prediction machinery is unchanged; the fixation *source* changes from an external RNG to an internal focus map.

---

## Section 2 — What a saccade IS: v2 vs our as-built

### 2.1 Side-by-side

| Dimension | **v2 epistemic (MPC-epi/stoch)** | **Our as-built (`scripts/mpc/mnist/`)** | Trace (as-built) |
|---|---|---|---|
| **Fixation selection** | argmax (or Gumbel-sample) of a focus map = saliency + accumulated prediction-error/EFE | **uniform-random** integer over [0,28)², drawn fresh every saccade from a never-reset seeded stream | `glimpse.py:39-49`; ledger H-verified-facts |
| **What the glimpse sees** | same 6-stream multi-scale composite (+ v2 LGN ON/OFF front-end) | 4 foveal 8×8 raw + 1 parafoveal 16→8 + 1 peripheral 24→8, no LGN | `glimpse.py:117-134` |
| **What persists across saccades** | **the focus map's accumulated term** — prediction-error/epistemic state must carry forward for "accumulated" to mean anything | **nothing** — latents feedforward-init per glimpse (`z=W·φ(below)`), no cross-saccade memory (F5) | `model.py:230-235`; README F5 |
| **Loop topology** | **CLOSED**: fixation → glimpse → settle → update focus map → next fixation | **OPEN**: RNG → fixation → glimpse → settle → (no feedback) → RNG → next fixation | `train.py` loop; F5 |
| **Budget K** | 15 train / 60 test | 10 train = test | spec §7; probe.py |
| **Probe feature** | KNN on frozen latents (coords status unknown in fetch) | linear probe on concat of top-layer latents `z^{L-1,v}` — **fixation coords NOT in feature** | `probe.py:71-73` (7680-dim = 10×6×128) |

### 2.2 The single load-bearing structural difference

Our as-built loop is an **open loop**: an external random number generator supplies each fixation; nothing the model computes influences where it looks next. The v2 epistemic loop is a **closed loop**: the model's own accumulated prediction-error/epistemic state *is* the fixation source. This is not a hyperparameter difference — it changes what a "saccade sequence" *is*. In v1/as-built, the K saccades are a **bag of i.i.d. samples** (order-invariant, memoryless). In v2, the K saccades are a **trajectory** (each fixation is chosen given all prior ones).

Note carefully: **our as-built random loop is FAITHFUL to v1.** v1 uses random saccades and hits 97.5%. So the open-loop random policy is *not a v1 fidelity bug* — it is the v1 vanilla baseline. It becomes a *deviation* only relative to **v2**, and v2 is explicitly out of scope for the current vanilla contract (spec D1). This digest is therefore scoping a **forward extension**, not an audit finding — which is why every option in Section 4 is human-gated, not a fix to apply.

### 2.3 The F5 coupling — can epistemic selection even work without latent memory?

This is the mechanistically load-bearing observation of the digest, and it is an **inference from the verbal description**, labeled as such:

The v2 focus map's top-down term is **accumulated prediction-error / epistemic demand**. To accumulate prediction error across saccades, *something must carry state across saccades* — an error-map buffer, a running latent, or a persistent belief. Our as-built model has **exactly none of this** (F5: latents are feedforward-initialized fresh per glimpse, `model.py:230-235`; the K saccades share only the slowly-updating weights). 

**Therefore the v2 epistemic mechanism cannot be bolted onto our as-built substrate as-is** — the substrate lacks the cross-saccade state the focus map's "accumulated" term reads from. Any v2-informed build must *also* add cross-saccade memory (the F5 fix), or reduce the focus map to its **bottom-up saliency term only** (which needs no accumulation and no memory). This coupling is why Section 4 options (b) and (c) are distinct foundational choices rather than one.

**Honesty caveat:** this inference rests on the word "accumulated" in a WebFetch summary. If the v2 paper's actual focus map recomputes from scratch each saccade off the *current* glimpse (no accumulation), the F5 coupling weakens. The word "accumulated" is the only evidence, and it is second-hand. Re-fetching v2 to check whether the focus map is genuinely path-dependent is the first thing a build session should do.

---

## Section 3 — Mapping onto tonight's evidence

How the v2 mechanism addresses (or does not) each established clue and finding. "Addresses" here means *mechanistically plausible relief*, not demonstrated — nothing v2 is tested on our substrate.

| Tonight's evidence | What it says | Does the v2 mechanism speak to it? |
|---|---|---|
| **Clue (b) — sampling dominance** (central 73.1% vs random-K10 21.4%; README:28) | *where* the model looks dominates the result | **YES, directly and strongly.** v2's entire thesis is that fixation selection is worth optimizing. A focus map that steers fixations onto the digit (rather than uniform over mostly-blank border) is the paper's own mechanism for capturing what the central-fixation arm captures by hand. This is the tightest v2↔evidence coupling. |
| **H9 — location-binding** (probe cannot condition on WHERE; coords absent from feature, `probe.py:71-73`; central-pinning predicted ≥+20pp) | the readout is location-unbound; a good fixation recovers the digit | **PARTIALLY.** v2 changes *fixation choice* but the fetch does not say v2's KNN probe includes coordinates. Epistemic saccades reduce the *need* to condition on WHERE by making WHERE consistently informative (always steering to the digit) — but if the readout is still location-unbound, the benefit is that every fixation is good, not that the probe learns to gate on location. H9's central-pinning probe is the cheap surrogate that isolates this. |
| **F2 — stream redundancy** (4 foveal near-copies + shared DC; README:111) | cross-prediction among near-copies ≈ identity → weak SSL signal | **INDIRECTLY.** v2's LGN ON/OFF front-end and power-kernel topology (bundled with, but separate from, epistemic saccades) target representation redundancy; the saccade mechanism itself does not. Epistemic saccades could *worsen* redundancy if the focus map keeps returning to the same peak — MPC-stoch's Gumbel noise exists precisely to spread fixations, which is suggestive that the authors hit this. |
| **H1 — F1 mean-centering REFUTED** (epoch-0 34.2 → final 29.9; FE ↓20%; ledger H1) | input preprocessing does not redirect SSL toward digits | **CONSISTENT / re-frames.** F1's refutation pushed the cause from preprocessing toward the glimpse policy. v2 is the paper's principled intervention on *exactly that* axis. So the refutation of F1 is what makes the v2 saccade mechanism the next suspect line, not a competitor to it. |

**Synthesis.** The v2 epistemic-saccade mechanism speaks most directly to **clue (b)** — the single sharpest established result — and to the post-F1-refutation glimpse-policy hypothesis (H8/H9). It speaks only indirectly (via its bundled, separable LGN/topology changes) to the redundancy finding F2, and not at all to the input-distribution finding F1 (which is refuted anyway). It is the paper's own answer to the question tonight's evidence forced open.

---

## Section 4 — Foundation options (ALL `[blocked:human-decision]`)

The glimpse/saccade policy is **foundational** — it defines what an "example" *is* for this model (a bag of random glimpses vs a chosen trajectory). Per [[novel-research-escalate-dont-default]], mapping v2's epistemic saccades onto our substrate is a "paper-gives-no-recipe" case (the focus-map equation is not even traceable), so **every option below is tagged `[blocked:human-decision]` and NONE carries a `[decided…]` tag.** This section lays out the option space with evidence-for and evidence-against per option. It makes **no recommendation** — the ranking of these options is the human's to set.

### Option (a) — central-fixation-only baseline `[blocked:human-decision]`

Replace uniform-random fixations with a single fixed central fixation (or K identical central fixations), no focus map, no learned policy.

- **Evidence FOR:** clue (b) shows central-single = 73.1% vs random-K10 = 21.4% — a **+51.7pp** swing on the same encoder (README:28). This is the largest single lever measured, and it is nearly free (a one-line change). It isolates "is the policy the bottleneck?" cleanly and is the natural floor for any saccade-policy comparison.
- **Evidence AGAINST:** it abandons the multi-fixation premise entirely — it is not MPC's saccade model at all, it is "look once at the middle." It cannot generalize to inputs where the object is not centered (MNIST digits are centered by construction; the geospatial H3 target is not). It is a diagnostic baseline, not a faithful mechanism.
- **F5 coupling:** none — no accumulation, no memory needed.
- **Connects to:** H9 (central-pinning extraction probe is the zero-training surrogate for this option).

### Option (b) — epistemic saccades per the v2 paper `[blocked:human-decision]`

Implement MPC-epi / MPC-stoch: build a focus map (saliency + accumulated prediction-error/epistemic demand), select fixations by argmax (or Gumbel-sample).

- **Evidence FOR:** it is the paper's own principled mechanism and it is what lifts the paper's MNIST result to 98.10% (MPC-epi) / 97.57% (MPC-stoch) (§4(a)). It is the faithful v2 answer to clue (b). It carries the paper's authority.
- **Evidence AGAINST:** **the focus-map equation is not traceable** — building it requires re-fetching v2 and verbatim-checking the form, which has not been done. It cannot be built on the as-built substrate without ALSO adding cross-saccade memory (Section 2.3 — the "accumulated" term needs state the substrate lacks), so option (b) *implicitly contains* option (c). It couples the saccade change to v2's LGN + power-kernel + KNN-probe changes in the paper's reported result, so the 98.10% number does not cleanly attribute to saccades alone.
- **F5 coupling:** **REQUIRED** — the accumulated top-down term needs cross-saccade state (unless reduced to saliency-only, which is closer to option (d)).
- **Connects to:** requires the v2 PDF re-fetch as a hard precondition; foundational per the novel-research gate.

### Option (c) — epistemic saccades + explicit latent memory (the F5 fix) `[blocked:human-decision]`

Option (b) plus a deliberate cross-saccade state mechanism (warm-start latents from the previous fixation's settled state, or a persistent error-map buffer that the focus map reads).

- **Evidence FOR:** it is the *coherent* version of option (b) — it supplies the state the "accumulated" focus-map term structurally requires (Section 2.3). v1 itself hedges toward allowing it ("possibly their prior expectation at t−1", `paper_ground_truth.md` §1.4). It would also address clue (b)'s memoryless-dilution mechanism (F5) independently of the policy.
- **Evidence AGAINST:** it stacks **two** foundational changes at once (policy + memory), which makes attribution hard and violates the one-thing-changed discipline the fidelity cells followed. Latent persistence is paper-SILENT (F5) — the paper neither mandates nor specifies it, so this is invention beyond even v2. Larger build surface, more ways to be subtly unfaithful.
- **F5 coupling:** this option *is* the F5 fix, made explicit.
- **Connects to:** F5 (README:131), H9; the "possibly prior expectation" v1 hedge.

### Option (d) — learned-but-simpler saliency policy `[blocked:human-decision]`

A lightweight fixation policy that steers toward high-information regions using **only the bottom-up saliency term** (e.g. fixate high-ink-density or high-contrast regions), no accumulated/epistemic term, no cross-saccade memory.

- **Evidence FOR:** captures most of clue (b)'s benefit (steer onto the digit, off the blank border) at a fraction of the complexity; needs no cross-saccade state (sidesteps the F5 requirement); is a clean one-thing-changed step from the random baseline. Saliency-only is exactly the term that *doesn't* need memory.
- **Evidence AGAINST:** it is **not** the paper's mechanism — it drops the "epistemic demand" term that is the whole point of MPC-epi (looking where the model is *uncertain*, not just where the image is *bright*). On centered MNIST, ink-density saliency ≈ "look at the middle", so it may collapse toward option (a) and not test anything new. Departs from v2 fidelity while wearing v2 vocabulary — the exact "vocabulary ≠ mechanism" trap the DistantGapMPC audit (README:215) warns against.
- **F5 coupling:** none (saliency-only needs no accumulation).
- **Connects to:** the bottom-up half of the §1.2 focus-map decomposition; H8 (raw-glimpse policy-bottleneck probe).

### Cross-option summary

| Option | Faithful to | Adds memory (F5)? | Build cost | Cleanest attribution to clue (b)? |
|---|---|---|---|---|
| (a) central-only | neither (diagnostic) | no | ~1 line | yes (isolates policy) — but not a saccade model |
| (b) epistemic per v2 | v2 (needs re-fetch) | yes (implicit) | high | no (bundled with LGN/topology/KNN) |
| (c) epistemic + memory | beyond v2 (invents F5 fix) | yes (explicit) | highest | no (two changes at once) |
| (d) saliency-only | neither (v2-flavored) | no | medium | partial (may collapse to (a) on centered MNIST) |

**No option is recommended here.** The table is decision-support for the human, not a ranking. The one hard precondition on options (b)/(c): the v2 focus-map equation must be re-fetched and verbatim-checked before either can be a ratifiable foundation (novel-research gate; the focus-map form is currently untraceable).

---

## Section 5 — Pre-registerable hypotheses (glimpse-policy question)

Bayesian-style rows for the glimpse-policy question, phrased so they can be lifted into [`hypothesis_ledger.md`](hypothesis_ledger.md) when/if a cell is human-ratified. Each states a **prior** (belief the claim is a material cause, before the evidence) and the **evidence that would discriminate**. These connect to the ledger's already-pre-registered extraction probes **H8** (raw-glimpse policy-bottleneck) and **H9** (central-fixation extraction). They are **candidates**, not committed rows — no cell is scheduled and no training is proposed here.

| ID | Hypothesis | Prior (material cause) | Discriminating evidence | Connects to |
|---|---|---|---|---|
| **HP1** | The glimpse **policy** (not the encoder) is the dominant bottleneck: a better fixation policy lifts the probe far more than any preprocessing/objective fix | **~65%, moderate-high** — clue (b)'s +51.7pp swing dwarfs every SSL lever tested (F1 refuted, F8 refuted) | H8 (raw-glimpse ≈ latent, both low-30s ⇒ policy-bound) AND H9 (central-pin ≥ +20pp) both firing = policy is the bottleneck | H8, H9, clue (b) |
| **HP2** | An **epistemic/saliency** fixation policy (any of options b/d) beats uniform-random on the same encoder — steering onto the digit is what matters | **~60%, moderate** — direct extrapolation of clue (b); v2's 97.57–98.10% is existence proof *in the paper*, unproven on our substrate | a policy cell (saliency or epistemic) vs random control, same K, same probe: policy final > random final by ≥ +10pp | Section 4 (b)(d); clue (b) |
| **HP3** | Epistemic selection **requires cross-saccade memory** to beat a memoryless saliency policy — the "accumulated" term is load-bearing, not decorative | **~45%, uncertain** — rests on the untraceable "accumulated" word (Section 2.3); genuinely open | option (c) vs option (d) head-to-head: if (c) > (d) by ≥ +5pp, the accumulated/memory term earns its complexity; if (c) ≈ (d), saliency-only suffices | F5; Section 2.3, 4(c)(d) |
| **HP4** | On **centered** MNIST, a saliency-only policy (d) collapses toward central-fixation (a) — ink-density saliency ≈ "look at the middle" | **~55%, moderate** — MNIST digits are centered by construction; the confound is real | option (d) vs option (a): if (d) ≈ (a), saliency adds nothing over central on this dataset (and the real test needs an off-center dataset / the H3 target) | Section 4 (a)(d); clue (b) |
| **HP5** | Making WHERE consistently informative (good fixations) matters **more** than making the readout location-aware (coords in probe) | **~50%, even** — H9 (central-pin) vs H11 (coord-augmented linear probe) are the two arms; the ledger already priors H9 high (~75%) and H11 low (~20%) | H9 large jump AND H11 null ⇒ fixing the fixation beats handing the probe coordinates (a linear head can't gate on location anyway) | ledger H9, H11 |

**Order-gate note (if any of these is ever promoted):** like the ledger's H1 and H8–H11, any of HP1–HP5 promoted to a committed row must have its prior committed *before* the discriminating probe/cell runs, with the git timestamp as proof (per `hypothesis_ledger.md` "order gate"). These rows are written as *candidates* precisely so the prior is on record now, before any v2-policy cell exists.

---

## Section list (for the return)

1. Provenance honesty box (traceability asymmetry: v1 on-disk vs v2 WebFetch-only vs focus-map-equation untraceable)
2. Section 1 — What v2 adds, mechanically (5-row delta; focus-map quantity; MPC-epi/stoch Gumbel; v1 substrate equations with sidebars)
3. Section 2 — What a saccade IS: v2 vs as-built (side-by-side; open-vs-closed loop; the F5 coupling)
4. Section 3 — Mapping onto tonight's evidence (clue (b), H9, F2, H1-refuted)
5. Section 4 — Foundation options (a/b/c/d, all `[blocked:human-decision]`, evidence-for/against, no recommendation)
6. Section 5 — Pre-registerable hypotheses (HP1–HP5, connected to H8/H9/H11)

---

*Written from `paper_ground_truth.md` §4(a) (v2 = WebFetch extraction, not on-disk PDF) and §1 (v1 = on-disk PDF, verbatim). The v2 focus-map equation is not traceable in any on-disk artifact — an honest thin digest of a mechanism whose principled details await a v2 re-fetch. No `[decided…]` tag appears anywhere; the glimpse policy is foundational and the four options are the human's to rank.*
