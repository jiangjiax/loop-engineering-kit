---
name: code-review-loop
description: |
  Example loop using the loop-foreman scaffold. Generates code review comments from
  3 perspectives (security / correctness / style) in parallel, cross-reviews them,
  synthesizes, critics for tone + technical accuracy, archives.

  Demonstrates the foreman pattern for NON-TEXT, multi-perspective decision-making work.
  This is the closest thing to a "full" production loop in the examples directory.

  Trigger when the user asks "review this diff" or "review PR #X" and wants the
  multi-perspective verifier pattern applied.
user_invocable: true
---

# code-review-loop · Multi-perspective code review with verifier isolation

> **This is a worked example.** It demonstrates how `loop-foreman` applies to non-text-shaped work — specifically, multi-perspective judgment tasks (code review, design critique, decision support, anything where multiple expert viewpoints need to be reconciled).
>
> Read this alongside `../../skills/loop-foreman/SKILL.md` to see how the scaffold gets filled in.

## Why code review is a great fit for the foreman pattern

A good code review typically requires **at least 3 distinct expert perspectives**:

- 🔒 **Security perspective** — does this introduce new attack surface?
- ✅ **Correctness perspective** — does this actually do what it claims?
- 🎨 **Style perspective** — does this fit the team's idioms and maintainability standards?

A single reviewer rarely holds all three simultaneously at full depth. **What you want is 3 independent reviewers + a synthesizer.** That's exactly the foreman pattern with `parallel makers` + `cross-review` + `synthesis` + `critic`.

The kit's verifier-isolation principle solves a real bug in single-reviewer code review: a reviewer who's already convinced "this PR is fine" will rubber-stamp it; you need someone who **didn't see the PR yet** to challenge each finding.

## What this loop does

| Phase | Action | Who runs it | Output |
|---|---|---|---|
| 0 | Init STATE | main thread | STATE.md current_loop block |
| 1 | Generate 3 review drafts in parallel: security / correctness / style | 3 fresh subagents (parallel) | 3 review drafts |
| 1.5 | Taste vote: which draft caught the most real issues? | 3 fresh subagents | locked draft + ranking |
| 2 | Generate 2 tone variants of the merged review (collaborative / direct) | main thread, serial dual | 2 variants + step-5 taste vote |
| 3 | Skipped — no third maker stage needed for reviews | — | — |
| 3.5 | Polish: remove condescension, soften without weakening | main thread | polished review |
| 4 | Critic round 1: tone-rules + technical-rules | fresh subagent | verdict + findings |
| 5 | Auto-fix issues from review-rules.yaml | main thread + fresh critic round 2 | iterations log |
| 6 | Archive review to `out/reviews/[PR-id].md` | main thread | file + log appended |
| 7 | Sediment patterns into review-rules.yaml | main thread | written to phase-8 report |
| 8 | Final report with action items for the PR author | main thread | structured summary |

**Phase 1 uses parallel subagents** (not sequential makers like `email-loop`) because the 3 perspectives are genuinely independent — security review shouldn't see correctness's analysis or vice versa, otherwise you lose perspective diversity.

## When to use this example

1. **As a tutorial**: read end-to-end to see how `loop-foreman` extends from text to judgment work
2. **As a starting point**: copy this skill, replace the 3 perspectives with your domain's expert viewpoints
3. **As a pattern reference**: anytime you need "3 independent experts + synthesizer" — design critique, research methodology review, business proposal evaluation

## How to invoke

```
Run code-review-loop on the diff at [path-to-diff-or-PR]
```

Or for a remote PR:

```
Run code-review-loop on https://github.com/owner/repo/pull/482
```

The foreman fetches the diff, runs all phases, archives the review.

## What you need before running

1. The 3 state files in your project root (`STATE.md` / `loop-budget.md` / `loop-run-log.md`). Copy from `loop-engineering-kit/templates/`.
2. **The diff to review** — either a local file (`.diff` / `.patch`) or a fetchable URL.
3. Optionally: `team-conventions.md` at your project root (style guide, naming rules, architectural decisions). If present, the style-perspective reviewer reads it.
4. The `references/review-rules.yaml` and `references/technical-rules.yaml` files in this skill (included).

**Note on file paths**: when subagents Read reference YAMLs, the foreman passes them **absolute paths**. Subagent CWD is not guaranteed.

## Phase-by-phase walk-through

This skill follows `../../skills/loop-foreman/references/phase-templates.md` with the following placeholders filled in:

- `[MAKER_1]` = parallel review drafting (3 perspectives, 3 fresh subagents)
- `[MAKER_2]` = tone variant generation (main thread, serial)
- `[MAKER_3]` = (skipped — no third maker stage)
- `[POLISHER]` = condescension remover (main thread, inline)
- `[CHECKER]` = review critic
- `[CRITIC_RULES_PATH]` = `references/review-rules.yaml` + `references/technical-rules.yaml`

### Phase 0 · Init

Read STATE.md, loop-budget.md, loop-run-log.md from project root. Write `current_loop` block:
- `selection: code-review-for-[PR-id-or-file]`
- `started_at: [ISO timestamp]`
- `status: phase_1_parallel_review`

Fetch the diff:
- Local file → Read tool
- URL → WebFetch (or `gh pr diff [number]` if `gh` CLI is available)

Save the diff to `out/reviews/[PR-id]-source.diff`.

### Phase 1 · Generate 3 review drafts in parallel (3 fresh subagents)

**This is the key architectural choice for code review**: makers run in **parallel fresh subagents**, not the main thread.

Why? Each perspective needs to read the diff **without seeing the other perspectives' analysis**. Otherwise:
- Security might miss things if it sees correctness already covered them
- Style might rubber-stamp if security already said "no major issues"

Parallel + isolated = each perspective produces its strongest independent finding list.

Spawn 3 fresh `general-purpose` subagents in parallel using the Agent tool:

**Security reviewer prompt**:
```
You are a security-focused code reviewer.

Required reading:
1. [absolute-path]/out/reviews/[PR-id]-source.diff

Your job: identify security risks in this diff. Look for:
- Input validation gaps
- Auth/authz changes (does this expand or weaken trust boundaries?)
- New external dependencies
- Secret/credential handling
- SQL/command injection vectors
- Race conditions in security-relevant paths

Output strict JSON:
{
  "findings": [
    {
      "severity": "fatal" | "serious" | "general",
      "location": "file:line",
      "issue": "1 sentence describing the risk",
      "evidence": "quote the specific code that creates the risk",
      "suggested_fix": "1-2 sentences",
      "confidence": "high" | "medium" | "low"
    }
  ],
  "overall_security_verdict": "BLOCKING" | "NEEDS_DISCUSSION" | "OK"
}

Default stance: REJECT until proven otherwise. If you're not sure, mark as
"NEEDS_DISCUSSION" with low confidence — don't APPROVE by default.
```

**Correctness reviewer prompt** (same structure, different focus):
```
Your job: identify correctness bugs in this diff. Look for:
- Off-by-one errors
- Null/undefined handling
- Logic that doesn't match the PR description
- Edge cases the diff doesn't handle
- Test coverage gaps for new behavior
- Concurrency issues
```

**Style reviewer prompt** (with team-conventions.md if available):
```
Required reading:
1. [absolute-path]/out/reviews/[PR-id]-source.diff
2. [absolute-path]/team-conventions.md (if exists — your team's style guide)

Your job: identify style and maintainability issues. Look for:
- Deviations from team-conventions.md (if exists)
- Naming inconsistencies with surrounding code
- Premature abstraction / over-engineering
- Missing or unclear comments on non-obvious decisions
- Test naming and structure consistency
```

Save each draft to `out/reviews/[PR-id]-security.md`, `out/reviews/[PR-id]-correctness.md`, `out/reviews/[PR-id]-style.md`.

**STATE.md write**:
```yaml
phase_1_parallel_review:
  spawned_via: Agent_tool_3_fresh_subagents_parallel
  perspectives: [security, correctness, style]
  drafts:
    security:
      path: out/reviews/[PR-id]-security.md
      finding_count: N
      verdict: BLOCKING | NEEDS_DISCUSSION | OK
    correctness: {...}
    style: {...}
status: phase_1_5_taste_vote_in_progress
```

### Phase 1.5 · Taste vote — which review caught the most real issues?

Unlike email-loop where taste vote picks "best fit for user voice," code review taste vote answers a harder question: **which of the 3 reviews surfaced findings most likely to be real bugs?**

Spawn 3 fresh `general-purpose` subagents in parallel:

```
You are a Phase 1.5 taste vote subagent for code review selection. You are NOT a reviewer yourself.
Your job: rank the 3 perspective reviews by which most likely caught real issues.

Required reading (in order):
1. [absolute-path]/out/reviews/[PR-id]-source.diff (read the actual code first)
2. [absolute-path]/out/reviews/[PR-id]-security.md
3. [absolute-path]/out/reviews/[PR-id]-correctness.md
4. [absolute-path]/out/reviews/[PR-id]-style.md

Score each review on 3 dimensions (1-10):
1. specificity: are findings tied to specific code, with quoted evidence?
2. real_bug_likelihood: how many findings look like real bugs vs. theoretical concerns?
3. action_clarity: can the PR author act on this without asking questions?

Default stance: when scores are < 2 points apart, lean toward whichever has the
highest specificity (vague findings waste author time more than missing findings
get caught downstream).

Return strict JSON:
{
  "ranking": ["security", "correctness", "style"],  // best to worst
  "scores": {
    "security": {"specificity": X.X, "real_bug_likelihood": X.X, "action_clarity": X.X, "total": X.X},
    ...
  },
  "reason": "1-3 sentences, must quote a specific finding from the top-ranked review"
}
```

**Foreman vote-tally rules** (slightly different from email-loop):
- All 3 agents agree on #1 → strong signal, foreman uses that as primary review
- 2:1 majority on #1 → adopt majority, note runner-up as plan B
- 1:1:1 split → foreman uses **all 3 reviews merged** (this is unique to code review — when perspectives genuinely disagree on importance, the right answer is to surface all findings, not pick one)

Write the foreman-locked review base to `out/reviews/[PR-id]-locked-v1.md`. This is the **merged finding list**, deduplicated:
- Same finding appearing in multiple perspectives → consolidate to 1 entry
- Conflicting verdicts (e.g., security says BLOCKING, correctness says OK on the same line) → flag as "PERSPECTIVES_DISAGREE" for the synthesizer

### Phase 2 · Tone variants (main thread, serial)

> **Iron Law 5 inline exception**: tone shaping is a transformation, not a generator. Inline is correct here for this minimal example.

Generate 2 tone variants of the locked review **sequentially in one turn**:

- **Collaborative variant**: framed as "I noticed X, what do you think?" / "Could we consider Y?"
- **Direct variant**: framed as "X is a bug, fix by doing Y" / "Z violates team-conventions"

Save to `out/reviews/[PR-id]-collaborative.md` and `out/reviews/[PR-id]-direct.md`.

**Foreman synthesizes** by picking based on:
- Junior author (≤1 year experience) or first-PR → collaborative
- Senior author or hot fix or security-critical → direct
- Unknown → direct (better to be specific than ambiguous; you can always soften in conversation)

Log the classification to STATE.md `phase_2_audience_classification`.

**Step-5 taste vote**: spawn 3 fresh subagents using the same pattern as Phase 1.5, but with collaborative / direct / synthesis as candidates. Score on:
- `psychological_safety`: does this tone make the author defensive or open?
- `action_clarity`: can the author act without translation?
- `professional_appropriateness`: does this match how senior engineers talk?

2:1 against synthesis = roll back.

### Phase 3.5 · Polish: remove condescension

> Inline polisher, runs in main thread.

Scan the locked review for condescension markers:
- "obviously" / "clearly" / "simply" / "just"
- "you should know that..."
- "this is basic..."
- Sarcasm / passive-aggression

Use Edit tool to remove or rephrase. Don't soften the technical content — only neutralize the tone.

Save to `out/reviews/[PR-id]-polished.md`.

### Phase 4 · Critic round 1 (fresh subagent)

Spawn fresh `general-purpose` subagent:

```
You are the critic for a code review draft.

Required reading:
1. [absolute-path]/examples/code-review-loop/references/review-rules.yaml — review tone rules
2. [absolute-path]/examples/code-review-loop/references/technical-rules.yaml — technical accuracy rules
3. [absolute-path]/out/reviews/[PR-id]-polished.md — the review to critique
4. [absolute-path]/out/reviews/[PR-id]-source.diff — the original code

Default stance: REJECT until proven otherwise.

For each finding in the review, ask:
- Is it specific (cites file:line + quoted code)?
- Is it actionable (suggests a fix)?
- Is it accurate (does the quoted code actually do what the finding says)?
- Is the tone professional (no condescension, no sarcasm)?
- Is the severity calibrated (not crying wolf, not under-flagging)?

Tiers:
- fatal: review accuses the code of a bug that doesn't exist (false positive) — these damage trust irreversibly
- serious: review misses a real bug visible in the diff (false negative) — these get caught later but cost time
- general: tone or specificity issues that don't change correctness
- escalate: requires team/project context the critic can't infer

Verdict: APPROVE | APPROVE_WITH_NOTES | REJECT_LIGHT | REJECT | ESCALATE_HUMAN
```

### Phase 5 · Routing

#### Worked routing example

Say Phase 4 critic returns 3 findings:

```json
{
  "verdict": "REJECT_LIGHT",
  "findings": {
    "fatal": [
      {
        "rule_id": "false_positive_security_claim",
        "tier": "fatal",
        "location": "review line 23",
        "issue": "Review claims auth.ts:45 introduces SQL injection, but the quoted code uses parameterized queries — the claim is wrong",
        "auto_fix_available": false,
        "evidence_from_diff": "db.query('SELECT * FROM users WHERE id = ?', [userId])"
      }
    ],
    "serious": [
      {
        "rule_id": "missing_severity_calibration",
        "tier": "serious",
        "location": "review line 47",
        "issue": "Style finding marked as BLOCKING but it's a naming preference — should be NEEDS_DISCUSSION"
      }
    ],
    "general": [
      {
        "rule_id": "condescension_marker",
        "tier": "general",
        "location": "review line 12",
        "issue": "Uses 'obviously this is wrong' — condescension marker"
      }
    ]
  }
}
```

Foreman routing:

| Finding | Bucket | Action |
|---|---|---|
| `false_positive_security_claim` (fatal) | **C** (truly unanswerable) | Foreman cannot decide whether the original Phase 1 security reviewer was right or the Phase 4 critic is right — both have specific code citations. **Escalate**: write to Phase 8 report as "BLOCKER: Phase 1 security finding contested by Phase 4 critic, both cite specific evidence. User must adjudicate." |
| `missing_severity_calibration` (serious) | A | review-rules.yaml has the rule. Foreman uses Edit to downgrade the finding from BLOCKING to NEEDS_DISCUSSION. |
| `condescension_marker` (general) | A | review-rules.yaml has the rule + example.good. Foreman uses Edit to remove "obviously". |

After auto-fixing A items, spawn Phase 5 critic round 2 (fresh subagent) to verify fixes.

**Round 4 fallback for code-review-loop**: if critic still rejects after 3 rounds, the review is fundamentally miscalibrated — write to Phase 8 report: "after 3 iterations, suggest user re-runs Phase 1 with different perspective prompts, or escalates to a human reviewer."

### Phase 6 · Archive

Use Bash `cp` to copy `out/reviews/[PR-id]-polished.md` → `out/reviews/[PR-id]-final.md`.

Read loop-run-log.md → Edit to append JSON entry:
```json
{
  "run_id": "[ISO timestamp]",
  "skill": "code-review-loop",
  "selection": "code-review-for-[PR-id]",
  "phases_completed": ["state_init", "parallel_review_3", "taste_phase_1_5", "tone_variants", "polish", "critic_round1", "archive"],
  "iterations": [N],
  "final_verdict": "APPROVE_WITH_NOTES",
  "findings_total": [N],
  "blocking_findings": [N],
  "duration_minutes": [estimate]
}
```

Clear `current_loop` block in STATE.md via Read + Edit.

### Phase 7 · Sediment

For code-review-loop, sediment is **higher value than email-loop** because reviews repeat patterns:
- Same false-positive type caught multiple times → add to review-rules.yaml as "common false positive in domain X"
- Same condescension phrasing flagged → add to ban list
- Same severity miscalibration → tighten the calibration guidelines

Write sediment suggestions to Phase 8 report. Don't modify rules files directly — user approves later.

### Phase 8 · Final report

```markdown
# Code review loop report · PR [id]

**Time**: [N] min  **Iterations**: [N]  **Final verdict**: [verdict]

## ✅ Final review
Path: out/reviews/[PR-id]-final.md
Findings: [N total] / [N blocking] / [N needs-discussion] / [N suggestions]

## 🔍 Perspective breakdown (Phase 1)
- Security: [verdict + top finding summary]
- Correctness: [verdict + top finding summary]
- Style: [verdict + top finding summary]

## 🪑 Unadopted variants
- Collaborative tone variant: [why direct preferred / vice versa]
- Phase 1 runner-up perspective: [which perspective's findings didn't make the final cut, why]

## 🟡 Contested findings (need user adjudication before posting to PR)
[C-bucket items where Phase 4 critic disputes Phase 1 findings]

## 🌱 Sediment suggestions
[Patterns observed this run that should be added to review-rules.yaml or technical-rules.yaml]

## 📌 Action items for PR author
[The actual finding list, prioritized by severity, ready to copy to PR comment]
```

---

## How this example shows what the simple example couldn't

Compared to `email-loop`:

| Aspect | email-loop | code-review-loop |
|---|---|---|
| Maker pattern | Inline sequential (3 tones) | **Parallel fresh subagents** (3 perspectives) |
| Why parallel? | Tone variants are aesthetic | Perspectives must be **mutually unaware** to preserve independence |
| Taste vote criterion | Best fit for user voice | **Most likely real bugs** + **specificity** |
| Polish stage | Skipped | **Required** (condescension removal) |
| Critic tiers | tone violations | **False positives are fatal** (worse than missed bugs) |
| Round 4 fallback | "User clarifies goal" | "Re-run with different perspective prompts" |
| Sediment value | Low (emails are unique) | **High** (review patterns repeat) |

The same 9-phase scaffold, calibrated for **judgment work** instead of **text production**.

## How to adapt this example to your domain

This example is the closest the kit has to a **production-grade loop**. To adapt:

1. **Replace the 3 perspectives** with your domain's expert viewpoints:
   - Design critique: visual hierarchy / accessibility / brand consistency
   - Research methodology: prior art / experimental design / statistical validity
   - Business proposal: market fit / competitive moat / execution feasibility
2. **Replace review-rules.yaml** with your tone calibration for that domain
3. **Replace technical-rules.yaml** with your domain's correctness check
4. **Tune the round-4 fallback** to your domain's "what does it mean if 3 iterations don't converge"
5. Optionally: add a Phase 3 if your domain has a "main body" beyond the tone variants (e.g., for research proposals, Phase 3 could be the detailed methodology section)
