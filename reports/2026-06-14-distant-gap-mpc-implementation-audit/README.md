# DistantGapMPC — Implementation Audit (read-only verdict)

**Date**: 2026-06-14 · **Auditor**: stage2-fusion-architect (`pine-running-sky`) · **Mode**: read-only, no edits/runs/probes
**Subject**: `DistantGapMPC` — branch-B distant-scale-gap MPC, trained + embedded + probed 2026-06-13.

**Headline verdict**: **MOSTLY** — implemented faithfully to its *own* (corrected) spec; model/trainer/`run_info` are internally consistent. One real correctness issue (bounded-small, un-whitened target leakage) and a cluster of provenance-hygiene gaps. No tautology that invalidates the result, but the trainer's `best_loss` is a *training* loss, not generalization.

Files audited line-by-line this session (private repo only, not part of this public atlas): `distant_gap_mpc.py` · `train_distant_gap_mpc.py` · `ancestor_fovea.py` · `cone_mpc.py` (contrast only) · `06_meta_rep_cone.md` · `run_info.json` · `paths.py` `write_run_info`.

---

## (a) What the code ACTUALLY does vs what it claims

| Claim | Reality | Evidence |
|---|---|---|
| Encoder `f(res9+res8+res7)→z(64D)` | TRUE | `distant_gap_mpc.py:138,149-152,164` |
| Predictor `g(z)→(res6,res5)` | TRUE | `distant_gap_mpc.py:153-157,165` |
| Loss = distant-gap MSE only (no trivial near-scale term) | TRUE — quoted: `loss = F.mse_loss(pred, distant)` | `distant_gap_mpc.py:166` |
| "Shared scale-free MLP" (RGM φ-operator) | OVERCLAIM — it's a plain MLP over a flat 3-stream **concat**; first layer `Linear(192→256)` has position-specific columns per scale, no weight-tying across scales | `_mlp` `distant_gap_mpc.py:93-109`; called `:149,154` |
| Ancestors = parent-mean up the res9→res5 chain, parent direction | TRUE, correct direction | `ancestor_fovea.py:111` (`cell_to_parent` in means), `:161,163` (fovea); `cell_to_children` only for cone enumeration `:181` |
| `create_distant_gap_mpc(model_size=...)` factory, medium ≈ 247k params | TRUE — recomputed 131,648 + 115,328 = **246,976**, matches `run_info` | `distant_gap_mpc.py:178-206`; `run_info.json:27` |
| One training sample = one res9 fixation | TRUE | `ancestor_fovea.py:120-173`; trainer iterates `n` fixation rows `train_*.py:221-234` |

The model is genuinely the corrected (dossier U2/U3) object: the trivial res9→res8 term the empirical gates killed is *absent from the loss* (res8/res7 are input context only). The divergence from `cone_mpc.py` (which keeps the trivial term via its `all_to_all` loss) is **sound and empirically justified**.

---

## (b) Issues table

| `file:line` | Severity | Description |
|---|---|---|
| `ancestor_fovea.py:114` + `:159` | **moderate (correctness)** | **Target leakage.** `df9.groupby(parent_of).mean()` averages the whole grid, so a fixation's res6/res5 ancestor **target** includes that fixation's own res9 cell — which the encoder also receives verbatim as input (`:159`). Self-weight ≈ **1/343 (0.29%) at res6**, **1/2401 (0.042%) at res5**; **coverage-dependent** — spikes where a parent has few covered children (means are over covered cells only, `:24`). Target is not strictly out-of-sample w.r.t. input. Bounded-small on average → does NOT make `best_loss` tautological, but it is an uncontrolled impurity. |
| `train_distant_gap_mpc.py:221-250` | **moderate** | **No held-out/val split.** Trains on ALL `n` fixations; `best_loss` (`:244`) is min **training** loss. The "held-out 0.73 @120 cones" in dossier U4 was a *separate* architect check, not produced here. `run_info best_loss 0.000144` must not be read as generalization. |
| `run_info.json:5` vs `paths.py:551-557` | minor | **`git_hash` points at an unrelated chore commit** `b8d54e9` (`chore(coordinator): tender-slowing-wave …`). Mechanism is benign: run finished 19:44:33, MPC code committed `e90484d` at 19:55:22 — **11 min later**. `write_run_info` truthfully records `git rev-parse HEAD`; the run was executed from the working tree *before* the code was committed. The recorded hash does not identify the code that produced the artifact (run-before-commit gap, not a `write_run_info` bug). |
| `paths.py:530-573` (trainer `:296`) | moderate | **No `*.run.yaml` sidecar.** `write_run_info` emits only `run_info.json`; no `SidecarWriter`/`data_vintage`. The embeddings parquet has no per-artifact sidecar, so the AlphaEarth-2022 vintage is unrecorded at artifact level (contra CLAUDE.md + `specs/artifact_provenance.md`). Figures DO have `.provenance.yaml`; the gap is the embeddings/trainer path. |
| `distant_gap_mpc.py:117-120` | minor | "scale-free operator / RGM-grounded" label on a concat-MLP (see table above). Labeling overclaim, not a functional bug. |

**Internal consistency of `run_info.json`**: PASS. Params (246,976), n_fixations (397,622), z_dim/res/lr/batch all reconcile with the model code and dossier U4.

---

## (c) Faithfulness gaps vs paper/dossier

The dossier's own U1/U5/U7 already flag most of these honestly; the divergences are *acknowledged design choices*, not silent shortcuts — with one open foundational question.

1. **Encoder-only, Adam-not-Hebbian, no decoder** — faithful-by-choice (U7). Defensible.
2. **Cross-*scales-of-one-modality* vs the paper's cross-*streams* (C+F+P).** The paper's parallel streams carry *independent* content; AlphaEarth resolution-levels are provably *not* independent (coarse = parent-mean of fine — the gates' core finding). U5 itself says the independence MPC needs "lives ACROSS MODALITIES, not across scales of one modality." So the shipped model is a scale-gap *reinterpretation* the paper does not directly sanction. **Deepest faithfulness question — foundational.**
3. **No saccade loop, no action-conditioning `A·a(t)`.** Central to the paper; absent here. "Fixation = one res9 hex" reframes the saccade as a *static dataset* rather than a *loop with an action coordinate as input*. Arguably equivalent reformulation, arguably a different object.
4. **No latent settling (E-step) / no NWTA sparsity.** `z` is a single forward pass; the paper's representation emerges from iterative latent inference.

---

## (d) OPEN QUESTIONS FOR THE HUMAN (not recommendations)

Per this project's own `.claude/rules/novel-research-escalate-dont-default.md` (private repo only, not part of this public atlas) — these are surveyed, not resolved. The human read the paper and implemented the idea before.

- **Q-H1 `[blocked:human-decision]` (foundational)** — Is "predict the distant scale-gap of ONE aggregated modality" a faithful MRPC port, or a different object? The paper's streams are independent content channels; AlphaEarth resolution-levels are not (gates proved it). U5 points at cross-modal as the real MRPC independence.
- **Q-H2 `[blocked:human-decision]`** — Target leakage: the fixation's own res9 is averaged into its res6/res5 target (≈0.29% / 0.042% weight, coverage-dependent, can spike where a parent is sparsely covered). Acceptable impurity, or should the ancestor parent-mean exclude the fixation's own res9 (leave-one-out) so `z` trains on a strictly out-of-sample target?
- **Q-H3 `[blocked:human-decision]`** — The saccade loop + action coordinate `A·a(t)` are central to the paper but absent (static per-fixation MLP). Acceptable port, or does it drop MRPC's defining mechanism?
- **Q-D1 `[blocked:design]`** — "Shared scale-free MLP / RGM φ-operator" labels a plain concat-MLP (no cross-scale weight-tying). Relabel, or implement an actually scale-tied operator?
- **Q-D2 `[blocked:design]`** — No latent settling (E-step) / no NWTA. Single forward pass for `z`. Intended v0 simplification, or a gap to close?
- **Q-D3 `[blocked:design]`** — Trainer logs only training loss (no val split); `best_loss` is min-train-loss. Wire a held-out split into the trainer, or keep generalization as an external probe step?
- **Q-D4 `[blocked:design]`** — Provenance: (i) run-before-commit → `git_hash` on an unrelated commit; (ii) no `*.run.yaml` / `data_vintage` sidecar on embeddings. Adopt `SidecarWriter` in the trainer + commit-then-run discipline?

---

*Audit method: [[reproduce-before-diagnose]] applied to code — every prior-inread claim was verified against actual source (all three confirmed true; the `git_hash` mechanism was subtler than "bug"). Load-bearing claims cite `file:line` read this session.*
