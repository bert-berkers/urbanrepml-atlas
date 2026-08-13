# Rule Corpus Proposal — a PROPOSED diff over `.claude/rules/`

**Session**: `russet-burning-river`, 2026-08-12, unattended (human asleep).
**Thread**: C of the ratified harness audit.
**Status**: **PROPOSAL ONLY. NOTHING EXECUTED.**

> ## ⛔ Constraint compliance statement
>
> **Zero rule files were modified. Zero rule files were deleted. Not one.**
> `git status` over `.claude/rules/` is clean for this thread; the only files this thread wrote are
> this report and the spec-writer scratchpad entry. Every recommendation below is a proposal for a
> human to accept, reject, or amend. Where I became confident a rule should change, that confidence
> is written into the justification column — not into a file.
>
> Rules in this repo encode incidents someone already paid for: a host crash from six parallel
> subagents (2026-05-15), an identity theft that corrupted a live peer's supra (2026-05-29), a
> retired-baseline POI artifact that sailed through three verification passes (2026-06-12), an
> 11-minute-stale `git_hash` (2026-06-14). The point of this audit is to price them, not to reopen them.

---

## 0. Executive summary — three findings, in order of importance

**Finding 1 — the problem is modality, not volume.** The audit's own history supplies both signs, and
they do not point the same way. Where doctrine is **prose**, it demonstrably fails: two `[escalated|2d]`
items record rule-prose failing to change behaviour across **five and six consecutive passes**
(`.claude/scratchpad/ego/2026-08-07-russet-cooling-ledge.md:657,661,691-692`). Where doctrine is a
**mechanism**, it works: a `SelfletGateError` hard gate forced this very session to produce its central
measurement. The sharpest local case is `coordinator-coordination.md`'s claim-squatting anti-pattern —
it has prose *and* a stderr reporter (`subagent-stop.py::check_claim_narrowing`) and still ran 5 passes
with 1 conversion. **Prose failed; prose + reporter failed; a raising gate succeeded.** Cutting rule
volume does not address this. Converting the right rules to gates does.

**Finding 2 — the rule corpus is not where the context money is.** The 9 unconditional rules cost
**22,636 tokens**, 64% of the 35,391-token floor. But of that, only ~**4,100–6,100 tokens** are safely
convertible to just-in-time loading (§4, Tier A/B) — 12–17% of the floor. Meanwhile **one hook,
`subagent-context.py`, injects 8,003 tokens on *every* subagent spawn**, and this session's ledger logs
**at least 9 `subagent_start` events** (`.claude/ledger/raw/2026-08-12-russet-burning-river.jsonl`; a
**lower bound** — the ledger only went live at 14:23, mid-session, so earlier spawns were never
recorded) = **≥ ~72,027 tokens**, which is **≥ 2.0× the entire unconditional floor**, paid repeatedly
where the floor is paid once. A rule-corpus diet is the small lever. The per-dispatch payload is the big one, and it is
out of this thread's scope — flagged for the harness lane.

**Finding 3 — the "9 unconditional rules" framing understates the always-on set.** Five conditional
rules carry the glob `.claude/**` (`concurrent-edit-hazard`, `coordinator-coordination`,
`multi-agent-protocol`, `no-fallbacks`, `training-discipline`). Any session that touches the harness at
all — i.e. every coordinator session, by construction, since `/valuate` and `/niche` read `.claude/`
paths — loads all five. Measured this session: **16,370 tokens of "conditional" rules fired**, making the
**effective floor ~51,761 tokens**, not 35,391. `.claude/**` is a glob that is technically conditional and
practically unconditional.

---

## 1. Step 1 — what current best practice actually says (and where it is thin)

I searched for authoritative guidance rather than listicles, and I flag below where the evidence is
strong, where it is weak, and where it is genuinely unsettled.

### 1.1 STRONG evidence: instruction-following degrades measurably with instruction density

[**IFScale** (Jaroslawicz et al., arXiv:2507.11538)](https://arxiv.org/abs/2507.11538) is the most
directly relevant primary source found. Verbatim from the abstract:

> "We introduce IFScale, a simple benchmark of 500 keyword-inclusion instructions for a business report
> writing task to measure how instruction-following performance degrades as instruction density
> increases. We evaluate 20 state-of-the-art models across seven major providers and find that **even the
> best frontier models only achieve 68% accuracy at the max density of 500 instructions.** Our analysis
> reveals model size and reasoning capability to correlate with 3 distinct performance degradation
> patterns, **bias towards earlier instructions**, and distinct categories of instruction-following errors."

Two things matter for this audit. First, **degradation is real and measured on frontier models** — this
is the strongest available support for the human's overconstraint hypothesis. Second, **primacy bias**:
models preferentially follow instructions appearing *earlier*. In our harness, rule ordering within the
system prompt is not something any author controls, which means late-loaded rules are structurally
disadvantaged.

**Where this evidence is thin, stated plainly**: IFScale's instructions are *keyword-inclusion
constraints on a single generation task* — 500 orthogonal "include word X" rules. Our rules are ~24
narrative documents encoding conditional policies over a long agentic trajectory. The mapping from "500
keyword constraints → 68%" to "24 policy documents → ?" is **not established by this paper and I will
not pretend it is**. IFScale proves density hurts; it does not tell us where 24 policy docs sit on that
curve. Treat it as directionally load-bearing, not as a calibrated dose-response curve for our case.

### 1.2 STRONG evidence: long context degrades non-uniformly, and distractors specifically hurt

[**Context Rot** (Hong, Troynikov, Huber — Chroma Research, July 2025)](https://www.trychroma.com/research/context-rot)
evaluated **18 LLMs** across Anthropic, OpenAI, Google and Alibaba. Findings relevant here:

- Models "do not use their context uniformly and performance varies significantly as input length
  changes, even on simple tasks."
- **Distractors**: "Even a single distractor reduces performance relative to the baseline (needle only)."
- Retrieval degrades with **needle-question semantic distance** — content that is topically adjacent but
  not on-point is worse than absent.
- Anthropic models specifically: "Claude Sonnet 4 and Opus 4 are particularly conservative and tend to
  abstain when uncertain" (relevant to how *our* model fails — by omission, not confabulation).

The distractor finding is the one that bites us. A rule loaded for a task it does not govern is not
neutral ballast — it is a measured distractor. This session is a live instance: `viz-discipline.md`
(2,271 tok, about raster stamp sizes and README chrome) fired into **this** dispatch because I read a
path under `reports/**`, while writing a harness audit containing zero figures.

**The authors' own caveat, quoted**: "our evaluation is not exhaustive of real-world use cases. In
practice, long context applications are often far more complex, requiring synthesis or multi-step
reasoning." So: the direction is well-evidenced; the magnitude for agentic coding work is not.

### 1.3 STRONG (vendor-authoritative): minimal-set, right-altitude, just-in-time

[Anthropic, *Effective context engineering for AI agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
is the closest thing to a first-party position. Verbatim:

- > "you should be striving for the minimal set of information that fully outlines your expected
  > behavior" — with the qualifier "minimal does not necessarily mean short; you still need to give the
  > agent sufficient information up front."
- On rule bloat, directly: > "teams will often stuff a laundry list of edge cases into a prompt in an
  > attempt to articulate every possible rule the LLM should follow for a particular task. **We do not
  > recommend this.**"
- The **"right altitude"** framing: > "The right altitude is the Goldilocks zone between two common
  > failure modes" — at one end "complex, brittle logic in their prompts", at the other "vague, high-level
  > guidance."
- On hardcoding: > "hardcoding complex, brittle logic in their prompts...creates fragility and increases
  > maintenance complexity over time."
- Trajectory: > "As model capabilities improve, agentic design will trend towards letting intelligent
  > models act intelligently, with progressively less human curation."
- **Just-in-time**: agents should "maintain lightweight identifiers (file paths, stored queries, web
  links, etc.) and use these references to dynamically load data into context at runtime using tools."

This is guidance, not measurement — it carries vendor authority but no numbers. I weight it as strong
*normative* evidence and zero *empirical* evidence.

### 1.4 The sharpest single criterion I found for "does this rule earn its tokens"

From the Anthropic skill-authoring best-practices corpus
([obra/superpowers mirror of the guidance](https://github.com/obra/superpowers/blob/main/skills/writing-skills/anthropic-best-practices.md?plain=1)):

> **"Only add context agents don't already have."**

and, as summarised in practitioner distillations of the same guidance, *the only thing worth putting in
a skill is information that pushes the model away from its defaults.* This is the criterion I applied in
§3's "incident-backed vs restates default competence" column. It is the most actionable thing found in
the whole search, and it happens to be exactly the question the human asked.

### 1.5 Just-in-time / path-matched loading — and an important caveat about our mechanism

JIT loading is the converging industry answer
([TrueFoundry](https://www.truefoundry.com/blog/jit-context-just-in-time-context-agents),
[Sourcegraph](https://sourcegraph.com/blog/context-engineering)). The framing that matters most for an
agentic loop, from TrueFoundry: preloading was rational for single-turn chat, but *"a loop re-sends its
window on every step, so excess is billed repeatedly, not once."* That is precisely our
`subagent-context.py` situation (Finding 2).

Progressive disclosure is the mechanism Anthropic ships for this: metadata always loaded → body on
invocation → references on need. Practitioner writeups of CLAUDE.md hygiene converge on *"every standing
instruction spends attention on every turn"*
([CLAUDE.md best-practices roundup](https://dev.to/nishilbhave/claudemd-best-practices-the-complete-2026-guide-435j)).

**⚠️ CONTESTED / RISK — the `paths:` frontmatter mechanism itself.** Two open Claude Code issues report
the feature is unreliable:
[#17204](https://github.com/anthropics/claude-code/issues/17204) reports the documented `paths:` key
failing where an undocumented `globs:` key works, and
[#22170](https://github.com/anthropics/claude-code/issues/22170) reports `paths:`-carrying rules in
*user-level* `~/.claude/rules/` being silently ignored.

**Our local measurement contradicts the general pessimism for our case**: project-level `paths:` in
`.claude/rules/` **demonstrably works here** — 6 of 15 conditional rules fired in the census dispatch,
and a *different* subset fired in mine (I got `viz-discipline`; the census dispatch got
`helper-output-discipline`), which is exactly the per-dispatch differentiation the feature promises.
So the mechanism is empirically live in this repo. But it is an **undocumented-ish, upstream-flaky
surface**, and every `add-paths-frontmatter` recommendation below inherits that risk. Any conversion
should be verified by a live re-dispatch, not assumed.

### 1.6 What is genuinely UNSETTLED — stated as unsettled

I did not find, and I do not believe there exists, published evidence for any of the following. Where the
audit needs these, it needs local measurement, not citation:

1. **Whether ~24 narrative policy documents (~65k tokens max) measurably degrades a gen-5 coding agent.**
   IFScale is 500 orthogonal keyword constraints; Chroma is retrieval. Neither is our regime. **Nobody has
   published this. I will not fabricate a number.**
2. **Where the crossover point is** between "instruction helps" and "instruction is a distractor" for
   policy-shaped (as opposed to constraint-shaped) instructions.
3. **Whether path-matched loading improves *task* outcomes**, as opposed to saving tokens. The token
   saving is arithmetic; the quality effect is unmeasured anywhere I could find.
4. **Whether a rule that loads *after* the first action on a matched path prevents anything.** This is a
   mechanism question specific to our harness — see §2.3, where I derive an answer from local evidence.

---

## 2. Step 2 — the measured local facts (cited, not re-derived)

All figures from `.claude/ledger/digest/2026-08-12-injection-cost-census.json` (the census) and
`reports/2026-08-12-harness-selection-audit/CARTOGRAPHY.md`. **Caveat preserved verbatim from the
census**: token counts are a labelled `chars/4` approximation; line and char counts are exact.

| Quantity | Value | Source |
|---|---|---|
| Rule files total | 24 | verified on disk this thread (`ls .claude/rules/*.md \| wc -l` → 24) |
| Unconditional (no `paths:` frontmatter) | **9**, 90,545 chars, **22,636 tok** | census; frontmatter re-verified this thread |
| JIT path-matched (`paths:` frontmatter) | **15**, 119,344 chars, **29,836 tok** if all fire | census |
| Conditional rules that fired in the census dispatch | 6 of 15 = **16,370 tok** | census, `measurement_kind: exercised` |
| Unconditional floor (CLAUDE.md + MEMORY.md + 9 rules + SessionStart payload) | 141,563 chars = **35,391 tok** | census `totals.unconditional_floor` |
| Theoretical max (all 24 rules resident) | 260,907 chars = **65,227 tok** | census `totals.theoretical_max` |
| **Effective floor for any harness-touching session** (floor + the five `.claude/**` rules + `helper-output`) | **~51,761 tok** | derived, this thread, from the two rows above |
| `subagent-context.py` SubagentStart payload | **8,003 tok, per spawn** | census, exercised |
| `subagent_start` events logged today | **≥ 9** (lower bound — ledger live from 14:23, mid-session; earlier spawns unrecorded) | `.claude/ledger/raw/2026-08-12-russet-burning-river.jsonl`, 9 rows excl. 1 synthetic replay-test row |
| **Total SubagentStart injection today** | **≥ ~72,027 tok** (≥ 9 × 8,003) | derived, this thread — a bound, not a total |
| `valuate/SKILL.md` / `niche/SKILL.md` | 9,629 / 9,333 tok | census — the two priciest single files anywhere |

### 2.1 The two-class split is the analytical key

A JIT rule that fires only on its own domain's files **costs nothing when idle**. Its cost/benefit is
categorically different from an unconditionally-loaded rule. So §3 asks two different questions:

- **Unconditional rules** → *does this earn a tax on every session regardless of task?*
- **JIT rules** → *is the match condition right?* (too broad = distractor tax; too narrow = silent miss)

### 2.2 The nine unconditional rules were never *chosen* to be unconditional

This is the finding the census framing implies and the cartography states (row 3): **the selector is the
absence of frontmatter.** Nobody scored "should this always load." It loads because nobody scoped it. An
author who forgets `paths:` has silently promoted a rule to a permanent ~2–4k token tax on every session,
and nothing flags this. That is not an argument that the nine are wrong — four of them are explicitly
human-ratified — but it means their unconditional status carries no evidential weight in itself.

### 2.3 ⚠️ A mechanism constraint I derived this thread, load-bearing for every recommendation

The census records the JIT rules as *"Observed injected mid-conversation this dispatch, **immediately
after** a Bash/Read call touched a matching `.claude/**` path."*

**JIT rules arrive AFTER the first action on a matched path, not before it.**

Consequence: **a rule whose entire value is preventing the FIRST action on a path cannot be safely
converted to JIT.** It would arrive one action too late, every time. This single constraint disqualifies
several otherwise-attractive conversions below (`identity-before-work` most sharply) and it is why my
`add-paths-frontmatter` list is shorter than the token arithmetic alone would suggest.

Rules whose value is *evaluative* (read artifact → then check its sidecar → then form a verdict) survive
JIT fine, because the check happens after the read by design. Rules whose value is *prohibitive of the
first move* do not.

---

## 3. The audit axes

Each rule below is classified on four axes.

- **Load class**: unconditional, or JIT with its glob.
- **Fired this session?**: measured, not assumed. Note that "this session" has two dispatches with
  different path histories — the census dispatch and mine — and they loaded *different* rule subsets.
- **Incident-backed?**: does it encode a **dated incident with a real, named cost**, or does it restate
  behaviour a gen-5 model does by default (§1.4's criterion)? Where an incident exists it is **named with
  its cost**.
- **Gate or narration?**: is compliance *checked by a mechanism*, or *cited by an agent while proceeding*?

**Recommendation vocabulary**: `keep-as-is` · `keep-but-convert-to-gate` · `demote-to-spec` ·
`merge-with-X` · `add-paths-frontmatter` · `narrow-paths` · `fix-glob` · `propose-delete`.

---

## 4. THE PROPOSAL — all 24 rules, ordered by my confidence (most confident first)

| # | Rule | Load class | Tok | Fired? | Incident-backed? | Gate or narration? | **Recommendation** | Confidence |
|---|---|---|---|---|---|---|---|---|
| 1 | `plan-format.md` | **uncond.** | 2,807 | n/a | Weak — formatting convention, no costed incident | Narration + a *reporter* hook (`plan_format_check.py`, always returns 0) | **`add-paths-frontmatter`** → `.claude/plans/**`, `.claude/skills/valuate/**`, `.claude/skills/niche/**` | **HIGH** |
| 2 | `coordinator-coordination.md` | JIT `.claude/**`, `skills/**` | 3,941 | ✅ | Yes — 2026-04-19 supra-ghost; 2026-04-17/19 general-purpose orphans | Mixed; claim-squatting is a *reporter* that failed 5 passes | **`fix-glob`** (`skills/**` is dead) + **`keep-but-convert-to-gate`** for claim narrowing | **HIGH** |
| 3 | `multi-agent-protocol.md` | JIT `.claude/**` | **6,062** | ✅ | Yes — 2026-05-18 orphaned bg tasks; 2026-04-18 flywheel audit | Markov contract IS a gate (`markov_check.py`, fail-closed for coordinator) | **`dedupe`** — §Scratchpad Format is *re-stated verbatim* by the 8,003-tok SubagentStart payload | **HIGH** |
| 4 | `srai-spatial.md` | JIT stage/scripts/utils | 502 | ❌ | Project-specific by construction (SRAI-over-h3-py is not a model default) | Narration; audit checklist | **`keep-as-is`** | **HIGH** |
| 5 | `script-discipline.md` | JIT `scripts/**` | 324 | ❌ | Yes — rasterization helpers duplicated 4+ times | Narration | **`keep-as-is`** | **HIGH** |
| 6 | `data-code-separation.md` | JIT stage/scripts/utils | 262 | ❌ | Foundational project invariant (CLAUDE.md #5) | Narration | **`keep-as-is`** (+ optional `data/**` glob add) | **HIGH** |
| 7 | `multi-terminal-commit-discipline.md` | **uncond.** | **794** | n/a | Yes — 2026-06-12 commit `4a20488` swept a peer lane's staged rename | Narration | **`keep-as-is`** — cannot be path-scoped (a `git commit` has no triggering file path) | **HIGH** |
| 8 | `selflet-consumption-is-a-hard-gate.md` | JIT valuate/niche/selflet | 2,846 | ❌ | Yes — 2026-06-20 skipped consumption check | **GATE** — `SelfletGateError` raised in `write_kapstok()` | **`keep-as-is`** — cite as the corpus exemplar | **HIGH** |
| 9 | `index-contracts.md` | JIT stage/scripts/utils | 571 | ❌ | Yes — recurring `h3_index`/`region_id` redo-work ("THIS KEEPS FUCKING US UP") | Narration | **`keep-as-is`** + **`convert-to-gate`** (already an open item: `domain-guardrails` Tier-2) | **HIGH** |
| 10 | `helper-output-discipline.md` | JIT hooks/skills/utils | 1,312 | ✅ (census) | Yes — 2026-05-17: 99.2% of a helper's output was stderr noise; carry-items truncated out of context | Narration, but with an in-code cap precedent | **`keep-as-is`** | **HIGH** |
| 11 | `data-lifecycle.md` | JIT `data/**`, scripts, stage3 | 1,001 | ❌ | Yes — direct user directive 2026-03-14 ("data is not archived ever") | Narration | **`keep-as-is`** | **HIGH** |
| 12 | `concurrent-edit-hazard.md` | JIT `.claude/**` | 1,100 | ✅ | Yes — 2026-03-07 silent cross-session revert of infra edits | Narration | **`keep-as-is`** | **HIGH** |
| 13 | `no-fallbacks.md` | JIT `.claude/**`+stage+utils | 1,382 | ✅ | Yes — user directive + repeated dual-path bugs | Narration | **`keep-as-is`** | **HIGH** |
| 14 | `viz-discipline.md` | JIT `scripts/**`, `stage3_analysis/**`, **`reports/**`** | 2,271 | ✅ **in MY dispatch** | Yes — 2026-03-09 chunky hexes; 2026-05-08 README chrome rejection; D62 baked-in titles | Narration + reporter (`check_figure_provenance.py`) | **`narrow-paths`** — drop bare `reports/**`; scope to `reports/**/*.tex`, `reports/**/figures/**`, builder scripts | **MED-HIGH** |
| 15 | `reproduce-before-diagnose.md` | **uncond.** | 1,317 | n/a | **Ported, not local** — origin is an APMT-sibling incident (2026-06-10), no local costed failure | Narration | **`add-paths-frontmatter`** → `scripts/**`, `stage3_analysis/**`, `utils/visualization.py` | **MED-HIGH** |
| 16 | `training-discipline.md` | JIT **`.claude/**`**+scripts+stage1/2 | 2,573 | ✅ | Yes — **2026-05-15 host OOM/crash from 6 parallel subagents**; 2026-05-16 silent-slow misdiagnosis | Narration | **`narrow-paths`** — drop `.claude/**`; move the serial-dispatch clause into `coordinator-coordination.md` which already owns dispatch | **MEDIUM** |
| 17 | `selflet-semantics.md` | JIT valuate/niche/selflet | 2,964 | ❌ | Yes — 2026-05-18 (0.17 recursive-invisibility) and 2026-06-07 (3 live judge failures) | Narration (interpretive) | **`demote-to-spec`** for §"Architecture history" only (2 retired pivots, ~35% of file); keep score bands + the three zero-cases | **MEDIUM** |
| 18 | `close-out-protocol.md` | JIT coord-scratchpad/inreads/skills | 2,724 | ❌ | Yes — 4 clustered 2026-05 user corrections; 2026-07-10 resume-surface staleness | Mixed — the `/sync` ops-pulse IS a mechanism (`ops_pulse.py`, reporter-first) | **`keep-as-is`**, with a flagged gap: the "don't close out the chat mid-session" clause needs to fire *early*, and only does so via the `.claude/skills/**` glob | **MEDIUM** |
| 19 | `identity-before-work.md` | **uncond.** | 1,316 | n/a | **Yes — 2026-06-07 orphaned half-applied hook edits; lineage to the 2026-05-29 identity theft that corrupted a live peer's supra** | Narration (the `read_ppid_identity() → None` halt is a gate, but it's in the skill, not the rule) | **`keep-as-is`** — §2.3 disqualifies JIT: its whole value is blocking the FIRST side-effecting action, and JIT arrives after it | **MEDIUM** |
| 20 | `domain-guardrails.md` | **uncond.** | 3,336 | n/a | Yes — 5 named failures incl. the 97%-variance incident and stale-grid mixes | **Self-declared narration** — its own thesis is "invariants are gates, not prose" | **`keep-but-convert-to-gate`** — ship the 3 gate boundaries as raising validators in shared helpers, *then* the prose can shrink. `[blocked:human-decision]` (RATIFIED 2026-07-04) | **MED-LOW** |
| 21 | `novel-research-escalate-dont-default.md` | **uncond.** | **4,204** | n/a | **Yes — the most expensive incident in the corpus: DistantGapMPC vibecoded + trained across ~3 sessions / 4 days, then deleted at `cab305a`** | Hybrid — §"The Hard Gate" is a coordinator *obligation*, checked by no code | **`keep-but-convert-to-gate`**: a mechanical `grep '\[decided-by-human:' <spec>` precondition at dispatch. Only then reconsider the prose. `[blocked:human-decision]` (Hard Gate ratified 2026-06-14) | **MED-LOW** |
| 22 | `artifact-provenance.md` | **uncond.** | 3,664 | n/a | Yes — retired-baseline POI (3 sessions); GTFS with no sidecar; 11-min-stale `git_hash` | Reporters, explicitly "not gates" (`ccf27df`) | **`[blocked:human-decision]` — present, do not change.** Its always-load property was *itself* the ratified decision: "rules auto-load into every session's context; specs do not" `[decided-by-human:2026-07-03]` | **LOW (deliberately)** |
| 23 | `boundary-verdict-discipline.md` | **uncond.** | 3,210 | n/a | Yes — 2026-06-20 fern-slowing-cliff ("did you FULLY close out?" answered with a glossing yes) | Narration | **`demote-to-spec`** for the Luhmann/Levin theory + memory-organ sections only (~40% of file); keep both clauses, the vibe-check rule, anti-patterns. `[blocked:human-decision]` (RATIFIED 2026-07-04) | **LOW (deliberately)** |
| 24 | `poi-filter-discipline.md` | **uncond.** | 1,988 | n/a | **Yes — the retired-baseline POI that sailed through THREE verification passes and wasted the user's filter_v2 work** | Narration; explicitly de-fanged at 4 layers in the origin incident | **`add-paths-frontmatter`** is *technically* available (the rule writes its own globs in prose) — **but I rank it LAST and recommend against acting on it without the human.** See §5.24 | **LOWEST — deliberately last** |

**No rule receives a `propose-delete` recommendation.** Per acceptance criterion 4, this is stated
plainly rather than left implicit: after auditing all 24, **I found no rule where the evidence supports
deletion**. Two came close and are discussed in §6.

---

## 5. Per-rule justification (only where the table's one-liner is insufficient)

### 5.1 `plan-format.md` → `add-paths-frontmatter` — the single highest-value, lowest-risk move
Its own **second line** reads: *"Scoped to `.claude/plans/**`."* It is a domain-scoped rule that was
never given the frontmatter its own text describes. Three things make this safe where §2.3 would
normally bite: (a) the plan-writing consumers are `/valuate` and `/niche`, which load
`.claude/skills/{valuate,niche}/**` at session entry — **before** any plan is written, so including those
globs makes the rule arrive in time; (b) there is a PostToolUse reporter (`plan_format_check.py`,
matcher `Write|Edit`) as an independent backstop; (c) **no costed incident is encoded** — the failures it
prevents are formatting failures ("naked numbered steps with no OODA"), not a crash, a corruption, or a
lost artifact. Saves 2,807 tok/session. Fully reversible (delete four lines of frontmatter).

### 5.2 `coordinator-coordination.md` — a dead glob, and the corpus's cleanest gate-vs-narration case
**Dead glob, verified this thread**: the frontmatter lists `skills/**`, but `ls -d skills` returns
*"No such file or directory"* — there is no top-level `skills/` directory in this repo (the skills live at
`.claude/skills/`, already covered by the `.claude/**` glob). This is zero-risk to fix and costs nothing
either way; it is listed high because it is *certainly* correct, not because it is *important*.

**The gate case**: this rule's claim-squatting anti-pattern already has BOTH prose AND a stderr reporter
(`subagent-stop.py::check_claim_narrowing`, warns after 15 min). The ego's 2026-08-07 pass records
**5 passes recommending manual narrowing, done once**
(`.claude/scratchpad/ego/2026-08-07-russet-cooling-ledge.md:661,692`), with the explicit conclusion
*"Route it to the harness — derive from the HELLO-broadcast string at W0."* This is the audit's
central thesis with a clean control: prose failed, prose+reporter failed, and the proposed fix is
derivation (a mechanism). **Recommend deriving `claimed_paths` rather than instructing it.**

### 5.3 `multi-agent-protocol.md` — measured duplication with the priciest hook
This is the single most expensive rule (6,062 tok) and it fires on `.claude/**`, i.e. effectively always.
Its §"Scratchpad Format (MANDATORY)" is **restated nearly verbatim** by `subagent-context.py`'s
SubagentStart payload — I can confirm this first-person: my own dispatch received a "## Scratchpad
Protocol (auto-injected by SubagentStart hook)" block covering entry format, append-don't-overwrite,
prior-entries index, and reference-large-outputs-by-path. That content is being paid for **twice** on
every subagent dispatch: once in the 6,062-tok rule and again inside the 8,003-tok hook payload.

There is a second, sharper observation in the cartography (row 6) worth surfacing here: the hook's
`CONTEXT_LINES=100` truncation constant is **compensated for by this rule** (§Scratchpad Discipline
instructs agents to write summaries at the *bottom* so they survive the tail-cut). A rule written to
patch a selector's blind spot is evidence the selector, not the authors, is the thing to fix.

**Recommendation**: pick one source of truth for scratchpad format. My suggestion is the hook (it reaches
subagents unconditionally and cannot be missed), with the rule reduced to a pointer. `[blocked:human-decision]`
on which side survives — but the duplication itself is measured, not argued.

### 5.7 `multi-terminal-commit-discipline.md` — why the cheapest unconditional rule stays unconditional
794 tokens, the cheapest in the corpus. It cannot be path-scoped: its trigger is *"a `git commit` while a
peer terminal is live,"* and a git commit is a Bash invocation with **no file-path read that precedes it**.
There is no glob that fires here. It is unconditional-or-nothing, it is incident-backed (commit `4a20488`
swept a live peer lane's staged rename into the wrong lane's commit), and at 794 tokens it is 3.5% of the
unconditional rule budget. **Keep.**

### 5.8 `selflet-consumption-is-a-hard-gate.md` — the exemplar, cited as the model
The only rule in the corpus whose compliance is enforced by a **raising** mechanism:
`write_kapstok()` refuses to write and raises `SelfletGateError` unless
`selflet_cache_has_entry(identity, date, intent_slug, "all")` returns true. Its "empty vs absent"
distinction is a genuinely non-obvious design insight (the gate checks *did the agent run*, not *did it
find anything*). Note its own recorded lineage: it is described as the **third instance in ~9 days** of
the same "gate-not-guidance" fix. That is three independent confirmations that the mechanism/prose axis
is the real one. Its prose not firing this session cost nothing, because the gate does the work — which
is precisely the property to propagate.

### 5.14 `viz-discipline.md` → `narrow-paths` — first-person misfire evidence
This rule fired into **this dispatch** because I read files under `reports/**` while writing a harness
audit with zero figures in it. 2,271 tokens about raster stamp sizes, README chrome tolerance, and D62
baked-in figure titles, delivered to a thread that will never render a pixel. By §1.2's distractor
finding, that is not free.

The glob `reports/**` conflates two different things: report *prose* (`.md`, `.tex` bodies — mostly not
governed by this rule) and report *figures* (governed). Proposed narrowing: `reports/**/figures/**`,
`reports/**/*.tex`, `reports/**/*.png`, plus the existing `scripts/**` and `stage3_analysis/**` which
already catch every builder. Risk: a session editing a report's `.md` and choosing which figure to embed
would no longer get the chrome-calibration table. That is a real loss and I flag it — the human may
prefer the tax.

### 5.15 `reproduce-before-diagnose.md` → `add-paths-frontmatter`
The weakest incident backing in the unconditional set, and I say so plainly: its origin is an
**APMT-sibling** incident ported via `5f8ef34a`, with examples rescoped afterward. **No local costed
failure is recorded in it.** It is also the rule closest to default competence ("reproduce before you
diagnose" is standard debugging practice). But it is *not merely* default competence — the clause *"a
QAQC pass whose test data cannot exercise the fixed path is NOT a pass for that fix"* is a specific,
non-obvious enforcement requirement that a model would not reliably supply on its own. Recommendation is
conversion, not deletion, and §2.3 permits it: the rule's value is evaluative (it governs how you
investigate once you've opened the render script), not prohibitive-of-first-move.

### 5.16 `training-discipline.md` → `narrow-paths`
This rule encodes the **most physically expensive incident in the corpus** — a host OOM and PC crash from
6 simultaneously-dispatched background subagents (2026-05-15). Nothing about the content is in question.

The question is the `.claude/**` glob. It caused the rule to fire into the census dispatch (a
documentation/measurement thread) and would fire into any harness-meta session, none of which train
anything. The defensible counter-argument, which I want on record because it may be decisive: **the
serial-dispatch clause governs *dispatch*, and dispatch happens from `.claude/`-touching coordinator
sessions** — so `.claude/**` may be exactly right for that clause even though it is wrong for the rest of
the file. Hence the recommendation is a *split*, not a cut: serial-dispatch discipline moves to
`coordinator-coordination.md` (already `.claude/**`-scoped and already the owner of dispatch protocol),
and the remaining training-operational content (wandb, MSYS buffering, process-state diagnosis, autonomy
audit) narrows to `scripts/**` + `stage1_modalities/**` + `stage2_fusion/**`. **If that split is not
made, keep the glob** — losing the serial rule would risk re-opening a host crash, and that trade is not
worth 2,573 tokens.

### 5.19 `identity-before-work.md` — why I did NOT propose converting it
On token arithmetic this looks like an obvious JIT candidate (1,316 tok, scoped by its own text to
`/valuate`, `/niche`, `/sync`). **§2.3 kills it.** The rule's entire value is *"complete the identity
preamble before any side-effecting work."* Its origin incident is a session that ran `/valuate` and then
**immediately edited `.claude/hooks/librarian_selflet.py` before Step 0/Step 1**, leaving two of three
edits applied and one rejected mid-stream — an orphaned half-applied helper with no attribution trail. A
JIT rule matching `.claude/hooks/**` would have arrived *after* that first edit. The rule that must be
resident before the first action cannot be loaded by the first action. **Keep unconditional.**

Its lineage also reaches the 2026-05-29 identity-theft incident, which corrupted a *live peer's* supra —
a cross-terminal blast radius. That is a paid-for lesson I am not going near.

### 5.20 `domain-guardrails.md` — apply its own thesis to itself
This rule's title is *"Invariants Are Gates, Not Prose"* and its central claim is that
*"a rule that fires as narration an agent recites while proceeding stops nothing."* It is 3,336 tokens of
prose. That is not hypocrisy — it explicitly routes implementation to later engineering lanes — but it
does mean **the correct recommendation is to finish the routing, not to trim the prose**. Ship gates (i),
(ii), (iii) as raising validators in the shared loaders; once they raise, most of the prose becomes
documentation and can move to `specs/`. Doing it in the other order (trim prose, gates still unbuilt)
would remove the only thing currently carrying the invariant.

**`[blocked:human-decision]`** — RATIFIED `[decided-by-human:2026-07-04]`, and its own P2 clause requires
a recorded human nod for every reporter→gate promotion. I am not authorized to schedule those, only to
name them.

### 5.21 `novel-research-escalate-dont-default.md` — the biggest unconditional rule, and the biggest incident
4,204 tokens, the most expensive unconditional rule. It also encodes the most costly incident in the
corpus: DistantGapMPC was vibecoded and trained across **~3 sessions over 4 days**, found not to be MRPC
at all, and deleted at `cab305a`. The rule's own §Why contains the diagnosis that makes this audit's
thesis: *"the rule fired as narration, not as a gate"* — an agent cited it at 0.84 confidence while in the
same breath surfacing an agent-seeded next-step as a ready deliverable.

So its §"The Hard Gate" is a *coordinator obligation* that **no code checks**. The obvious mechanisation
is trivial: before any implementation/training wave on a novel method, `grep '\[decided-by-human:' <spec>`
and verify a tag on each foundational choice. That is a five-line reporter, promotable to a gate under
`domain-guardrails` P2. **Build the gate first; revisit the 4,204 tokens after.**

One thing I must not do here, and am flagging because the rule itself names it as an anti-pattern: an
agent applying `[decided-by-human:…]` is forging the signature the gate exists to require. Nothing in
this report applies, moves, or implies such a tag.

### 5.22 `artifact-provenance.md` — the always-load property IS the ratified decision
I want to be exact about why this is ranked low rather than high despite being 3,664 unconditional
tokens with globbable scope. The rule contains, in its own §"Relationship to the canonical specs":

> "**This rule SUMMARIZES the enforcement doctrine. `specs/artifact_provenance.md` +
> `specs/run_provenance.md` STAY CANONICAL for the full sidecar schema.** `[decided-by-human:2026-07-03]`
> The rule exists because **rules auto-load into every session's context; specs do not.** A provenance
> requirement that lives only in a spec is a requirement no cold-start session sees until it goes looking
> — the same consumption gap that let the retired-baseline POI ship."

That is a human, on the record, deciding that **unconditional injection is the mechanism** — after a
specific failure caused by the alternative. A `demote-to-spec` or `add-paths-frontmatter` recommendation
here would be proposing to undo a ratified decision by re-running the argument it already settled.
**Presented for the human's awareness; not recommended.**

### 5.24 `poi-filter-discipline.md` — deliberately ranked last
The technical case for conversion is real: the rule writes its own globs in prose (*"Fires on:
`stage1_modalities/poi/**`, POI artifacts under `data/study_areas/{area}/stage1_unimodal/poi/**`, POI
regen/queue scripts, and any stage2/stage3 code that loads a POI hex2vec parquet"*), and 1,988 tokens is
real money.

**Three reasons I rank it last and recommend against acting without the human:**

1. **Its own text forecloses the move.** It calls itself *"an artifact-invariant check — it must fire
   regardless of how the session frames its intent."* Path-matching is not intent-matching, so this is
   not fatal — but the rule was written by someone who had just watched framing-dependence fail.
2. **§2.3 half-bites.** The *consumer* side (read parquet → then check `filter_id`) survives JIT fine. The
   *producer* side does not: the origin incident's proximate cause was a kick script running bare
   `python -m stage1_modalities.poi --embedder hex2vec`, a Bash call with **no POI file read preceding
   it**. A POI-scoped glob would not have fired on the command that caused the failure.
3. **The cost was borne by the user personally.** Their filter_v2 work — built specifically to remove
   nature-reserve swathing — was thrown away, and three verification passes certified the wrong artifact
   because they measured coverage instead of labels. This is the exact profile of "a lesson someone
   already paid for."

If the human wants the 1,988 tokens back, the safe version is: convert **and** simultaneously land the
producer-side gate (a raising assert in `stage1_modalities/poi/__main__.py` that refuses to run without an
explicit filter selection). Prose-off without gate-on is the trade I will not recommend.

---

## 6. Step 4 — the human's hypothesis, answered two-sidedly

**The hypothesis, as offered for falsification**: *a gen-5 model routes around a lacking harness — it
parses whatever files it needs regardless of injection — so injection may pay a context tax for value it
does not deliver; and gen-5 is more easily OVERconstrained than under-constrained.*

### 6.1 Evidence FOR the hypothesis

1. **Substantive retrieval was 100% pull, 0% push.** The three most load-bearing sources for *this*
   thread — `.claude/ledger/digest/...census.json`, `CARTOGRAPHY.md`, and the 24 rule files themselves —
   were **not injected**. I found all of them by Read/Grep/Bash. The cartography's A′ section reports
   the identical pattern independently for its own three sources. Two dispatches, same finding: the
   ~35k floor bought orientation; it did not buy "here is the file that answers your question."
2. **The selflet pipeline is the strong form of the claim, and it was measured.** The ranking agent's own
   unprompted Grep/Read supplied 4 of 8 final carry-items, and the agent's own counterfactual on dropping
   the pushed pool was *"same, possibly marginally better."* **The model routing around the harness beat
   the harness**, on the harness's own retrieval task.
3. **Distractor tax is confirmed present, first-person.** `viz-discipline.md` (2,271 tok on raster stamps
   and README chrome) loaded into a zero-figure documentation thread. Chroma's finding — "even a single
   distractor reduces performance relative to the baseline" — makes that a cost, not neutral ballast.
4. **Instruction density degrades frontier models** — IFScale, 68% at 500 instructions, with primacy bias.
5. **Vendor guidance agrees in direction**: *"We do not recommend this"* about laundry-list rule stuffing;
   *"only add context agents don't already have."*

### 6.2 Evidence AGAINST the hypothesis

1. **The JIT rules that fired were actually followed, and I can show it in this thread's own conduct.**
   `multi-agent-protocol` (scratchpad format, prior-entries index, `[open|Nd]` tags),
   `multi-terminal-commit-discipline` (pathspec'd commit), `no-fallbacks`, `close-out-protocol` — all
   loaded and all obeyed. This is not the model routing around a lacking harness; it is a working S1
   selector delivering content the model then complied with.
2. **The hypothesis's mechanism claim fails on project-specific facts.** "The model parses whatever it
   needs" presupposes the model *knows what to look for*. Nothing in a gen-5 model's weights contains
   `filter_v2`, the BES-free canonical boundary, `region_id`-not-`h3_index`, or the fact that a bare
   `--embedder hex2vec` silently produces a retired baseline. **A model cannot route around not knowing a
   requirement exists.** That is precisely the consumption gap `artifact-provenance` was ratified to close.
3. **The strongest counter-evidence is the corpus's own failure record.** The rules that *failed* did not
   fail from overconstraint — they failed from **under-mechanisation**. Claim-narrowing: 5 passes, 1
   conversion, with prose AND a reporter present. Timestamp derivation: 6 passes, contradicted the same
   day. Those items were not ignored because the model was overloaded; they were ignored because nothing
   checked. Adding a gate fixed the equivalent case (`SelfletGateError`) *immediately and completely*.
   **If overconstraint were the binding problem, adding a hard constraint would have made things worse.
   It made them work.**
4. **Cutting rules would not have prevented a single incident in the corpus**, and would plausibly have
   caused several.
5. **The magnitude claim is unsupported.** §1.6: nobody has published whether ~24 policy documents
   degrades a gen-5 coding agent. IFScale's 500 orthogonal keyword constraints is not our regime.
   The hypothesis's *direction* has evidence; its *applicability at our scale* does not.

### 6.3 Verdict — partly true, in a specific way

**PARTLY CONFIRMED, and the qualification changes the recommended action.**

The hypothesis is **right about retrieval and wrong about governance.** For *substantive* retrieval
("which file has the fact I need"), push-injection contributed approximately nothing this session that
pull-search did not have to redo anyway — and in the selflet's case the push actively lost to the pull.
For *procedural governance* ("commit pathspec'd", "don't fall back", "write the scratchpad") the
injected rules were loaded and followed, and the model has no way to route around a project-specific
requirement it does not know exists.

The overconstraint half is **not supported by our local evidence**. Our measured failures are
under-mechanisation failures, not overload failures: 5 and 6 consecutive passes of correct prose changing
nothing, versus one hard gate changing behaviour immediately. The honest local claim is not "too many
rules" but **"too many rules in the wrong modality"** — narration where a check belongs.

Practical consequence: the highest-value change is **not** cutting the rule corpus (max safe saving
~4–6k tokens, 12–17% of floor). It is (a) converting 3–4 specific rules from prose to raising gates, and
(b) attacking the per-dispatch `subagent-context.py` payload, which at **≥ 9** logged spawns cost
**≥ ~72,027 tokens** today (a lower bound; the ledger started mid-session) — **more than twice the
entire unconditional floor** — and which grows linearly with every dispatch while the floor is
paid once.

---

## 7. The rule I was most tempted to propose deleting — and did not

**`boundary-verdict-discipline.md`** (3,210 tok, unconditional).

**Why I was tempted.** Roughly 40% of it is not operational instruction: a Luhmann section on operational
closure and second-order observation, a Levin section on level-N externalisation, and a "memory organ"
essay on why this repo sidesteps built-in auto-memory. Against §1.4's criterion — *only add context
agents don't already have* — its core clause ("ship the basis with the verdict; re-observe before
propagating") reads close to something a competent gen-5 model does by default. It is unconditional, it
is expensive, and its theory sections are the purest "distractor" shape in the whole corpus under
Chroma's finding. Every axis I built pointed at it.

**Why I did not.** Its **vibe-check clause** — *when the human asks "did you FULLY do X?", enumerate X's
checklist item-by-item and answer each explicitly; never compress to a yes* — is not default behaviour.
It is a correction to a **specific, dated, human-observed model failure** (2026-06-20 fern-slowing-cliff:
the human asked the precise question and still got an imprecise answer, because the model summarised one
checklist item away). And the web evidence I gathered predicts exactly that failure: IFScale's **bias
toward earlier instructions** and Chroma's degradation-under-length both say a compressed "yes" is what
comes out under load. **Deleting this rule would remove the single clause the published evidence says I
would most reliably fail.** That is close to the worst possible thing to cut.

Two further reasons that would have stopped me regardless: it is **human-ratified**
`[decided-by-human:2026-07-04]` after item-by-item in-chat review, and its own §Why records that the ego
which discovered the failure *deliberately declined to self-codify it* because a durable behavioural rule
is the human's to ratify. An agent deleting it unattended would be the same forge-the-signature
anti-pattern in reverse.

**What I propose instead** (§4 row 23): move the two theory sections to `specs/` where they remain
retrievable, keep both clauses, the vibe-check rule, the How-to-apply list and the anti-patterns. Est.
saving ~1,200 tok. **`[blocked:human-decision]`** — it is ratified, so even this trim is the human's call.

*Runner-up, for the record*: `reproduce-before-diagnose.md` — the only unconditional rule with **no local
costed incident** (ported from an APMT sibling) and the one closest to default debugging competence. I
recommend conversion (§4 row 15), not deletion, because its QAQC clause is a real non-default enforcement
requirement.

---

## 8. Acceptance-criteria check (enumerated, per `boundary-verdict-discipline` §producer clause)

| # | Criterion | Status | Basis |
|---|---|---|---|
| 1 | Report exists; **zero rule files modified or deleted** | ✅ | This file is the only report written. `.claude/rules/` was **read-only** throughout — every access was `Read`/`Bash cat`/`Bash head`. No `Edit`, no `Write`, no `rm` against any path under `.claude/rules/`. Stated explicitly in the banner and meant. |
| 2 | Web findings cited with URLs; thin/contested evidence flagged | ✅ | §1, 6 primary/authoritative sources with URLs. Thin flagged at §1.1 (IFScale regime mismatch), §1.2 (authors' own caveat), §1.3 (guidance ≠ measurement). Contested flagged at §1.5 (two open GitHub issues on `paths:` reliability). Unsettled enumerated at §1.6 with an explicit refusal to fabricate. |
| 3 | All 24 rules classified, per-rule justification, ordered by confidence | ✅ | §4 table, 24 rows, HIGH→LOWEST. §5 carries extended justification for the 12 rows where a one-liner was insufficient. |
| 4 | Every `propose-delete` names the incident it reopens — or states none is encoded | ✅ | **Zero `propose-delete` recommendations issued**; stated explicitly under §4. The two deletion temptations are named, priced, and declined in §7 with their incidents (2026-06-20 fern-slowing-cliff; APMT-ported, no local incident). |
| 5 | Hypothesis verdict is two-sided | ✅ | §6.1 five points FOR (incl. the selflet measurement where the model beat the harness), §6.2 five points AGAINST (incl. that our failures are under-mechanisation, not overload), §6.3 a qualified verdict that confirms one half and rejects the other. |

**Scope discipline**: this thread also refrained from acting on the two adjacent items it surfaced —
the `subagent-context.py` payload (Finding 2) and the `skills/**` dead glob (§5.2) — because both are
edits, and the mandate is *propose*. Both are handed to the harness lane.

---

*Written 2026-08-12, `russet-burning-river`, harness selection audit Thread C. All local figures lifted
from the census (`chars/4`-approx caveat preserved) or verified against source this thread; all external
claims carry a URL. **Zero rule files were modified or deleted.***

## Sources

- [IFScale — *How Many Instructions Can LLMs Follow At Once?* (arXiv:2507.11538)](https://arxiv.org/abs/2507.11538)
- [Chroma Research — *Context Rot: How Increasing Input Tokens Impacts LLM Performance*](https://www.trychroma.com/research/context-rot)
- [Anthropic Engineering — *Effective context engineering for AI agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic Engineering — *Writing effective tools for AI agents*](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Anthropic skill-authoring best practices (mirror)](https://github.com/obra/superpowers/blob/main/skills/writing-skills/anthropic-best-practices.md?plain=1)
- [TrueFoundry — *Just-in-Time Context for AI Agents: A Runtime Discipline*](https://www.truefoundry.com/blog/jit-context-just-in-time-context-agents)
- [Sourcegraph — *Context Engineering: A Practical Guide for AI Agents (2026)*](https://sourcegraph.com/blog/context-engineering)
- [CLAUDE.md Best Practices: The Complete 2026 Guide](https://dev.to/nishilbhave/claudemd-best-practices-the-complete-2026-guide-435j)
- [claude-code issue #17204 — `.claude/rules/` frontmatter format: `globs` works, `paths` does not](https://github.com/anthropics/claude-code/issues/17204)
- [claude-code issue #22170 — `paths` field in `~/.claude/rules/` frontmatter not loading](https://github.com/anthropics/claude-code/issues/22170)
