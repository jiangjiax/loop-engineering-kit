---
name: email-loop
description: |
  Example loop using the loop-foreman scaffold. Drafts → tone check → spam check → final.
  This is the simplest possible loop (3 stages + 1 critic) to show how the foreman pattern works in practice.
  Trigger when the user says "write an email to X about Y" and wants the foreman pattern applied.
user_invocable: true
---

# email-loop · Worked example of loop-foreman

> **This is an example skill that demonstrates the loop-foreman pattern.**
> It's intentionally simple — three stages and one critic — so the pattern is easy to read.
>
> Read this alongside `loop-foreman/SKILL.md` to see how the scaffold gets filled in.

## What this loop does

| Phase | Action | Who runs it | Output |
|---|---|---|---|
| 0 | Init STATE | main thread | STATE.md current_loop block |
| 1 | Generate 3 draft emails (different tones: friendly / direct / formal) | main thread | 3 candidate drafts |
| 1.5 | Taste vote on which draft fits the user's voice | 3 fresh subagents | locked draft |
| 2 | Generate 2 tone variants (warm / professional) of the locked draft | main thread, serial dual | locked tone variant + step-5 taste vote |
| 3.5 | (Skipped — no polish stage in this minimal example) | — | — |
| 4 | Critic round 1 — tone check + spam check | fresh subagent | verdict + findings |
| 5 | Auto-fix issues found in critic-rules.yaml | main thread + fresh critic round 2 | iterations log |
| 6 | Archive draft to `out/[recipient]-[topic].md` | main thread | file written + log appended |
| 7 | Sediment (optional, often empty for short emails) | main thread | written to phase-8 report |
| 8 | Final report to user | main thread | 8-section summary |

**Phase 3 is skipped** in this minimal example because for emails we don't need a third maker stage (there's no "long body" beyond the tone variants). For richer domains (e.g., articles, scripts), Phase 3 generates the main body.

## When to use this example

1. **As a tutorial**: read this skill end-to-end to see how `loop-foreman` skeletons become a real loop
2. **As a starting point**: copy this skill, rename to your domain, replace the makers
3. **As a sanity check**: run this loop on a real email and verify the foreman pattern works in your Claude Code setup

## How to invoke

```
Run email-loop to draft an email to [recipient] about [topic], goal: [what you want them to do]
```

The foreman takes over. Expect a final report in 3-5 minutes.

## What you need before running

1. The 3 state files in your project root (`STATE.md` / `loop-budget.md` / `loop-run-log.md`). Copy from `loop-engineering-kit/templates/`.
2. Optionally: a `user-voice.md` file at your project root describing your communication preferences (taste-vote subagents will Read this). If not present, taste-vote will skip the persona-fit dimension.
3. The `references/tone-rules.yaml` and `references/spam-rules.yaml` files in this skill (already included).

**Note on file paths**: when subagents Read `references/tone-rules.yaml` etc., they need an **absolute path**. The foreman passes the absolute path to the subagent's prompt — never a relative path, since subagent CWD is not guaranteed to match the skill directory or project root. Example: pass `/Users/you/.claude/skills/email-loop/references/tone-rules.yaml`, not `references/tone-rules.yaml`.

## Phase-by-phase walk-through

This skill follows the templates in `loop-foreman/references/phase-templates.md` exactly, with the following placeholders filled in:

- `[MAKER_1]` = generate 3 draft emails (different tones)
- `[MAKER_2]` = generate 2 tone variants of the locked draft
- `[MAKER_3]` = (skipped)
- `[POLISHER]` = (skipped)
- `[CHECKER]` = tone + spam critic
- `[CRITIC_RULES_PATH]` = `references/tone-rules.yaml` + `references/spam-rules.yaml`

### Phase 0 · Init

Read STATE.md, loop-budget.md, loop-run-log.md from project root. Write `current_loop` block with:
- `selection: email-to-[recipient]-about-[topic]`
- `started_at: [ISO timestamp]`
- `status: phase_1_drafting`

### Phase 1 · Draft 3 versions (main thread)

> **Iron Law 5 inline exception for minimal examples**: when the maker logic is small enough to inline (a few hundred tokens of prompting) AND no domain-specific reference files exist, the foreman may inline the maker logic in the main thread instead of invoking a separate Skill. This is the **only such exception** in this example. Production loops with non-trivial makers should always invoke a real Skill per Iron Law 5.

Generate 3 drafts **sequentially in one turn** (not actual parallelism — the main thread is single-threaded), varying only tone. Each draft uses a different tone:

- **V1 friendly**: warm opener, casual close
- **V2 direct**: single-paragraph, no padding
- **V3 formal**: structured, professional close

Save to `out/drafts/[recipient]-v{1,2,3}.md` using the Write tool.

### Phase 1.5 · Taste vote

> Read the full pattern at `loop-foreman/references/taste-subagent.md` before spawning. The summary below specializes the pattern for email-loop.

Spawn 3 fresh `general-purpose` subagents in parallel using the Agent tool. Each subagent receives this hardcoded prompt:

```
You are a Phase 1.5 taste vote subagent for an email draft selection. You are NOT a writer.
You are a taste evaluator — pick which of 3 drafts best fits the user's voice.

Required reading (Read in order):
1. [project-root]/user-voice.md (if exists — describes user's communication style)
2. out/drafts/[recipient]-v1.md (friendly)
3. out/drafts/[recipient]-v2.md (direct)
4. out/drafts/[recipient]-v3.md (formal)
5. references/tone-rules.yaml (for context only — don't evaluate against these here, critic will)

Score each draft on 3 dimensions (1-10, decimals allowed):
1. persona_fit: alignment with user-voice.md (if exists). If user-voice.md is missing, skip this dimension and set persona_fit: null.
2. goal_fit: alignment with the user's stated goal for this email
3. recipient_appropriateness: does the tone match the inferred recipient relationship

Default stance: when scores are < 2 points apart, lean toward the version with highest goal_fit
(getting the recipient to act > sounding nice).

Return strict JSON:
{
  "vote": "v1" | "v2" | "v3",
  "scores": {
    "v1": {"persona_fit": X.X | null, "goal_fit": X.X, "recipient_appropriateness": X.X, "total": X.X},
    "v2": {...},
    "v3": {...}
  },
  "reason": "1-3 sentences, quote a specific phrase from the chosen draft"
}

Final message MUST be the JSON only, no prose around it.
```

**Foreman vote-tally rules**:
- **3 votes unanimous** → adopt, write `picked_by: taste_unanimous` to STATE.md
- **2:1 majority** → adopt majority, minority + reason goes to Phase 8 report
- **1:1:1 split (tie-breaker)** → pick the version with the highest summed `total` across all 3 subagents. If still tied (extremely rare), pick V2 (direct) as the default — direct emails have the lowest stylistic risk.

After tally, copy the winning draft to `out/[recipient]-locked-v1.md` using the Read + Write tools (not `cp` — you want to optionally normalize formatting during the copy).

**Note on `user-voice.md`**: this file is optional. If you frequently write emails in a recognizable style, create a file at your project root describing it (1-2 paragraphs is enough). Example content:
> "I prefer short, direct emails. I rarely use exclamation marks. I sign off with just my first name. I avoid corporate-speak like 'circle back' or 'touch base'."

If `user-voice.md` doesn't exist, the taste-vote will still work — subagents just skip the `persona_fit` dimension.

### Phase 2 · Generate 2 tone variants of locked (main thread, serial)

> **Same Iron Law 5 inline exception as Phase 1** — for this minimal example, the maker logic is small enough to inline. Production loops should invoke a real Skill.

Generate two refined versions **sequentially in one turn**:

- **Warm variant**: locked structure + warmer phrasing
- **Professional variant**: locked structure + tighter, more professional phrasing

Save to `out/[recipient]-warm.md` and `out/[recipient]-professional.md` using the Write tool.

**Foreman synthesizes** by picking the version that better matches the user's stated goal:
- Goal contains "build relationship" / "thank" / "celebrate" → pick warm
- Goal contains "request" / "approve" / "decide" / "follow up" → pick professional
- Goal contains both or neither → pick professional (lower stylistic risk)

Log the goal classification to STATE.md `current_loop.phase_2_goal_classification` before synthesizing. Output the chosen version to `out/[recipient]-synthesis.md`.

**Step-5 taste vote**: spawn 3 fresh subagents using the same template as Phase 1.5, but swap the read paths to:
- `out/[recipient]-locked-v1.md` (the locked v1 from Phase 1.5, for reference)
- `out/[recipient]-warm.md` (candidate A)
- `out/[recipient]-professional.md` (candidate B)
- `out/[recipient]-synthesis.md` (candidate C, the foreman's synthesis)

Subagent votes on which of A/B/C best serves the goal. 2:1 against synthesis (C) = roll back to the majority-voted original (A or B).

### Phase 4 · Critic round 1 (fresh subagent)

Spawn fresh `general-purpose` subagent with this hardcoded prompt:

```
You are the critic for an email draft.

Required reading:
1. references/tone-rules.yaml — tone violations and their tiers
2. references/spam-rules.yaml — spam pattern violations
3. out/[recipient]-synthesis.md — the draft to review

Default stance: REJECT until proven otherwise.

For each finding, return one of these tiers:
- fatal: would damage the sender's reputation (e.g., aggressive tone, fake urgency)
- serious: significantly weakens the email (e.g., bury the ask, apologetic close)
- general: style polish issue (e.g., filler phrase, weak verb)
- escalate: requires sender judgment (e.g., is this recipient familiar enough for casual greeting?)

Verdict: APPROVE | APPROVE_WITH_NOTES | REJECT_LIGHT | REJECT | ESCALATE_HUMAN
```

### Phase 5 · Routing

Per `loop-foreman/references/phase-templates.md` Phase 5 routing matrix.

For each finding:
- **A bucket** (gate file has answer) → foreman uses Edit tool to apply the rule's `auto_fix` action, citing the rule's `example.good` field as the precedent
- **B bucket** (context inferable from goal/recipient) → foreman Edits based on the user's stated goal
- **C bucket** (truly unanswerable, needs sender judgment) → write to Phase 8 report, don't try to fix

#### Worked routing example

Say Phase 4 critic returns 3 findings on the Alex/Q4 budget draft:

```json
{
  "verdict": "REJECT_LIGHT",
  "findings": {
    "fatal": [],
    "serious": [
      {
        "rule_id": "ask_must_be_in_first_paragraph",
        "tier": "serious",
        "location": "paragraph 2",
        "issue": "The ask 'Could you make 30 min Thursday?' is in paragraph 2 after background context",
        "auto_fix_available": true
      }
    ],
    "general": [
      {
        "rule_id": "avoid_corporate_padding",
        "tier": "general",
        "location": "paragraph 1, sentence 1",
        "issue": "Starts with 'I hope this email finds you well'",
        "auto_fix_available": true
      }
    ],
    "escalate": [
      {
        "issue": "Is 'Alex' the right level of casualness for the sender's relationship with this recipient?",
        "tier": "escalate"
      }
    ]
  }
}
```

Foreman routing:

| Finding | Bucket | Action |
|---|---|---|
| `ask_must_be_in_first_paragraph` (serious) | A | tone-rules.yaml has the rule + `example.good` showing how to lead with the ask. Foreman uses Edit to restructure paragraph 1 — move the meeting request to sentence 1, push the Q4 context to sentence 2. |
| `avoid_corporate_padding` (general) | A | tone-rules.yaml `auto_fix.action: remove_phrase`. Foreman uses Edit to delete the opener. |
| Recipient relationship escalate | C | Foreman cannot judge sender↔recipient familiarity from the draft alone. Log to Phase 8 report as "user check before sending: confirm 'Alex' is appropriate vs 'Alexandra' or formal salutation." |

After auto-fixing the A/B items, spawn Phase 5 critic round 2 (fresh subagent, NOT reuse of round 1) to verify the fixes didn't introduce new findings.

Max 3 iteration rounds. **Round 4 fallback for email-loop**: if the critic still rejects, the user's input prompt (goal + recipient + topic) is probably ambiguous — write to Phase 8 report: "after 3 iterations, suggest user clarifies: [the specific ambiguity, e.g., 'is this a request for approval or for discussion?']." For longer-form domains (articles, scripts), this fallback is "roll back to outline stage."

### Phase 6 · Archive

`cp out/[recipient]-synthesis.md out/[recipient]-final.md`

Read loop-run-log.md → Edit to append:
```json
{
  "run_id": "[ISO timestamp]",
  "skill": "email-loop",
  "selection": "email-to-[recipient]-about-[topic]",
  "phases_completed": ["state_init", "draft_3", "taste_phase_1_5", "tone_variants", "critic_round1", "archive"],
  "iterations": [N],
  "final_verdict": "APPROVE_WITH_NOTES",
  "draft_word_count": [N],
  "duration_minutes": [estimate]
}
```

Clear `current_loop` block in STATE.md (via Read + Edit, not sed).

### Phase 7 · Sediment

For email-loop, sediment is usually empty (emails are short, rules cover most cases). But scan auto-fixes:
- Any fix that didn't have a critic-rules source → suggest adding the new pattern + example to `tone-rules.yaml` or `spam-rules.yaml`
- Any phrasing the user explicitly approved during taste vote → suggest adding to an approved-phrasings list (create `approved-phrasings.md` in project root if it doesn't exist)

**Output location**: write all sediment suggestions inline in the Phase 8 report (the section "🌱 Sediment suggestions"). Do NOT modify `tone-rules.yaml` / `spam-rules.yaml` directly — the user reviews suggestions in the Phase 8 report and approves them later via:

> "Apply sediment B1 and skip B2"

When the user approves, the foreman then uses Edit to update the gate files. If the user is silent for 24h+ on the same suggestion across multiple loops, the Phase 8 report adds a "repeated suggestion not adopted — batch decision needed" line.

### Phase 8 · Final report

```markdown
# Email loop report · to [recipient] about [topic]

**Time**: [N] min  **Iterations**: [N]  **Final verdict**: [verdict]

## ✅ Final email
Path: out/[recipient]-final.md
Word count: [N]
Tone: [warm / professional]
Ask placement: [first sentence / first paragraph]

## 🪑 Unadopted candidates
- V1 (friendly draft): [why taste-vote rejected]
- V2 (direct draft): [why taste-vote rejected]
- Warm variant: [why synthesis preferred professional]

## 🟡 Notes for user (check before sending)
[C-bucket items, if any]

## 🌱 Sediment suggestions
[Any auto-fix patterns that should be added to tone-rules.yaml]

## 📌 Next steps
- Review out/[recipient]-final.md
- Send when ready
```

---

## How this minimal example demonstrates the foreman pattern

Even with just 2 maker stages and 1 critic, you get:

- **Verifier isolation**: critic and taste-vote subagents are fresh, the maker can't grade its own draft
- **Multi-vote convergence**: 3 taste-vote subagents reduce single-evaluator bias
- **Auto-fix loop**: critic findings auto-resolve through tone-rules + spam-rules without asking the user
- **Resumable state**: STATE.md tracks where you are, loop-run-log appends every run
- **Zero interruption** (when working correctly): user invokes once, gets a finished email

Scale this up to 4-5 maker stages + 1 critic + 1 polisher for richer domains (articles, scripts, code reviews, etc.). The pattern is the same.

---

## How to extend this example

To add a "main body" stage for longer-form content:

1. Add Phase 3 from `loop-foreman/references/phase-templates.md` (conservative + sharp dual version)
2. Add a Phase 3 cross-review (2 fresh subagents reviewing each other's version)
3. Add Phase 3 step-5 taste vote (3 subagents on synthesis vs conservative vs sharp)

To add a "polish" stage:

1. Write a polisher skill (e.g., "remove AI smell")
2. Add Phase 3.5 with the polisher skill name
3. Update the foreman to invoke it via Skill tool after Phase 3 and before Phase 4 critic

The full template is in `loop-foreman/references/phase-templates.md`.
