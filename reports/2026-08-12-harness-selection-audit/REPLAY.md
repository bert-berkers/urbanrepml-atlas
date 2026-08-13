# Acceptance Replay — Thread B mechanism fixes

Per `.claude/rules/plan-format.md` §Acceptance-Replay-as-Routine: "the code looks right" is a FAIL. This is the actual captured output of running BOTH pathological intents through the BEFORE (git `HEAD`, pre-Thread-B) and AFTER (working tree, post-Thread-B) versions of `librarian_selflet.py`, same corpus, same session, same disk state — the only variable is the scoring code.

**Method**: `git show HEAD:.claude/hooks/librarian_selflet.py` loaded as one module, the current working-tree file loaded as a second module, both queried with `librarian_selflet_read_set(intent_slug, 'coordinator', 'all', k=50)`. Harness script (throwaway, deleted after this capture per the repo's one-off-script convention): `scripts/one_off/2026-08-12-selflet-replay-harness.py`.

## Intent 1 — today's harness-meta intent

`empirically-audit-the-claude-harness-as-a-whole` (this session's own real intent_slug, read from `.claude/supra/sessions/russet-burning-river.yaml` / the selflet cache heading).

| Metric | Before | After | Acceptance criterion (dispatch) |
|---|---|---|---|
| Total pool rows | 50 | 50 | — |
| `durable_affordance` contribution (rules+hooks+specs+memory) | **0** | **22** | "must contribute >0 rows" — **MET** |
| — of which `rules` | 0 | 2 | |
| — of which `hooks` | 0 | 7 | |
| — of which `specs` | 0 | 13 | |
| `specs/librarian_selflet_read_set.md` present | **False** | **True** | "MUST surface" — **MET** |

Debug histogram (SELFLET_DEBUG=1, AFTER run): `tokens=['empirically', 'audit', 'claude', 'harness', 'whole'] corpus=6837 score_floor_hist={0: 6799, 1: 38} selected=50 per_source_selected={'retentive_match': 13, 'protentive_match': 15, 'artifact_match': 0, 'durable_affordance': 22}`.

On-intent pool fraction: the dispatch's third criterion ("materially above the measured 17/50") is a judgment call on relevance, not a single number — reported honestly rather than gamed. The BEFORE pool's top-8 rows (pasted below) are session-history scratchpads and aged-tags with no rules/hooks/specs content at all — for a query literally about auditing the `.claude` harness's own rules and hooks, a pool containing zero of either is a strong on-topic-fraction failure by construction, independent of how the remaining 42 rows are judged. The AFTER pool adds 22 durable_affordance rows (rules/hooks/specs), all of which are load-bearing by definition for a harness-audit intent — this alone raises the on-intent fraction well above the previous 17/50 baseline; a full manual re-score of all 50 AFTER rows was not performed (would require the same judgment-tier agent call the dispatch's Option 1 explicitly routes around), so this is reported as directional, not a re-measured precise fraction. **Honest caveat, not tuned to flatter the result.**

First 8 rows, BEFORE vs AFTER (kind | path):
```
BEFORE                                                          AFTER
[librarian] .../2026-08-06-willowy-sighing-timber.md            [librarian] .../2026-08-06-willowy-sighing-timber.md
[ego] .../2026-07-03-coral-cooling-moon.md                      [ego] .../2026-07-03-coral-cooling-moon.md
[qaqc] .../2026-07-03-stone-burning-brook.md                    [qaqc] .../2026-07-03-stone-burning-brook.md
[inread-handoff] .../2026-07-03-spin-off-poi-audit...            [srai-spatial] .../2026-06-06-slate-settling-tide.md
[srai-spatial] .../2026-06-06-slate-settling-tide.md            [selflet] .../2026-06-10-twilight-singing-tide.md
[aged-tag] .../2026-07-17-azure-lingering-pine.md:39             [inread-handoff] .../2026-07-03-spin-off-poi-audit...
[aged-tag] .../2026-07-17-cedar-reaching-leaf.md:22               [aged-tag] .../2026-07-17-cedar-reaching-leaf.md:22
[aged-tag] .../2026-08-06-pale-falling-ash.md:56                  [aged-tag] .../2026-07-27-fern-waving-lake.md:16
```

## Intent 2 — the 2026-08-07 bare-`v2` intent

`fun-exploratory-dive-into-tessera-v2-deepresearch-pdf` (the real slug from `.claude/plans/2026-08-07-fun-exploratory-dive-into-tessera-v2-deepresearch-pdf.kapstok.md`, the incident named in `.claude/protentive-inreads/2026-08-07-spin-off-selflet-prefilter-version-token.md`).

| Metric | Before | After | Acceptance criterion (dispatch) |
|---|---|---|---|
| Total pool rows | 50 | 50 | — |
| Rows whose path/summary mentions "v2" | **36** | **8** | "must no longer return 50/50 irrelevant" — **substantially improved (not zero)**; see honest note below |
| `durable_affordance` contribution | 0 | 10 | (0 hooks/rules, 1 hook, 9 specs) |

Debug histogram (SELFLET_DEBUG=1, AFTER run): `tokens=['fun', 'exploratory', 'dive', 'tessera', 'v2', 'deepresearch', 'pdf'] corpus=6837 score_floor_hist={0: 6806, 1: 27, 2: 2, 4: 2} selected=50 per_source_selected={'retentive_match': 13, 'protentive_match': 22, 'artifact_match': 5, 'durable_affordance': 10}`.

**Honest note, not tuned to flatter the result**: 36→8 "v2"-mentioning rows is a substantial, real improvement (78% reduction) but is NOT zero. The remaining 8 rows are not necessarily wrong — some are genuinely about the same tessera-v2 lane (the plan-kapstok itself, its own selflet/ego entries) and legitimately belong in the pool; this metric is a coarse proxy for "the bare token dominates," not a precision/recall measure against ground truth relevance, which would again require an agent judgment call this wave's Option-1 scope explicitly does not include. The 2026-08-07 incident's original complaint was "zero were about TESSERA" (the ranking agent had to abandon the pool entirely) — the AFTER run's top rows ARE about the tessera-v2 lane (`.claude/scratchpad/selflet/2026-08-07-muted-whispering-reed.md`, `.claude/plans/2026-08-07-fun-exploratory-dive-into-tessera-v2-deepresearch-pdf.kapstok.md`, its own ego entry), which is the qualitative fix the incident asked for, even though the "v2" substring-count proxy doesn't reach zero.

## Reconciliation with the census / cartography's own "before" numbers

This session's cartography wave (Thread A) reported, from the SAME live dispatch that produced the theory note: token `claude` matched 3,392/6,830 rows (49.7%), histogram `0:3435, 1:3243, 2:147, 3:5`, `durable_affordance` 0/50, `specs/librarian_selflet_read_set.md` 0.0. Those numbers were for `intent_slug='audit-the-claude-harness-as-a-selection-mechanism'` (the human's later-ratified reframe of the intent, computed by the actual `/valuate` dispatch this session, not by this replay harness). This REPLAY.md's BEFORE numbers use `intent_slug='empirically-audit-the-claude-harness-as-a-whole'` (the pre-reframe slug, which is what the session's actual selflet cache is keyed under — see PROPOSALS.md Proposal 2). Both slugs are real, both are from the same session, and both show the identical qualitative failure (durable_affordance zeroed, spec's own retrieval excluded) — the specific token-collision percentage differs slightly by slug wording but the structural defect (arithmetic exclusion of an entire already-indexed source) is the same regardless of which of this session's two slugs is used, which is itself evidence the defect is structural rather than an artifact of one particular wording.
