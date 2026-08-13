# Geo-MPC windmill hillclimb — South Holland

**Status: COMPLETE (2026-07-20, session `deep-spinning-tide`). All numbers read from on-disk artifacts.**
**Single source of truth**: this README. `latex/report.tex` mirrors it; `latex/report.pdf` is the built PDF.

---

## The verdict, in five lines

1. **What we did.** Trained seven geo-MPC arms on South Holland (res9, 178-D input), varying the
   *windmill glimpse geometry* (bounded sectors, foveal core) and the *settling calibration*
   (deeper E-step, slower M-step). One arm per lever, then combinations. ~6–11 min each, all seed 42.
2. **What happened.** Six of seven arms still **erode**: the trained model scores *worse* on the
   downstream probe battery than its own untrained initialisation. The seventh — **a7 `stack_0720`,
   which combines all three levers — is the first geo-MPC arm ever to land on the positive side of
   zero: +0.0040 mean R²** against its own untrained twin (baseline a1: −0.0840).
3. **What it means.** The **settling calibration is the big lever** (−0.084 → −0.016, an 81% reduction
   in erosion). Bounded sectors alone barely move it (+0.008 / +0.012). k0-fovea alone helps a bit
   (+0.036). Only *stacked* do they cross zero. Erosion is not yet solved — it is *neutralised*.
4. **How confident.** Not very, and we say so up front: a7's +0.0040 has an across-target spread of
   ±0.0187 (SD), 4 of 8 targets positive, t = 0.60 → **statistically indistinguishable from zero**.
   The honest claim is "no longer destructive", not "now constructive".
5. **What's next.** The settling knobs (`m_step_dt`, `n_settle`) are visibly the live lever and were
   tested at exactly *one* setting. **A calibration sweep is the highest-value next experiment** —
   see [Affordances](#affordances--open-ends) (c).

![arm ladder](figures/fig09_arm_ladder.png)

---

## Mission (recorded human directive)

> *"Hillclimb the geo-MPC implementation using lessons from the vanilla-MNIST MPC approach; windmill
> glimpse geometry with **BOUNDED sectors (extend a bit beyond the cores, not infinite)**
> `[decided-by-human:2026-07-20 in-chat]`; exploratory autonomous day-run — no perfect scores
> expected; deliverable: a clear, rigorous, reproducible, easily digestible PDF report with lots of
> images and explicit actionable affordances/questions/open ends for the human."*
> — recorded verbatim in `.claude/plans/2026-07-20-hillclimb-the-geo-mpc-implementation-using-lessons-from.kapstok.md:8`

Operational reading: in the vanilla windmill glimpse a sector reaches all the way to the edge of the
anchor image. The human asked whether **bounding the sector radius** — a windmill with *shorter
sails* — plus lessons carried from the MNIST reproduction, can lift the trained MPC above its
untrained anchor.

---

## Two warnings before any number

### Warning 1 — the two number-families (carried from W1b)

| Family | What it is | Fold regime | Typical magnitude | Role here |
|---|---|---|---|---|
| **1. OOF probe matrix** | Ridge R² on downstream targets (rudifun, leefbaarometer, …) | Spatial 5-fold **out-of-fold**, one shared canonical fold assignment | 0.07 – 0.78 per target; noncirc means 0.35 – 0.47 | **THE verdict metric.** Every arm is judged here. |
| **2. Erosion-microtrace vrz instrument** | A ridge tracker fit *during* training to watch the representation erode | In-sample, lightly regularised | 0.52 – 0.86 (much higher by construction) | **Training-dynamics diagnostic only.** Reads the *shape* of erosion, never the *level* of quality. |

Never average, difference, or table these together. Every number in this report is Family 1 unless
explicitly labelled Family 2.

### Warning 2 — the readout mode (NEW, added this wave)

The canonical "0.474 bar" from 2026-07-07 was measured on the **`anchor` readout**. Every arm and
every untrained twin in *this* report uses the **`average` readout** (fixation-averaged core-disc
latents), because that is what the trainer's own inference path emits. The two are different
embeddings from the same weights:

| Embedding | anchor readout | average readout | Δ |
|---|---|---|---|
| `mpc_untrained` | **0.4741** | **0.4310** | −0.043 |
| `k0fovea_untrained` | 0.4735 | 0.4114 | −0.062 |

So the untrained twins here scoring **0.409 – 0.432** is **not a regression of the untrained
baseline** — it is the same initialisation read out a different way, and it is exactly where the
2026-07-07 `average`-readout numbers already sat. **Within this report every comparison is
average-vs-average, so it is internally like-for-like.** The one thing you must not do is compare an
arm here to the 0.474 anchor-readout bar. See Affordance (g) for the like-for-like re-run.

---

## The seven arms

Exact CLI transcribed from `logs/mpc_hillclimb_chain_2026-07-20.log`. All arms carry
`--device cuda --seed 42 --run-tag <tag>`; only the deltas differ. Run order is load-bearing:
baseline control first, stacked best-guess last.

| # | run-tag | Extra CLI flags | Lever | trained | own untrained twin | **Δ (trained − twin)** | wall |
|---|---|---|---|---|---|---|---|
| a1 | `baseline_0720` | *(defaults)* | control | 0.3469 | 0.4310 | **−0.0840** | 383 s |
| a2 | `smr5_0720` | `--sector-max-radius 5` | bounded sectors | 0.3555 | 0.4319 | **−0.0764** | 394 s |
| a3 | `smr4_0720` | `--sector-max-radius 4` | tighter bound | 0.3596 | 0.4319 | **−0.0723** | 389 s |
| a4 | `ngccal_0720` | `--m-step-dt 0.05 --n-settle 40` | settling calibration | 0.4146 | 0.4310 | **−0.0164** | 657 s |
| a5 | `k0fovea_0720` | `--core-radii 0,1,2,3` | foveal core | 0.3637 | 0.4114 | **−0.0477** | 388 s |
| a6 | `smr5_ngccal_0720` | `--sector-max-radius 5 --m-step-dt 0.05 --n-settle 40` | bounded + calibration | 0.4159 | 0.4319 † | **−0.0160** † | 661 s |
| **a7** | **`stack_0720`** | `--sector-max-radius 5 --core-radii 0,1,2,3 --m-step-dt 0.05 --n-settle 40` | **everything stacked** | 0.4130 | 0.4090 | **+0.0040** | 659 s |

All values are Family-1 **mean OOF ridge R² over the 8 non-circular targets** (rudifun,
leefbaarometer, gezondheid, groen, nabijheid, lucht, inkomen, vierkant). `poi_composition` is probed
but excluded from every mean — it is partly circular with the POI input block.

† **a6's twin carries an asterisk**: it reuses a2's `smr5` untrained twin, which was built at
`n_settle 20` while a6 infers at 40. Flagged in the CSV as `twin_settling_mismatch=True`. a6's Δ is
approximate to about ±0.002; it does not affect the a7 headline.

**Trainer defaults (a1)**: `--sector-max-radius None`, `--core-radii 1,2,3`, `--n-settle 20`,
`--m-step-dt 0.1`, `--dt 0.1`, `--latent-dim 64`, `--n-w 15`, `--sigma-c 30.0`, `--n-fixations 10`,
`--image-radius 8`, `--fixation-policy random`, `--norm-mode unit`, `--batch-size 100`, `--n-epochs 5`.

---

## Terminology, with pictures

![windmill geometry](figures/fig01_windmill_bounded_geometry.png)

*(fig01 and fig06 were rebuilt 2026-07-21 from **real H3 res9 cell polygons** of an actual South Holland
anchor hex — earlier drafts drew a synthetic hex lattice whose cells did not tessellate, leaving gaps that
made the glimpse read as a scattered flower/star blob. The hexagons now tile with zero gaps; the colour
semantics — core discs, sails, unsampled cells, the sector clip ring — are unchanged.)*

A geo-MPC **glimpse** is a *windmill* of hexagons around an **anchor** hex:

- **anchor / fixation** — the hex the model is currently looking at. Its k-ring neighbourhood (radius
  `image_radius`, default 8) is the "image".
- **core discs** — concentric rings close to the anchor (`core_radii`, default `1,2,3`). These are the
  fovea, and **they alone form the readout** (3 rings × 64 latent dims = 192-D embedding).
- **sectors ("sails")** — six angular wedges partitioning everything outside the cores. They never
  enter the readout directly; they *shape the settling* by predicting into the core streams.
- **bounded sector** (`--sector-max-radius`, the human's ask) — clip each sail at a fixed ring, so one
  glimpse can no longer reach across the whole image. `smr 5` keeps rings 4–5; `smr 4` keeps ring 4 only.
- **k0-fovea** (`--core-radii 0,1,2,3`) — add the anchor hex itself as a fourth core disc. Ten streams
  instead of nine, and a 256-D readout instead of 192-D.
- **saccade** — a jump of the anchor to a new hex; K = 10 fixations per image.
- **E-step (settling)** — weights frozen, let latents relax until prediction error stops shrinking.
  "Think." Depth = `n_settle × dt`.
- **M-step (learning)** — latents frozen at settled values, nudge weights by the local Hebbian rule at
  step `m_step_dt`. "Learn." No backpropagation anywhere.
- **free energy (FE)** — the scalar being minimised (total prediction error). A healthy run drives it down.
- **probe / OOF R²** — freeze the embedding, fit a spatial-CV ridge to a target, report out-of-fold R².

*(figure "four geometries" — `fig06_sector_extent_diagram.png` — not included in this public atlas)*

---

## Results

### Per-arm × per-target probe matrix

*(figure "probe matrix" — `fig02_arm_probe_matrix.png` — not included in this public atlas)*

### Where each arm gains and loses

*(figure "delta heatmap" — `fig08_arm_delta_vs_baseline.png` — not included in this public atlas)*

**All arms except a7 erode on every single target.** a1 loses on 8/8; a2 8/8; a3 8/8; a4 8/8; a5 7/8;
a6 8/8. a7 is the only arm with any spread across zero: **4 of 8 positive**.

### How much of a7's win is real?

| arm | mean Δ | median Δ | SD across targets | SE | t (H₀: Δ = 0) | best / worst target | targets positive |
|---|---|---|---|---|---|---|---|
| a1 | −0.0840 | −0.0823 | 0.0356 | 0.0126 | −6.68 | −0.0454 / −0.1534 | 0 / 8 |
| a2 | −0.0764 | −0.0702 | 0.0288 | 0.0102 | −7.50 | −0.0493 / −0.1348 | 0 / 8 |
| a3 | −0.0723 | −0.0665 | 0.0279 | 0.0099 | −7.32 | −0.0462 / −0.1300 | 0 / 8 |
| a4 | −0.0164 | −0.0154 | 0.0107 | 0.0038 | −4.34 | −0.0028 / −0.0370 | 0 / 8 |
| a5 | −0.0477 | −0.0466 | 0.0414 | 0.0146 | −3.26 | +0.0150 / −0.1283 | 1 / 8 |
| a6 | −0.0160 | −0.0167 | 0.0111 | 0.0039 | −4.08 | −0.0019 / −0.0342 | 0 / 8 |
| **a7** | **+0.0040** | **+0.0001** | **0.0187** | **0.0066** | **+0.60** | +0.0372 / −0.0231 | **4 / 8** |

**Read this honestly.** The six eroding arms are eroding *unambiguously* — |t| between 3.3 and 7.5, no
positive targets. a7 is the only arm whose sign is not resolved: t = 0.60 with 7 d.f. is a coin flip
(two-sided p ≈ 0.57), the median is +0.0001, and the sign test is 4-vs-4. **The defensible claim is
"a7 is the first arm whose training is not measurably destructive."** It is not evidence that training
is now *helping*. The human's "I don't expect perfect scores yet" is the right frame.

Per-target a7 deltas: vierkant +0.0372, inkomen +0.0220, gezondheid +0.0080, lucht +0.0045,
leefbaarometer −0.0044, nabijheid −0.0048, rudifun −0.0074, groen −0.0231.

### Mechanistic health — free energy

*(figure "FE curves" — `fig04_fe_curves.png` — not included in this public atlas)*

Every arm's free energy descends monotonically and plateaus; none diverge. The **plateau level**
differs a lot by arm (a1 total-FE plateau 4.99, a7 32.97), but that is a property of the objective
each geometry/settle-depth is solving — a 10-stream, 40-settle model minimises a different quantity
than a 9-stream, 20-settle one. **The FE level is not a quality score and must not be read as one.**
FE curves come from the local wandb datastore (`wandb/run-20260720_0*/run-*.wandb`), all 7 arms.

### What the winning arm looks like on the ground

*(figure "cluster maps" — `fig05_cluster_maps_best_arm.png` — not included in this public atlas)*

This is the most striking qualitative result. The **a1 baseline** produces a high-frequency speckled
mosaic — spatially incoherent, the visual signature of a representation dominated by per-fixation
noise. The **a7 stack** produces large, contiguous, recognisable regions: the dune coast, the Groene
Hart, the Rotterdam port/industry corridor, the Westland glasshouse belt. This is qualitative (no
metric attached) and is partly an expected consequence of deeper settling acting as a smoother — but
it is the first geo-MPC map that reads as urban geography rather than noise. The full map suite below
makes the contrast explicit across several views and two spatial scales.

### The full map suite — a1 vs a7, at two scales

The single fig05 panel is one slice of a larger battery. Figures **fig10–fig19** render the two
anchor arms — **a1 baseline** (192-D readout) and **a7 STACK** (256-D readout) — side by side across
three ways of looking at an embedding, at **two spatial extents**. Three view types, defined once:

- **PC1-turbo** (`fig10`, `fig15`): the single strongest axis of variation in the embedding
  (first principal component), rank-normalised and coloured with the *turbo* rainbow. A continuous
  1-D summary — read it as "how far along the dominant gradient is each hexagon".
- **PC-RGB** (`fig11`, `fig16`): the top three principal components mapped to red / green / blue.
  Colour encodes *position in the top-3 embedding space*; similar colours mean similar embeddings.
- **k-means** at k = 8, 12, 16 (`fig12`–`fig14`, `fig17`–`fig19`): every hexagon assigned to one of
  *k* discrete clusters — a categorical **urban type**. Colours are categorical: adjacent colours mean
  nothing, only *sameness vs difference* does.

**Colour-matching caveat (load-bearing).** In every panel the clusters are recoloured by **size rank**
(the largest cluster gets the same colour in both panels) so the two maps are visually alignable. This
is a *cosmetic* alignment only — **a1 (192-D) and a7 (256-D) are different embedding spaces, so their
clusters are NOT semantically the same type**. Compare *spatial coherence and boundary shape*, never
"cluster 3 here = cluster 3 there".

**South Holland, province scale (`fig10`–`fig14`, PCA + k-means fit on the full 34,302-hex anchor
grid).** The fig05 finding holds and sharpens across every view. In PC-RGB (`fig11`) a1 is a
high-frequency confetti of colour — no region larger than a few hexagons — while a7 resolves into
broad smooth gradients (a red/orange north-east band, a magenta-purple Rijnmond south, cyan on the
islands). In the cluster maps (`fig12`–`fig14`) the contrast is starkest: **a1's k-means is speckle**,
tiny clusters shattered across the province; **a7's is a clean partition** into the dune coast (a
coherent western strip), the Groene Hart green interior, the southern islands, and the port corridor.
Deeper settling acting as a spatial smoother is part of the mechanism, but the province-scale maps are
the clearest single picture of *what "training is no longer destructive" buys you*: a representation
whose nearest-neighbour structure is spatially organised rather than noise.

**Den Haag municipality, neighbourhood scale (`fig15`–`fig19`, PCA + k-means REFIT on the Den Haag
subset only).** Fitting the PCA and k-means on the whole province optimises for province-scale
contrasts, which washes out the finer variation *within* a single city. So the zoom **recomputes**
both PCA and k-means using only the hexagons inside the Den Haag (`'s-Gravenhage`) municipal boundary
— the whole point of the zoom. Doing so resolves clean within-municipality structure for **both**
arms: the dune/beach strip along the North Sea coast, the dense historic centre, the post-war
peripheral districts, and the detached south-eastern exclave all separate into their own clusters
(`fig17`–`fig19`) and their own smooth PC gradients (`fig15`, `fig16`). Two honest notes: (1) at this
scale the dramatic a1-speckle-vs-a7-coherence gap *narrows* — once you refit locally over only ~960
hexes, even the noisier a1 embedding yields contiguous neighbourhood blocks, so the arm difference here
is one of *degree* (a7's boundaries are a little cleaner) rather than the night-and-day province-scale
contrast; and (2) the zoom is at **res9 = 961 hexagons** for Den Haag. A finer res10 zoom (~7× more
hexes, ~6.7k) would show sharper neighbourhood edges, but that needs res10 arm embeddings, which do not
exist for these hillclimb arms — flagged, not rendered (no retraining was in scope).

---

## Discussion — MNIST-lessons transfer scorecard

*(figure "transfer scorecard" — `fig07_transfer_scorecard.png` — not included in this public atlas)*

| MNIST lesson | Geo-MPC arm(s) | Verdict | Evidence |
|---|---|---|---|
| **D8 under-settling** — settle deeper before the M-step | a4, a6, a7 (`n_settle` 20 → 40) | **HELPED — the single biggest lever** | −0.0840 → −0.0164 (81% of the erosion removed) |
| **D9 M-step calibration** — slow the weight update | a4, a6, a7 (`m_step_dt` 0.1 → 0.05) | **HELPED (co-applied)** | Not separable from D8 in this design — both moved together in every arm. A 2×2 would separate them. |
| **Sampling policy dominates** — where you look > how you settle | a2, a3 (bounded sectors) | **NEUTRAL — and this is the surprise** | Alone: +0.0076 (smr5) / +0.0117 (smr4) vs a1. The human's central ask is the *smallest* lever measured here. |
| **Central fixation / foveal detail** | a5, a7 (k0-fovea) | **ONLY WHEN STACKED** | Alone: +0.0363. But it is the difference between a6 (−0.0160) and a7 (+0.0040) — the geometry that finally crosses zero. |

**The surprise, stated plainly.** Going in, the bounded-sector geometry was the hypothesis. It is the
*weakest* of the three levers on its own. What actually moved the needle was the un-glamorous
numerical calibration carried from MNIST D8/D9 — settle longer, learn slower. The geometry levers
then contribute the last ~0.02 that takes the stack across zero. Both matter; the ordering was not
what we expected.

**One anomaly, reported not explained.** The `smr4` and `smr5` *untrained* twins are **byte-identical
on disk** (same MD5), and the `core0123` twin matches the 2026-07-07 `k0fovea_average` control to six
decimals. Empirically: **at untrained initialisation the sector geometry has no effect on the
core-disc readout** — only the *core* geometry does. The twin builder demonstrably passes
`sector_max_radius` to the sampler (verified in
`scripts/one_off/2026-07-20-mpc-hillclimb-probes.py`), so this is a property of the model at random
init, not a wiring bug. It is nonetheless unexplained, and it is why the smr arms legitimately share a
twin. Candidate mechanism (untested): at random init the cross-stream contribution to the core latents
is isotropic and washed out by unit-norm normalisation. See Affordance (e).

---

## Affordances & open ends

*The action list. Each item has a proposed next command.*

**(a) Canonicalize a `sector_max_radius`? — `[blocked:human-decision]`**
Bounding helps, but least of the three levers (+0.008 at smr5, +0.012 at smr4 vs unbounded). smr4
(tighter) beat smr5 in the isolated arms, but a7 used smr5. Recommendation: **do not canonicalize
yet** — the lever is too small to justify freezing geometry on. If you want to settle it:
```bash
python scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag smr4_stack_0721 \
    --sector-max-radius 4 --core-radii 0,1,2,3 --m-step-dt 0.05 --n-settle 40
```
i.e. the a7 stack with smr4 instead of smr5 — the missing cell of the design.

**(b) Adopt k0-fovea into the canonical geometry? — `[blocked:human-decision]`**
Note it only pays off *stacked*: alone +0.036 (still eroding), but it is the a6→a7 difference that
crosses zero. It also changes the embedding width (192-D → 256-D), so adopting it is a contract
change for every downstream consumer. Recommendation: **adopt provisionally for MPC experiments,
do not yet make it the canonical cone-glimpse geometry** — wait for (c) to confirm the effect holds
at a better-calibrated operating point.

**(c) The calibration sweep — the highest-value next experiment. `[recommendation]`**
`m_step_dt` and `n_settle` were tested at exactly one setting each (0.05 / 40) and produced 81% of the
total improvement. Nobody has looked at whether that is a plateau or the leading edge of a slope. A
2×3 grid over `n_settle ∈ {40, 80, 160}` × `m_step_dt ∈ {0.05, 0.02}` on the a7 geometry, ~11–40 min
per cell, is a single overnight chain:
```bash
for ns in 40 80 160; do for dt in 0.05 0.02; do
  python scripts/mpc/train_south_holland.py --device cuda --seed 42 \
    --run-tag cal_ns${ns}_dt${dt}_0721 --sector-max-radius 5 --core-radii 0,1,2,3 \
    --m-step-dt $dt --n-settle $ns
done; done
```
This is the one experiment most likely to move a7 from "not destructive" to "actually constructive".

**(d) `infer_south_holland.py` geometry-reconstruction bug — `[needs:stage2-fusion-architect]`**
Real bug found today. `scripts/mpc/infer_south_holland.py:139-144` rebuilds its
`WindmillGlimpseSampler` **without passing `core_radii` or `sector_max_radius`**, so it faithfully
reconstructs only the *canonical* geometry. Running it on a k0-fovea checkpoint crashes (10-stream
weights vs a 9-stream sampler); running it on a bounded-sector checkpoint **silently settles under the
wrong geometry**. The hillclimb worked around it with in-process inference. The durable fix is ~2
lines (pass the checkpoint config's `core_radii` + `sector_max_radius`), but it is a durable-producer
behaviour change and belongs to stage2's lane.

**(e) Sector geometry has no effect at untrained init — investigate? `[open|0d]`**
The byte-identical-twins anomaly above. Cheap check: instrument one untrained settling pass and log
the norm of the sector→core prediction contribution across settle steps. If it is ~0 at init, the
finding is benign (and worth documenting); if it is large but cancels, something is wrong with the
cross-stream wiring. Estimated 30 min.

**(f) Norm-axis dim0 vs dim1 — `[blocked:human-decision]`, carried unverified**
The `em_trainer` normalises on `dim=0` while the batched path normalises on `dim=1`. **This was
carried from the task brief and was NOT verified in code this session** — do not treat it as an
established finding. It needs a read of both code paths before it is called a bug or a design choice.

**(g) D13 retro-ratification carry-over — `[blocked:human-decision]`**
The MNIST D13 self-prediction wiring choice remains un-human-ratified (MNIST spec §11). The geo-MPC
inherits the same wiring. This hillclimb proceeded-and-documented rather than waiting. Does the
geo-MPC need its own ratification, or does the MNIST one cover it when granted?

**(h) Re-run the canonical anchor-readout protocol for a like-for-like 0.474 comparison. `[recommendation]`**
Everything here is `average` readout. The historical bar is `anchor` readout, and the anchor readout
is *worth ~0.04 R² more* on the untrained model. If a7 gains the same ~0.04 under the anchor readout
it would land near 0.45 — still under 0.474, but the comparison would finally be honest. This is
inference-only (no retraining), ~5 min per arm:
```bash
python scripts/one_off/2026-07-20-mpc-hillclimb-probes.py --only-arm a7   # + a --readout anchor pass
```
Note the probes script currently emits `average` only; adding an anchor-readout pass is a small edit.

**(i) a6's untrained twin is approximate — `[open|0d]`**
Build a6 its own `n_settle 40` smr5 twin so its Δ is exact. ~5 min, inference only. Low priority — it
does not touch the headline.

---

## Reproducibility

**Commits.** The bounded-sector / k0-fovea / chain-runner feature landed at **`18331f0`**. The runs
themselves executed under the shas recorded in each checkpoint sidecar — **a1 under `2536b03`, arms
a2–a7 under `866710e`** (both descendants of `18331f0`; a1 ran while a peer lane committed between
arms). **All sidecars record `git_dirty: true`** — the working tree carried uncommitted peer-lane
edits during the chain, so the shas identify the committed ancestor, not a byte-exact tree state.
This is a provenance weakness and is stated rather than hidden.

**Seed** 42 for every arm and every twin. `n_anchors_effective` = 34,302 for all seven arms —
identical training set, so the probe comparison is not confounded by cell counts (closes the old
"60k-equivalent" question).

**Per-arm CLI** — verbatim from `logs/mpc_hillclimb_chain_2026-07-20.log`:
```
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag baseline_0720
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag smr5_0720 --sector-max-radius 5
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag smr4_0720 --sector-max-radius 4
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag ngccal_0720 --m-step-dt 0.05 --n-settle 40
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag k0fovea_0720 --core-radii 0,1,2,3
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag smr5_ngccal_0720 --sector-max-radius 5 --m-step-dt 0.05 --n-settle 40
python -u scripts/mpc/train_south_holland.py --device cuda --seed 42 --run-tag stack_0720 --sector-max-radius 5 --core-radii 0,1,2,3 --m-step-dt 0.05 --n-settle 40
```
Chain wall time 3,531 s (58.85 min) total (arm-1 CMD to arm-7 OK per the chain log timestamps;
summed per-arm wall times below total 3,530.8 s — the 0.2 s gap is inter-arm logging overhead), all
`rc=0`.

**Evaluation.** `python -u scripts/one_off/2026-07-20-mpc-hillclimb-probes.py` — in-process
geometry-faithful inference per arm + 4 untrained twins, then the 2026-07-07 matrix ridge
command-builder so scores are protocol-identical to the historical table. Shared fold file
`data/study_areas/south_holland/probes/fold_assignments_res9_anchor_k5_seed42.parquet`.

**Figures.** `python scripts/one_off/2026-07-20-hillclimb-figure-suite.py` — every figure carries a
`*.provenance.yaml` sidecar (builder-regenerable, never copied) per
`.claude/rules/viz-discipline.md` §"Figure-creation discipline".

**Input artifacts** (all `region_id`-indexed, sidecars green, vintage 2022):

| Block | Path | Identity |
|---|---|---|
| AlphaEarth 64D | `stage1_unimodal/alphaearth/south_holland_res9_2022.parquet` | `data_vintage: 2022` |
| hex2vec 50D | `stage1_unimodal/poi/hex2vec/south_holland_res9_2022_filter_v2.parquet` | **`filter_id: filter_v2`** (the canonical standard, read from the sidecar not the filename) |
| roads 64D | `stage1_unimodal/roads/south_holland_res9_2022.parquet` | `data_vintage: 2022` |

Concatenated to 178-D, z-scored per block. Each arm's sidecar records `poi_filter_id: filter_v2` and
the SHA-256 of every input artifact under `extra.input_artifact_hashes`.

**Artifacts on disk**

| What | Where |
|---|---|
| Aggregate results | `data/study_areas/south_holland/stage3_analysis/hillclimb_2026-07-20/hillclimb_results.{csv,json}` (200 rows) |
| Arm + twin embeddings | `data/study_areas/south_holland/stage2_multimodal/mpc_cone_glimpse/arms_2026-07-20/` (11 parquets + sidecars) |
| Checkpoints | `.../mpc_cone_glimpse/checkpoints/mpc_south_holland_res9_2022_seed42_*0720*.pt` (+ `.run.yaml`) |
| Chain log | `logs/mpc_hillclimb_chain_2026-07-20.log` |
| FE history | `wandb/run-20260720_07*/run-*.wandb` (offline datastore, 7 runs) |

---

## Figures

**Only figures 01 and 09 are included in this public atlas copy** (`fig01_windmill_bounded_geometry.png`,
`fig09_arm_ladder.png`). The rest are listed below for completeness and live in the private repo.

| Fig | File | Content |
|---|---|---|
| 01 | `figures/fig01_windmill_bounded_geometry.png` | Windmill geometry: unbounded vs smr5 vs smr4 |
| 02 | `figures/fig02_arm_probe_matrix.png` | Arm × target OOF R², trained + twins + references |
| 03 | — | **DROPPED**: no per-arm erosion microtrace exists for the 0720 arms (see below) |
| 04 | `figures/fig04_fe_curves.png` | Free-energy trajectories, all 7 arms |
| 05 | `figures/fig05_cluster_maps_best_arm.png` | a7 vs a1, PC-RGB + k-means over South Holland |
| 06 | `figures/fig06_sector_extent_diagram.png` | Four glimpse geometries side by side |
| 07 | `figures/fig07_transfer_scorecard.png` | MNIST-lessons transfer scorecard |
| 08 | `figures/fig08_arm_delta_vs_baseline.png` | Per-arm × target Δ vs own untrained twin |
| 09 | `figures/fig09_arm_ladder.png` | **The headline**: Δ ladder with the zero crossing |
| 10 | `figures/fig10_sh_pc1_turbo.png` | Map suite — South Holland, PC1-turbo, a1 vs a7 |
| 11 | `figures/fig11_sh_pcrgb.png` | Map suite — South Holland, PC-RGB, a1 vs a7 |
| 12 | `figures/fig12_sh_kmeans_k8.png` | Map suite — South Holland, k-means k=8, a1 vs a7 |
| 13 | `figures/fig13_sh_kmeans_k12.png` | Map suite — South Holland, k-means k=12, a1 vs a7 |
| 14 | `figures/fig14_sh_kmeans_k16.png` | Map suite — South Holland, k-means k=16, a1 vs a7 |
| 15 | `figures/fig15_dh_pc1_turbo.png` | Map suite — Den Haag zoom (refit), PC1-turbo, a1 vs a7 |
| 16 | `figures/fig16_dh_pcrgb.png` | Map suite — Den Haag zoom (refit), PC-RGB, a1 vs a7 |
| 17 | `figures/fig17_dh_kmeans_k8.png` | Map suite — Den Haag zoom (refit), k-means k=8, a1 vs a7 |
| 18 | `figures/fig18_dh_kmeans_k12.png` | Map suite — Den Haag zoom (refit), k-means k=12, a1 vs a7 |
| 19 | `figures/fig19_dh_kmeans_k16.png` | Map suite — Den Haag zoom (refit), k-means k=16, a1 vs a7 |

fig10–19 are built by `scripts/one_off/2026-07-21-mpc-map-suite-v2.py` (Den Haag boundary from CBS
`gemeente2022.gpkg`, `'s-Gravenhage` GM0518, vintage-matched to the 2022 arms). Each carries a
`*.provenance.yaml` sidecar; the builder asserts rows-in == rows-plotted per panel.

**Why fig03 was dropped, not faked.** The plan called for Family-2 erosion-microtrace curves per arm.
`scripts/one_off/2026-07-07-mpc-erosion-microtrace.py` exists, but it was **not run for the 0720
arms** — the only microtrace CSVs on disk are the 2026-07-07 variants, which are different models.
Rather than plot old traces under new labels, the figure is dropped and this is stated. It is
regenerable at any time by running the microtrace script with the arm geometries in `MT_*` env knobs.

---

## Appendix — for LLM / machine consumption

### A1. Per-arm × per-target OOF ridge R² (Family 1)

| target | a1 | a2 | a3 | a4 | a5 | a6 | a7 | untr. canonical | untr. smr5/smr4 | untr. core0123 | untr. smr5_core0123 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| rudifun | 0.3179 | 0.3285 | 0.3305 | 0.3996 | 0.3589 | 0.4015 | 0.4033 | 0.4221 | 0.4203 | 0.4092 | 0.4107 |
| leefbaarometer | 0.4061 | 0.4105 | 0.4144 | 0.4651 | 0.4341 | 0.4638 | 0.4592 | 0.4679 | 0.4673 | 0.4656 | 0.4636 |
| gezondheid | 0.3159 | 0.3260 | 0.3346 | 0.3938 | 0.3222 | 0.3910 | 0.3909 | 0.4076 | 0.4056 | 0.3863 | 0.3829 |
| groen | 0.5578 | 0.5653 | 0.5731 | 0.6165 | 0.5794 | 0.6183 | 0.6178 | 0.6535 | 0.6525 | 0.6404 | 0.6410 |
| nabijheid | 0.3752 | 0.3802 | 0.3834 | 0.4057 | 0.3926 | 0.4064 | 0.4096 | 0.4226 | 0.4295 | 0.4114 | 0.4144 |
| lucht | 0.6257 | 0.6442 | 0.6490 | 0.7710 | 0.6380 | 0.7771 | 0.7687 | 0.7791 | 0.7790 | 0.7663 | 0.7642 |
| inkomen | 0.0707 | 0.0772 | 0.0759 | 0.1347 | 0.0645 | 0.1298 | 0.1183 | 0.1436 | 0.1381 | 0.1073 | 0.0963 |
| vierkant | 0.1061 | 0.1124 | 0.1156 | 0.1302 | 0.1196 | 0.1395 | 0.1364 | 0.1516 | 0.1629 | 0.1046 | 0.0992 |
| *poi_composition** | 0.3208 | 0.3358 | 0.3377 | 0.3682 | 0.3527 | 0.3700 | 0.3648 | 0.3768 | 0.3768 | 0.3767 | 0.3765 |
| **noncirc mean** | **0.3469** | **0.3555** | **0.3596** | **0.4146** | **0.3637** | **0.4159** | **0.4130** | **0.4310** | **0.4319** | **0.4114** | **0.4090** |

\* excluded from the noncirc mean (partly circular with the POI input block).

Reference embeddings (2026-07-07 matrix, `comparator_matrix_2026-07-07` rows in the CSV):
`mpc_untrained_anchor` 0.4741 · `k0fovea_anchor` 0.4735 · `ringagg_k8` 0.4672 · `raw178` 0.4468 ·
`mpc_trained_ns40` 0.4163 · `k0fovea_average` 0.4114 · `mpc_untrained_average` 0.4310 ·
`mpc_trained` 0.3497 · `mpc_trained_anchor` 0.3227.

### A2. Per-arm × per-target Δ (trained − own untrained twin)

| target | a1 | a2 | a3 | a4 | a5 | a6 | a7 |
|---|---|---|---|---|---|---|---|
| rudifun | −0.1042 | −0.0918 | −0.0898 | −0.0226 | −0.0503 | −0.0188 | −0.0074 |
| leefbaarometer | −0.0617 | −0.0569 | −0.0529 | −0.0028 | −0.0315 | −0.0035 | −0.0044 |
| gezondheid | −0.0916 | −0.0795 | −0.0709 | −0.0138 | −0.0642 | −0.0145 | +0.0080 |
| groen | −0.0956 | −0.0872 | −0.0794 | −0.0370 | −0.0611 | −0.0342 | −0.0231 |
| nabijheid | −0.0474 | −0.0493 | −0.0462 | −0.0169 | −0.0188 | −0.0231 | −0.0048 |
| lucht | −0.1534 | −0.1348 | −0.1300 | −0.0081 | −0.1283 | −0.0019 | +0.0045 |
| inkomen | −0.0729 | −0.0608 | −0.0621 | −0.0089 | −0.0429 | −0.0083 | +0.0220 |
| vierkant | −0.0454 | −0.0506 | −0.0473 | −0.0214 | +0.0150 | −0.0234 | +0.0372 |
| **mean** | **−0.0840** | **−0.0764** | **−0.0723** | **−0.0164** | **−0.0477** | **−0.0160** | **+0.0040** |

### A3. Run provenance

| arm | checkpoint | sidecar `git_commit` | dirty | seed | streams | core_radii | smr | FE plateau (total) | anchors | wandb run | wall |
|---|---|---|---|---|---|---|---|---|---|---|---|
| a1 | `..._seed42_baseline_0720.pt` | `2536b03` | true | 42 | 9 | 1,2,3 | — | 4.99 | 34,302 | `run-20260720_071103-6ax8c19l` | 383.0 s |
| a2 | `..._seed42_smr5_0720_smr5.pt` | `866710e` | true | 42 | 9 | 1,2,3 | 5 | 6.14 | 34,302 | `run-20260720_071729-guo93qfk` | 393.8 s |
| a3 | `..._seed42_smr4_0720_smr4.pt` | `866710e` | true | 42 | 9 | 1,2,3 | 4 | 6.99 | 34,302 | `run-20260720_072402-cdowwnk1` | 389.3 s |
| a4 | `..._seed42_ngccal_0720.pt` | `866710e` | true | 42 | 9 | 1,2,3 | — | 19.96 | 34,302 | `run-20260720_073030-34w0xabb` | 656.9 s |
| a5 | `..._seed42_k0fovea_0720_core0123.pt` | `866710e` | true | 42 | 10 | 0,1,2,3 | — | 7.21 | 34,302 | `run-20260720_074128-urtcsb8h` | 387.8 s |
| a6 | `..._seed42_smr5_ngccal_0720_smr5.pt` | `866710e` | true | 42 | 9 | 1,2,3 | 5 | 23.93 | 34,302 | `run-20260720_074755-dn6ka0vs` | 661.4 s |
| a7 | `..._seed42_stack_0720_smr5_core0123.pt` | `866710e` | true | 42 | 10 | 0,1,2,3 | 5 | 32.97 | 34,302 | `run-20260720_075857-x99cmp95` | 658.6 s |

`empty_sector_total = 0` for every arm — no glimpse produced an empty wedge, so no trivial zero
cross-targets were injected.

### A4. Family-2 (erosion microtrace) pointers

No 0720-arm microtraces exist. Historical Family-2 values, for context only, never to be compared
with the tables above: unit-norm 0.8469, k0fovea 0.8603, norm-none 0.5200, cross-stream-removed
0.4900 — CSVs under `reports/2026-07-07-mpc-final-look/analysis/`. Env knobs to regenerate for the
0720 geometries: `MT_CORE_RADII`, `MT_NORM_MODE`, `MT_SIGMA_C`, `MT_STEP_CAP`, `MT_SUFFIX` on
`scripts/one_off/2026-07-07-mpc-erosion-microtrace.py`.
