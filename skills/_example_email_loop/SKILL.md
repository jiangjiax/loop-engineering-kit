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
2. Optionally: a `user-voice.md` file describing your communication preferences (taste-vote subagents will Read this). If not present, taste-vote will skip the persona-fit dimension.
3. The `references/tone-rules.yaml` and `references/spam-rules.yaml` files in this skill (already included).

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

Generate 3 drafts in parallel using inline reasoning (no separate Skill tool call needed for this minimal example — the maker logic is small enough to inline). Each draft uses a different tone:

- V1: friendly — warm opener, casual close
- V2: direct — single-paragraph, no padding
- V3: formal — structured, professional close

Save to `out/drafts/[recipient]-v{1,2,3}.md`.

### Phase 1.5 · Taste vote

Spawn 3 fresh `general-purpose` subagents in parallel. Each Reads:
- `user-voice.md` (if present)
- The 3 draft files
- `references/tone-rules.yaml` (for context, not for evaluation)

Each subagent returns JSON:
```json
{
  "vote": "v1" | "v2" | "v3",
  "scores": { ... },
  "reason": "..."
}
```

Foreman tallies. Majority adopts. Write locked draft to `out/[recipient]-locked-v1.md`.

### Phase 2 · Generate 2 tone variants of locked (main thread, serial)

Generate two refined versions:

- **Warm variant**: locked structure + warmer phrasing
- **Professional variant**: locked structure + tighter, more professional phrasing

Save to `out/[recipient]-warm.md` and `out/[recipient]-professional.md`.

**Foreman synthesizes** by picking the version that better matches the inferred goal (warm if goal is "build relationship", professional if goal is "request a decision"). Output to `out/[recipient]-synthesis.md`.

**Step-5 taste vote**: 3 fresh subagents vote on synthesis vs warm vs professional. 2:1 against synthesis = roll back.

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
- **A bucket** (gate file has answer) → foreman Edits draft per the rule's auto-fix
- **B bucket** (context inferable) → foreman Edits based on outline/goal
- **C bucket** (truly unanswerable) → write to Phase 8 report

Max 3 iteration rounds. Round 4 = escalate as "outline issue" (in email's case: the goal might be ambiguous; foreman writes the goal-ambiguity to the report).

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
- Any fix that didn't have a critic-rules source → suggest adding example to tone-rules.yaml
- Any phrasing the user explicitly approved during taste vote → suggest adding to an approved phrasing pool

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
