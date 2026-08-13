# MPC-MNIST Fidelity Audit — RANKED FINDINGS + FALSIFICATION PLAN (W2 join-audit)

*Produced by the W2 join-audit agent (stage2-fusion-architect), 2026-07-14 overnight, T2 lane (teal-watching-bay). Persisted coordinator-direct from the agent's return (a write-permission hook blocked the subagent's report write; content verbatim).*

**Method**: line-by-line diff of the two W1 truth docs, each claim re-verified against source (**code wins over `as_built_mechanics.md`; the on-disk v1 PDF wins over `paper_ground_truth.md`**). All four code files re-read this session; all five coordinator seeds verified against source.

**The two clues every finding is scored against**:
- **(a) Probe-inversion.** Within the corrected run (T100/lr.01/decay0/axis1/20k), as FE FALLS (593.7→505.4), drift-probe test acc DECLINES 43.0→37.9→38.9. Sharp reading: **FE↓ ⇒ probe↓** — the SSL objective is anti-aligned with digit-discriminability.
- **(b) Sampling dominance.** One central fixation (768-dim) beats ten-random concat (7680-dim): **73.1% vs 21.4%** corrected.

## Doc-conflict check (both W1 docs vs source)

No material contradiction between the W1 docs. Two resolutions at source:
1. **Top-down transpose** (W1a flagged a silent-bug surface): paper writes `(W^{ℓ,v})ᵀ` but the correct matrix is `(W^{ℓ+1,v})ᵀ`. **Code uses `W[l+1]`** (`model.py:306-307`) = the *intended dimensionally-correct* transpose. **Code AVOIDED the trap — not a defect.**
2. **W1b "~75% overlap"** is arithmetically off: shift 4px on 8px windows = 50% per-axis / 25% area, not 75%. Conclusion (over-redundant streams) unchanged.

## PART 1 — Ranked findings

| # | Finding | Severity | Explains (a)? | Explains (b)? | Fix cost |
|---|---|---|---|---|---|
| **F1** | Per-patch mean-centering ABSENT (all-positive input) | **FUNDAMENTAL** | **YES (strong)** | weak | 4-line glimpse.py, flag |
| **F2** | Streams over-redundant (foveal 4px overlap + no mean-center + pool asymmetry) | **STRUCTURAL** | moderate | moderate | see F6 |
| **F3** | E-step omits v-as-target cross term (reading A vs B) | STRUCTURAL/PLAUSIBLE (paper self-inconsistent) | moderate *if* B | no | ~15-line, flag |
| **F4** | Self-pred excluded: 30 vs paper 36 (D13) | STRUCTURAL | weak | no | flag EXISTS |
| **F5** | Latents memoryless across saccades | STRUCTURAL — **paper-SILENT, not a violation** | no | **YES (strong)** | not a fix |
| **F6** | Foveal offset o=2 → 4px overlap vs paper 1-2px | PARAMETRIC (⊂F2) | no | weak | build `--foveal-offset` |
| **F7** | Norm axis: paper=column; corr-run used row | PARAMETRIC | possible (corr only) | no | flag EXISTS |
| **F8** | Settling depth T·β=2.0 vs SC 10.0 (D8) | PARAMETRIC — **tested, REFUTED as sufficient** | no | no | flag EXISTS |
| **F9** | M-step batch MEAN /B vs SC SUM (×B eff. lr) | PARAMETRIC | possible | no | `--lr` / build flag |
| **F10** | Action rescale `/27` vs paper `/28` | PARAMETRIC (micro) | no | no | 1-line |
| **F11** | Sidecar `"D1-D12"` omits D13 + norm_axis | PROVENANCE | no | no | 1-line |
| **F12** | Probe reads DENSE z, top-layer only (D10) | PROVENANCE/SILENT | no | no | probe option |

**Reading**: the INVERSION (clue a — the human's puzzle) is explained by objective-alignment findings **F1 (best), F3 (if reading B), F2 (partial)**. Sampling DOMINANCE (clue b) is explained by **F5 + the random policy — but F5 is paper-faithful/SILENT, so clue (b) is a symptom of a weak core, not a fidelity bug** (the paper's healthy model hits 97.5% *with* random saccades). **F1 is the single most likely FUNDAMENTAL cause.** F8 (D8 under-settling, the corrective run's hypothesis) is **refuted as sufficient** — deeper settling flipped FE downward but *worsened* the probe (that IS the inversion).

### F1 — Per-patch mean-centering ABSENT — FUNDAMENTAL — TOP SUSPECT

- **Paper says**: v1 p.9 VERBATIM (v2 same): image→[0,1], then **each extracted view mean-centered** — *"we only center it (i.e., subtract the mean value of patch from the patch group of pixels)"*. Each stream input `g^v` is zero-mean per patch.
- **We do**: no mean subtraction anywhere. `build_glimpse` (`glimpse.py:89-125`) does extract→pool→flatten; `pool_to_s:75-86` none. Input is raw [0,1], **all-positive** into every NWTA layer. (Seed #1 CONFIRMED.)
- **Mechanism**: without centering, each `g^v` carries a large positive **DC/common-mode** = patch ink-fraction; since 6 streams are nested windows of the same fixation, this DC is highly correlated across streams. Feedforward init `z=W·g` (`model.py:232-235`) inherits it; hard top-15 NWTA (`model.py:270`) keeps units aligned with the high-variance common-mode, not the low-variance digit residual. The cross-stream objective (`model.py:284-286`) is minimized *fastest* by predicting the shared DC (trivially cross-predictable), so as FE descends, W/R rotate to reconstruct the common-mode and **squeeze the digit residual out of the winner code**. The probe reads dense settled z (`probe.py:70`) → its digit content *decreases* as FE decreases. Centering removes the DC → forces the SSL signal onto the zero-mean digit contrast.
- **Explains clues?** (a) **YES strong** — cleanest FE↓⇒probe↓ mechanism. (b) weak.
- **Testable**: add `--mean-center-patches`; read drift-probe epoch0-vs-epoch2 + final. **Supports F1**: inversion flips (epoch2≥epoch0) AND final >53.5% (toward ≥70%). **Refutes**: inversion persists, final ≈ control.
- **Fix cost**: 4-line glimpse.py, flag-gated.

### F2 — Streams over-redundant — STRUCTURAL

- **Paper**: 4 foveal 8×8 overlapping **1-2px** (v1 appendix p.23) + para 16×16 + periph 24×24 = multi-scale non-redundant sample.
- **We do**: `o=2` (`glimpse.py:94,109`) → 4px overlap = 25-50% area; + no mean-center (shared DC) + foveal identity-pooled raw vs para/periph 2×2/3×3 averaged (variance ≈¼, ≈1/9). → 4 near-copies + 2 blurs, not 6 independent channels.
- **Mechanism**: cross-prediction among 4 near-copies ≈ identity map → near-zero SSL gradient; minimizing FE teaches copying, not generalizable structure → caps the ceiling, compounds F1's easy-minimum.
- **Explains clues?** (a) moderate, (b) moderate (redundant dims inflate the 7680 concat → random-K10 overfit).
- **Testable**: `--foveal-offset 3` (W3 build) → ~2px overlap; read per-stream probe (does foveal pathway rise?).
- **Fix cost**: expose `--foveal-offset` (build).

### F3 — E-step omits v-as-target (reading A vs B) — STRUCTURAL/PLAUSIBLE

- **Paper**: Eq.8 LHS is `∂F` (ensemble) → implies symmetric target-side pull `−Σ_{q'} e_C^{ℓ,q',v}` (**reading B**); literal RHS is predictor-only (**reading A**). Paper never resolves the inconsistency.
- **We do**: predictor-only (`model.py:313-316`, `if vv==v`), no target-side term. **FAITHFUL to literal Eq.8 (reading A).** (Seed #4 CONFIRMED — resolves W1b's open question: code == literal equation, not a defect.)
- **Mechanism (if B intended)**: under A a latent is only pushed to predict others, never pulled to consensus → predictors collapse toward easy-to-hit targets (the DC, F1) with no counter-pull keeping targets digit-informative → contributes to inversion. Only *if* B is what the authors ran.
- **Explains clues?** (a) moderate conditional on B; (b) no. PLAUSIBLE not CONFIRMED → Part 3.
- **Testable**: `--symmetric-cross` (reading B) vs A control; run only if F1 doesn't close it.

### F4 — Self-pred 30 vs paper 36 (D13) — STRUCTURAL (paper contradiction)

- **Paper**: st1 = "every column predicts all others AND themselves" (v1 p.10); v2 `γ_{v,v}=1`. **36 pairs, 6 self.**
- **We do**: 30 pairs (`model.py:99-104,58`, self excluded). (Seed #3 CONFIRMED.) D13 ratified as-built `[decided-by-human:2026-07-14]` but explicitly NOT immune from the audit.
- **Mechanism**: the self-pair `R^{ℓ,v,v}` gives within-stream lateral autoencoding stabilization; removing it drops one objective term per stream. Real structural gap, not a hyperparameter.
- **Explains clues?** (a) weak, (b) no. Escalated because it's a confirmed contradiction of a ratified deviation.
- **Testable**: `--include-self-pred` (flag EXISTS) → 36 pairs, isolated + combined with mean-center.

### F5 — Latents memoryless across saccades — STRUCTURAL, paper-SILENT (not a violation)

- **Paper**: SILENT; only "*possibly* prior expectation (t−1)" — warm-start allowed, reset equally consistent.
- **We do**: fully memoryless (`model.py:230-235` fresh alloc + feedforward init per saccade; `train.py:310-314`, `probe.py:65-68`). K=10 = bag of i.i.d. settles; only weights integrate.
- **Mechanism**: probe concatenates 10 independent random-fixation snapshots; uniform fixations over a centered digit → most snapshots near-blank border → concat mostly low-info dims. A central fixation captures the whole digit; 10 random ones dilute + inflate dimensionality.
- **Explains clues?** (a) no (constant over run); (b) **YES strong** — the 73.1% vs 21.4% mechanism. But paper-SILENT → symptom of weak core, not a bug.

*(F6–F12 short forms: F6 foveal offset ⊂F2 tunable knob; F7 norm-axis paper=column, baseline axis0 faithful, corr-run axis1 deviant — one of 4 entangled corrective levers; F8 D8 settling refuted-as-sufficient, hold T=100 fixed; F9 batch mean/B vs SC sum = ×100 sphere-rotation step, test via `--lr 1.0`; F10 action `/27` vs `/28` micro; F11 sidecar `"D1-D12"` omits D13+norm_axis — fix string; F12 probe dense-z top-layer, paper-SILENT lever.)*

## PART 2 — W4 GPU hypothesis cell plan

**Base config** (corrected, one-thing-changed per cell; all flags verified against `train.py:53-86` argparse):

```
--run-name <cell> --seed 42 --device cuda \
  --t-infer 100 --dt-tau-z 0.1 --lr 0.01 --weight-decay 0 --norm-axis 1 \
  --limit-train 20000 --epochs 5
```

**Wall-time reality (RTX 3090)**: model is Python-per-op-loop-bound → **cuda ≈ cpu** (GPU helps matmuls, not launch overhead). Corrected T100/20k/5ep ≈ **3-5h/cell**. To fit the core, run experimental cells at `--epochs 3` (gives epoch0+epoch2 → enough to see the inversion flip) ≈ 2-3h; keep CONTROL at 5ep. Probe steps are minutes (run `probe.py` on saved checkpoints).

**Readouts**: DRIFT-probe (`probe.py --checkpoint .../checkpoint_epoch{0,2}.pt --limit-train 10000 --limit-test 2000`; inversion=epoch2<epoch0, flip=epoch2≥epoch0); FINAL probe (`checkpoint_final.pt`, full 60k/10k); PER-STREAM probe (F2/F6).

**Cell 1 — `mpc_fid_control`** (CONTROL, 5ep): base config verbatim. Readout: FINAL (~53.5%) + drift (~43/38/39 inversion). Rule: must reproduce the 53.51%+inversion or all comparisons are void. ~3-5h.

**Cell 2 — `mpc_fid_meancenter`** (F1, FUNDAMENTAL — key cell): `+ --mean-center-patches` (**W3 BUILD**), 3ep. Readout: drift epoch0-vs-epoch2 (flip?) + FINAL. Rule: **F1 CONFIRMED** if epoch2≥epoch0 AND final>53.5% (≥70%); **F1 REFUTED as sufficient** if inversion persists & final≈control. Highest info gain. ~2-3h.

**Cell 3 — `mpc_fid_meancenter_selfpred`** (F1+F4, most paper-faithful): `+ --mean-center-patches --include-self-pred` (self flag EXISTS), 3ep. Readout: FINAL + drift vs Cell 2. Rule: Cell3>Cell2 → self-pred contributes (restore D13); ≈ → D13 exclusion defensible. Run if Cell 2 shows promise. ~2-3h.

*(Cell 4 `mpc_fid_selfpred` — `--include-self-pred` isolated, D13 escalation evidence; Cell 5 `mpc_fid_offset3` — `--foveal-offset 3` (W3 BUILD), per-stream readout for F2; Cell 6 `mpc_fid_lr_bigstep` — `--lr 1.0` for F9/F7 ×B lever.)*

**Budget**: Cells 1(5ep)+2,3,4(3ep) ≈ 10-14h → FUNDAMENTAL core fits one night. If tight, keep 1,2,3,5.

## PART 3 — `[blocked:human-decision]` candidates (morning brief)

1. **D13 self-pred (30 vs 36)** — CONFIRMED paper contradiction of a ratified-as-built deviation. Decision: keep anti-tautology exclusion or restore paper-literal "and itself"? Evidence: Cells 3+4. Foundational objective choice → human-gated.
2. **F1 mean-centering** — spec §7 MISSING a paper-prescribed step; fixing it changes the input *data distribution* (all-positive→zero-mean). Decision: ratify **D14: per-patch mean-centering** + make it the vanilla default. Evidence: Cell 2.
3. **F3 reading A vs B** — paper self-inconsistent; "no clean recipe" → auto-escalation. Decision: is the objective asymmetric (A, as-built) or symmetric-consensus (B)? Likely red herring (literal supports A).
4. **F7 norm-axis** — paper=column, corr-run=row; no D-row. Decision: canonical vanilla axis + add D-row.
5. **F11 provenance** — sidecar `"D1-D12"` is factually wrong; update to name D13 + new D14/D15 on the same ratification wave.
