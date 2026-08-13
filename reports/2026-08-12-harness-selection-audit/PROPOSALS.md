# Harness Selection Audit — Proposals Awaiting Human Ratification

**Session**: `russet-burning-river`, 2026-08-12, Thread B. **NONE of the below is implemented.** Per the dispatch's explicit routing and `.claude/rules/novel-research-escalate-dont-default.md`'s hard gate, these are foundational-enough choices (what a memory key means, whether ex-ante selection is the right architecture at all, whether retention and protention should share a mechanism) that an agent may not self-ratify them — including this one, having just landed the Option-1 mechanism fixes in the same session. Strong evidence is a reason to write a sharper proposal, not a license to act on it.

Companion to `reports/2026-08-12-harness-selection-audit/CARTOGRAPHY.md` (Thread A, read-only mapping) and this session's Thread B commit (mechanism fixes to `.claude/hooks/librarian_selflet.py` + `.claude/hooks/user-prompt-submit.py`).

---

## Proposal 1 — Abandon ex-ante selection for retrieval-on-demand (Option 3, NOT authorized to implement)

### The evidence, restated plainly

Every push-based selection layer measured this session shares one property: the harness decides what a consumer sees *before* the consumer has a live question to ask. Two independent observations point at the same conclusion — the first documented in the selflet's own written record, the second a **session self-report** (no instrument measured it; the ledger telemetry went live at 14:23, mid-session, and cannot retroactively instrument this audit):

1. **This session's own selflet ranking agent**, asked what it would have done without its pushed candidate pool, answered *"Same, possibly marginally better."* Four of its eight final carry-items came from its own unprompted Grep/Read — a selector that isn't wired into the architecture at all, improvised because the pushed pool was insufficient.
2. **The cartography wave's own writing process** (Thread A, same session): the three most load-bearing sources for that report — `specs/librarian_selflet_read_set.md`, `librarian_selflet.py`, and the 2026-07-27/2026-08-06 prior-art plans — were never injected by any push mechanism. All three were found by hand-Grep/Glob. *(Session self-report, corroborated by `CARTOGRAPHY.md` §5 — not an instrument reading.)*

Both are instances of the same pattern: **a consumer's own targeted search, run at the moment it actually needs something, outperforms a corpus pre-filtered by a heuristic that ran before the consumer's real question existed.** In the S0–S4 typology (`deepresearch/luhmann_selection_and_the_harness.md` §6), this is "S3 (deferred-to-use) beats S1→S2 (ex-ante heuristic then agent judgment on its output)" — observed twice, independently, in one session (once from a written artifact, once self-reported).

### What Option 3 would mean concretely

Instead of `/valuate` and `/niche` pushing a pre-ranked carry-item pool into context, retention/protention/artifact/durable-affordance lookup would be exposed as a **tool the model calls when it discovers, mid-task, that it needs memory** — e.g. a `search_harness_memory(query, kind)` tool with the model's own live formulation of what it's looking for, rather than a frozen `intent_slug` computed once at session entry. This is architecturally close to what the ranking agent already does when it "improvises" with Grep/Read — Option 3 would make that the *primary* path instead of the fallback.

### Why this is NOT decided here, even with the evidence in hand

Per `.claude/rules/novel-research-escalate-dont-default.md` §"The Hard Gate": this changes **what the memory system fundamentally is** — from a push-at-session-entry architecture to a pull-at-need architecture. That is a foundational choice about the object under design, not an engineering default inside an established contract. The rule is explicit that strong evidence for a foundational change is *precisely* the situation where self-ratification is most tempting and most wrong — "escalate, don't default" exists for exactly this evidentiary shape, not for the ambiguous cases.

Two concrete risks worth surfacing to the human alongside the evidence, so the proposal isn't one-sided:

- **The counter-example this same session found**: the JIT path-matched rules (Claude Code's native `paths:` frontmatter feature) are a genuinely *working* push selector — 6 of 15 fired usefully this session, on a mechanism the human evidently likes enough to have adopted for rules already. Push is not uniformly bad; it's the *corpus-search* push (selflet Stage 1) that measured badly, not push-as-such.
- **Tool-call retrieval has its own failure mode**: a model that doesn't know it needs memory won't call the tool. The current push architecture's value (when it isn't drowned by scoring collapse) is that it surfaces relevant context *the consumer didn't know to ask for* — the catastrophic-forgetting check the whole selflet system exists to run (`memory/feedback_catastrophic_forgetting_consumption_gap.md`). A pure pull model risks re-introducing exactly that gap for anything the consumer doesn't think to search for.

**Recommendation for the human's consideration (not a decision)**: a hybrid is plausible — keep a *lightweight* pushed signal (e.g. "N high-confidence matches exist, here are their one-line titles") as a discovery mechanism, but let the actual body-reading happen via S3 pull once the consumer decides something is worth reading. This session's Option-1 fixes (widened match surface, per-source floors, IDF weighting) make the push side meaningfully better regardless of which way this decision goes — they are not wasted if Option 3 is later adopted, since a pull-based tool would still need a corpus index and a relevance signal to search over.

---

## Proposal 2 — The S4 selflet cache-key defect: what should the key be?

### The defect (mechanism-level, already characterized by the theory note this dispatch builds on)

The selflet cache is keyed on `slugify(intent[:80])` (`.claude/skills/valuate/plan_kapstok.py:53,:481`). `read_selflet_carryitems()` matches by exact heading-suffix equality (`librarian_selflet.py:367`). When the human revises the session's intent — the single most important event a session can have, a ratified pivot — the slug changes, the old heading no longer matches, and `selflet_cache_has_entry()` correctly (from its own local logic) reports "never ran" for the new slug. The gate at `plan_kapstok.py:520` is not wrong given what it can see; what's wrong is that the **address of a memory it's about to force a redundant re-dispatch for was derived from the very string that just changed.**

**Not landed this wave.** This wave's Option-1 authorization was explicitly scoped to the Stage-1 scoring/surface/floor mechanism; the cache-key semantics were separately flagged by the dispatch as "the mechanism fix is authorized, but what the key should be is a design choice worth a human nod." Given the risk (`plan_kapstok.py:520`'s hard gate depends on this exact matching behavior, and a wrong interim choice could silently defeat the gate rather than merely inconveniencing it — a subtler and worse failure than the one it replaces), this session chose NOT to land even a "least-surprising interim fix" without the human's explicit steer on what the key should mean. Verified instead (see CARTOGRAPHY / this session's own replay) that the existing cache — 8 carry-items under today's actual slug — reads correctly under the Option-1-fixed code; nothing about the key mechanism was touched.

### Design options for the human to choose among (none pre-selected)

1. **Key on `(identity, date)` only; make `intent_slug` a field on the entry, not part of the address.** The gate's "did it run today?" check becomes "is there an entry today, and if its recorded intent differs from the current one, surface both to the consumer for a judgment call" rather than a silent binary hit/miss. Closest to what the theory note itself proposes (§5, "Proposed mechanism"). Tradeoff: multiple intents in one session-day now share one cache namespace; the reader needs new logic to disambiguate "same intent, re-run" from "different intent, same day."
2. **Content-hash of a *stable* intent core** (e.g. the first N tokens before any human-driven pivot, or a hash computed once at `/valuate` Step 0 and never revised even if the intent string is later edited). Closest to current behavior in spirit (still an S4 exact-match key) but decouples the key from the mutable display string. Tradeoff: requires deciding what counts as "the stable core" — itself a design question, and a wrong choice reproduces the same fragility one level removed.
3. **No intent component in the key at all — key purely on `(identity, date, sequence-number)`,** with intent recorded as metadata only, and the gate's semantics change from "did this specific intent run" to "has the selflet agent run at all today, and here's what it ran against." Simplest mechanically; biggest semantic change to what `selflet_cache_has_entry()` promises callers.

Each option changes the **gate's semantics** (`selflet-consumption-is-a-hard-gate.md`), not just an implementation detail — which is why `.claude/rules/domain-guardrails.md` P2 also applies here (a promotion/semantics change to an existing gate wants a recorded human nod, not an autonomous change). This proposal is deliberately unresolved between the three; the human's steer on what "the same request" should mean when intent legitimately evolves mid-session is the missing ingredient, not more engineering.

---

## Proposal 3 — Retention and protention sharing one selector and one quota

### The defect

`query_kind='all'` (`librarian_selflet.py`, both before and after this session's Option-1 fixes — **this proposal is explicitly NOT touched by this session's landed changes**) still runs retention and protention through the same tokenizer, the same weighting, and the same recency tiebreak, competing for slots in one pool. (This session's floor mechanism guarantees each of the four sources — including `retentive_match` and `protentive_match` separately — a minimum presence, which mitigates the *worst* form of this problem — one source zeroing out another — but does not resolve the deeper conceptual question below.)

Retention answers *"has this been done?"* — its failure mode is redoing work, and a six-week-old artifact is exactly as done as a six-hour-old one; the ratified `multi-agent-protocol.md` §"Selflet retention depth" mandates a **month-minimum** window specifically because recency is a *weak* signal here. Protention answers *"is this still the goal?"* — its failure mode is pursuing a superseded intention, and recency is a *strong* signal there (an intention from six weeks ago is more likely superseded than one from six hours ago). One shared `age_days`-ascending tiebreak structurally favors protention's temporal logic and works against the ratified retentive-depth rule.

### Why this is NOT decided here

Whether retention and protention should share a selector AT ALL — as opposed to two genuinely separate mechanisms with different scoring philosophies — is the kind of question this session's dispatch explicitly carved out as foundational ("propose, don't decide"). This session's floor fix is a **reversible engineering patch** that makes the current shared-quota architecture less bad; it is not an answer to whether the architecture is right. Two directions the human might choose between:

1. **Keep one selector, give retention and protention separate sort keys within it** (retention: relevance-then-month-wide-age-band; protention: relevance-then-recency-descending) — smallest change, keeps one code path, risks becoming two special cases bolted onto one function that neither reads cleanly.
2. **Split into two genuinely separate selectors with their own quotas, entry points, and (eventually) possibly different corpora or scoring philosophies** — cleaner conceptually, larger surface to build and maintain, and raises the question of what a shared `query_kind='all'` even means once the two are architecturally distinct.

Not resolved here. Flagged for the human alongside Proposals 1 and 2 as a batch of foundational memory-architecture questions this audit surfaced but does not have standing to close.

---

## Summary table — what's proposed, what evidence backs it, what's explicitly NOT decided

| Proposal | Evidence this session | Landed this wave? | What's undecided |
|---|---|---|---|
| 1. Retrieval-on-demand (Option 3) | Ranking agent's own "same, possibly marginally better" without its pool; this audit's own writing process found its 3 key sources by hand, not injection | No — write-up only | Whether to replace push with pull at all; if so, whether to keep a lightweight discovery signal |
| 2. Cache-key semantics | 8→0 carry-item loss on intent revision, measured live this session (per the theory note); this session's replay confirms the OLD key still reads correctly post-Option-1-fix | No — key untouched this wave | What "the same request" should mean when intent legitimately changes mid-session (3 options laid out, none pre-selected) |
| 3. Retention/protention shared quota | Ratified month-minimum retentive-depth rule contradicted by shared `age_days`-ascending sort; this session's floor fix mitigates but doesn't resolve | Partially mitigated (floors), core question untouched | Whether retention and protention should share a selector mechanism at all |

---

*Written 2026-08-12, `russet-burning-river`, Thread B. Companion to `CARTOGRAPHY.md` (Thread A). All evidence cited is either from this session's own live measurements or from `deepresearch/luhmann_selection_and_the_harness.md`, itself built from this session's instrumented dispatch.*
