# Can we crack res10 with cone-sampled probabilistic fusion? — scaling analysis

**Session**: stone-gathering-storm (overnight autonomous, 2026-07-20). Analyst: stage2-fusion-architect.
**Status**: READ-ONLY findings. No training ran. No GPU touched (peer terminal owns it).
**Audience**: the user + future sessions. Terms are defined at first use.

---

## The verdict, in five lines

1. **Full-Netherlands res10 = 3,752,940 hexes × 178 dimensions.** ("res10" = H3 resolution-10
   hexagons, ~15 m edge; "178D" = the fused feature vector per hex — AlphaEarth 64 + POI/hex2vec 50 +
   roads 64, see CLAUDE.md Stage 1.)
2. **Geo-MPC as it exists tonight (the full-resident windmill trainer) CANNOT run res10 — NO-GO.**
   Four hard blockers (below), the binding one being **VRAM**, not disk or RAM.
3. **A cone-lazy architecture (the `LazyConeBatcher` pattern) is the only feasible path**, and it is
   feasible *on paper* — the 284-cone res5→res10 cache already exists on disk (12 GB) and was built
   exactly to dissolve this memory wall. But no geo-MPC trainer consumes it today.
4. **The binding constraint is the per-anchor windmill geometry precompute, not the feature tensor.**
   The feature tensor is ~2.67 GB (survivable); the geometry precompute scales ~70× from South
   Holland and lands in the tens-of-GB range on a 24 GB card → OOM.
5. **A real res10 run is a multi-step engineering + foundational-decision job, not a tonight probe.**
   It needs a parametrized trainer, an anchor-set definition for NL res10, and a human decision on
   full-resident-vs-cone-lazy. See [What a real res10 run needs](#what-a-real-res10-run-needs).

---

## Definitions (read once)

| Term | Meaning |
|---|---|
| **geo-MPC** | The cone-glimpse Meta-Predictive-Coding fusion model. Trains by expectation-maximisation (settle latents, then a local Hebbian weight nudge — no backprop). Spec `specs/stage2_roster/07_cone_glimpse_mpc.md`. |
| **anchor** | A hex the model "looks at". Its k-ring neighbourhood (radius `image_radius`, default 8 ≈ 217 hexes) is the "image" for one training example. |
| **windmill glimpse** | The 9-stream (or 10 with k0-fovea) partition of an anchor's image into 3 concentric core discs + 6 angular sectors. |
| **full-resident trainer** | The current design: the *entire* study-area feature tensor is held in memory at once, and per-anchor geometry is precomputed for *all* anchors up front. Fast for South Holland; does not scale. |
| **cone-lazy trainer** | The unbuilt alternative: process independent "cones" (res5-rooted hex subtrees, ~1,500 hexes each) loaded 32-at-a-time from disk. Memory-bounded regardless of study-area size. |
| **VRAM / RAM** | GPU memory (RTX 3090 = 24 GB) / system memory (64 GB). |

---

## (a) Geo-MPC as-it-is-tonight = NO-GO — four blockers with numbers

The trainer is `scripts/mpc/train_south_holland.py`. Every claim below is `file:line`-cited.

| # | Blocker | Evidence (`file:line`) | Consequence |
|---|---|---|---|
| 1 | **Hardcoded to South Holland res9 — no retarget CLI** | `STUDY_AREA="south_holland"` `:68`; `RESOLUTION=9` `:69`; `_paths()` builds all paths from these `:98-110`; `build_parser()` `:405-511` has **no** `--study-area`/`--resolution` flag; checkpoint name hardcoded `mpc_south_holland_res9_...` `:225` | Res10-NL is a **code change**, not config-only |
| 2 | **NL res10 grid lacks the required `is_anchor` column** | Trainer raises if absent `:235-236`; verified on disk: `netherlands/regions_gdf/netherlands_res10.parquet` cols = `[geometry, region_id]` (3,897,535 rows); South Holland res9 has `[geometry, is_anchor, region_id]` | An anchor set must be **generated** before any run |
| 3 | **Full-resident architecture won't fit** | `SraiH3GridBackend(cell_ids, features=…)` holds the whole feature tensor `:239`; `TensorWindmillBatcher` precomputes per-anchor RF geometry for **all** anchors on-device `:270-279` | **VRAM OOM** (numbers below) |
| 4 | **Does not consume the pre-built res10 concat** | `_load_features` reads the **three** stage1 parquets separately and z-scores+concats in-process `:162-191` (not `netherlands_res10_2022.parquet`) | The "canonical concat exists" fact does not help this trainer; a loader rewrite is needed |

Plus a fifth, res9-scoped, surface worth flagging because it blocks even *inferring* the current best
checkpoint: **`scripts/mpc/infer_south_holland.py:139-144` rebuilds its glimpse sampler without
`core_radii`/`sector_max_radius`** → it CRASHES on the k0-fovea best-arm checkpoint (a7, 10 streams
vs a 9-stream sampler) and silently mis-settles bounded-sector ones. The hillclimb worked around it
with in-process inference (`scripts/one_off/2026-07-20-mpc-hillclimb-probes.py`). Durable fix ~2 lines.

### The science state behind the "is a probe even meaningful?" question

The geo-MPC's current best configuration (arm a7: bounded sectors smr5 + k0-fovea + settling
calibration) is **statistically indistinguishable from its own untrained initialisation**:
Δ = **+0.0040** mean out-of-fold R², t = 0.60, p ≈ 0.57, 4/8 targets positive
(`reports/2026-07-20-geo-mpc-windmill-hillclimb/README.md`, §"How much of a7's win is real?"). So a
res10 run tonight would be a **memory/scaling** probe of an architecture class whose *training* does
not yet add measurable value even at res9. That is a legitimate thing to test — but it is an
engineering-feasibility question, not a science-value one, and it is blocked on the four items above.

---

## The memory model, with numbers

All figures are for full-Netherlands res10. Anchor count is estimated from the South Holland ratio
(34,302 anchors / 53,602 cells = **64%**), so NL res10 ≈ 3.75M × 0.64 ≈ **~2.4M anchors**.

### Disk + RAM (system, 64 GB) — survivable

| Quantity | Value | Basis |
|---|---|---|
| res10 concat parquet on disk | **2.75 GB** | `ls` verified: `netherlands_res10_2022.parquet` |
| concat rows × cols | 3,752,940 × 178 | sidecar `extra.block_health.n_rows/n_cols` |
| fp32 feature tensor, resident | **~2.67 GB** | 3.75M × 178 × 4 B |
| pandas load-time peak (RAM) | **~10–15 GB** | `_load_features` `:171-191` reads 3 blocks (alpha 0.96 GB, hex2vec 0.75 GB, roads 0.96 GB fp32) + z-score copies + reindex + concat 2.67 GB + `.values` copy 2.67 GB |
| **RAM verdict** | **SAFE on 64 GB** | peak ~15 GB « 64 GB |

### VRAM (RTX 3090, 24 GB) — the crash boundary

The wall is the **per-anchor windmill geometry precompute** in `TensorWindmillBatcher`
(`stage2_fusion/mpc/tensor_windmill.py`), which hoists each anchor's image-cell indices + 9-stream
membership onto the device up front (`train_south_holland.py:263-279`).

| Quantity | South Holland res9 | NL res10 (est.) | Scaling |
|---|---|---|---|
| anchors | 34,302 | ~2.4M | **~70×** |
| Mmax cells/image (image_radius 8) | ~217 | ~217 | — |
| flat index tensor (anchors × Mmax × int64) | ~60 MB | **~4.2 GB** | 70× |
| + 9-stream membership masks / per-stream indices | small | **tens of GB** | 70× |
| + resident feature tensor on device | (small) | ~2.67 GB | — |
| + settling latent buffers (batch × streams × latent) | small | small (batch-bounded) | — |
| **VRAM verdict** | fits trivially (~60 MB geometry) | **OOM on 24 GB** | binding |

**Where the crash is**: the geometry precompute alone (flat indices + per-stream masks for ~2.4M
anchors) exceeds 24 GB before any settling runs. `--batch-size` does **not** rescue this — in this
trainer `--batch-size` is the Hebbian glimpse-averaging unit (`:490`), not a memory-bounded loader;
the full-anchor geometry is materialized regardless of batch size.

### Which settings keep us safe

| Lever | Effect | Recommendation |
|---|---|---|
| **Cone-lazy loading** (not resident) | never materializes the full grid or all-anchor geometry; caps memory at batch×cone size | **the only path that fits res10** |
| `--batch-size` | bounds settling buffers only, NOT the geometry precompute | does not fix the resident wall |
| fp16 feature tensor | halves 2.67 → 1.33 GB resident | marginal; does NOT touch the geometry wall; **and** the E-step precision (`sigma_c`, stability-critical `:451-453`) is untested in fp16 — do not adopt without a numerical check |
| Subsample anchors (`--limit-cells` `:502`) | fewer images → smaller geometry | only a smoke/plumbing tool, not a real res10 run |

### The cone-lazy envelope (what the memory model says is feasible)

The infrastructure to sidestep the resident wall **already exists on disk** (verified this session):

| Asset | Value | Path |
|---|---|---|
| res5→res10 cone cache | **284 cones, 12 GB** | `data/study_areas/netherlands/cones/cone_cache_res5_to_10/` |
| per-cone file size | ~19–23 MB each | `ls -la` sampled |
| `LazyConeBatcher` pattern | loads 32 cones at a time (~0.4–0.7 GB) vs ~6–9 GB for all | CLAUDE.md §"Cone-Based Training Memory Optimization"; `stage2_fusion/data/hierarchical_cone_masking.py` |

A cone-lazy geo-MPC would hold **~0.4–0.7 GB per batch of 32 cones** instead of the full grid — well
inside both 24 GB VRAM and 64 GB RAM. **But no geo-MPC trainer consumes cones today** — the cone cache
was built for the `ConeBatchingUNet` lane, and `train_south_holland.py` is full-resident. Bridging
geo-MPC to the cone loader is the core engineering task, and it raises a foundational question (what a
"glimpse" IS at cone scale) that is human-gated — see the pre-registration draft.

---

## What a real res10 run needs

Not a tonight job. In dependency order:

| Step | Task | Cost estimate (defensible) | Gate |
|---|---|---|---|
| 1 | **Parametrize the trainer** — replace hardcoded `STUDY_AREA`/`RESOLUTION` `:68-69` with CLI flags; generalize checkpoint naming `:225` + sidecar `study_area` `:374` | ~0.5 day engineering | engineering |
| 2 | **Generate `is_anchor` for NL res10** — define + write the anchor set the trainer requires `:235` | ~0.5 day + a definition decision | **`[blocked:human-decision]`** (what IS an anchor at res10) |
| 3 | **Decide full-resident vs cone-lazy** — resident OOMs (above); cone-lazy needs a new batcher bridging geo-MPC to `LazyConeBatcher` | days of engineering if cone-lazy | **`[blocked:human-decision]`** (foundational: what a glimpse IS at cone scale) |
| 4 | **Fix `infer_south_holland.py` geometry reconstruction** (`:139-144`) so any res10 checkpoint is inferable | ~2 lines | engineering |
| 5 | Only then: a single-seed res10 feasibility run under devops headroom discipline | 1 GPU-night, gated serial | training-discipline |

**Bottom line**: res10 is *reachable* — the cone cache exists, RAM is ample, and the model class runs
at res9 — but only through a cone-lazy trainer that does not yet exist, and only after a foundational
human decision about the cone/glimpse geometry at res10 scale. Full-resident res10 on a 24 GB card is
a hard NO.

---

## Provenance

Every numeric claim traces to: `scripts/mpc/train_south_holland.py` (cited by line),
`reports/2026-07-20-geo-mpc-windmill-hillclimb/README.md` (a7 statistics, SH anchor counts),
`netherlands_res10_2022.parquet.run.yaml` (row/col counts, block health), and on-disk `ls`/`du`/schema
checks run in session stone-gathering-storm. Anchor-count for res10 is an **estimate** (SH ratio ×
res10 cells), flagged as such. The VRAM "tens of GB" is an **order-of-magnitude** estimate from the
index+mask scaling, not a measured OOM — measuring it is itself blocked on steps 1–2.
