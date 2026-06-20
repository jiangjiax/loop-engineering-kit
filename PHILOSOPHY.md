# The 5 Iron Laws

These aren't suggestions. Each one is a failure mode someone (often me) hit in production, fixed, and locked behind a rule. Skip any of them and your loop will regress in a way that's hard to debug because the failure looks like a "weird LLM moment."

---

## Iron Law 1 · ESCALATE is not a delegation excuse

When your checker (critic) raises `ESCALATE_HUMAN`, the foreman's instinct is to forward it to the user. **Don't.**

Most "escalate" findings already have an answer in the gate files:
- A documented `example` in your critic-rules.yaml
- A rule in your `style-guide.md`
- A note in the input brief

Before escalating, the foreman **must search the gate files for a matching example**. Only the following truly need the human:

1. Gate files + outline + memory all silent on this issue
2. The fix requires muscle memory the LLM can't simulate (voice, taste, embodied knowledge)
3. The human has explicitly said "this category always goes to me"

For categories 2 and 3, write the finding into the final report as "suggest checking when recording / shipping" — **don't block the loop**.

---

## Iron Law 2 · The gate files ARE the verification gate

Your critic-rules.yaml, your style guide, your "don't say these words" list — these aren't documentation. They're the **gate the foreman must pass every output through**.

Before routing any critic finding, the foreman **must Read every readable gate file**, auto-resolve everything answerable from them, and only escalate the truly unanswerable.

If the foreman skips this and routes "I think this needs human input" directly, you're paying for a critic but throwing away its work.

---

## Iron Law 3 · Inside the loop, do NOT ask the user (except 5 blocking categories)

**This is why the loop exists.** The user invoked the foreman to make decisions *for them*, then look at the final result. **Every "yes or no?" question undoes the loop's value.**

### Allowed (5 blocking categories, exhaustive)

1. **Outline / brief missing** — can't find the spec the loop is supposed to execute
2. **State conflict** — another loop is already running in `current_loop`
3. **Infrastructure missing** — STATE.md / loop-budget.md / loop-run-log.md don't exist
4. **Budget exceeded 100%** — would burn past today's token cap
5. **Iteration cap hit** — same topic looped 6× without convergence

### Forbidden (every "do I auto-resolve this?" question)

- ❌ "V1 or V2?" → run taste voting subagents
- ❌ "Should I save this?" → decide, write to the report, let the user batch-review later
- ❌ "Should I re-run?" → follow the phase 5 routing matrix
- ❌ "Is this OK?" → check the gate files

### Self-check (before any 4-line phase report)

Before writing the phase summary, ask:
- Does this contain `?` or `？`?
- Does it end with "you choose" / "you decide" / "should I"?
- Is the next action a tool call (correct) or a stop (violation)?

If any answer is wrong, delete the question, auto-resolve, log the decision to STATE.md.

### Iron Law 3.1 · Skill-tool continuity

After invoking a Skill, **the same response must immediately invoke the next phase's tool call**. The Skill output format tends to assimilate the main thread into "done — wait for user." If you stop after a 4-line report, you've created an implicit "say continue" question = Iron Law 3 violation.

**Test**: Does the assistant's last message end with a tool call? Yes = correct. No = violation.

---

## Iron Law 4 · Don't skip phases via "v1 already locked, inheriting"

If v1 of a stage was good, your instinct is to skip the verification on v2 because "it's the same." **Don't.**

The risk: v1 was verified against v1's content. v2's content might differ subtly in ways that break v1's lock. The verification cost is small. The cost of shipping a regression is large.

Always re-verify, even if "obviously fine." Mark `re_verified: true` in STATE so you can audit later.

---

## Iron Law 5 · Skill tool ≠ Read SKILL.md

**Skill tool invocation** = the skill's template **takes over** the current turn. Rules have enforcement power.

**Read SKILL.md as a file** = the main thread treats it as documentation. The thread will instinctively cut corners.

Every maker phase requires three steps, none skippable:

1. `Bash find` to list reference files in the skill directory
2. **Skill tool invocation** (NOT a Read of SKILL.md)
3. Foreman Reads all references (every file in the listing except SKILL.md, no cherry-picking)

In STATE.md, every maker phase logs `invocation_method: Skill_tool / Read_only`. `Read_only = violation`. The next loop's Phase 0 scans for this and halts.

---

## Bash command red line (added v4.4)

Independent of the 5 iron laws, certain Bash patterns **trigger Claude Code permission prompts**, which break the zero-interruption promise of the loop. Forbidden in all phases:

| Forbidden Bash pattern | Use instead | Why |
|---|---|---|
| `cat >> file << 'EOF' ... EOF` | Read + Edit (append) | heredoc triggers prompt |
| `sed -i 's/X/Y/' file` | Edit tool | sed -i triggers prompt |
| `sed -n '32,148p' \| grep ...` | Grep tool with `path` and `multiline` | pipe sed/grep triggers prompt |
| `awk '...' file` | Read then process in main thread / Grep tool | awk triggers prompt |
| `find . -name "*.md" -exec ...` | Glob tool + main thread loop | find -exec triggers prompt |

**Heuristic**: if a Bash command contains a pipe (`|`), redirect (`>>` `>` `<<`), or stream editor (`sed -i` `awk`), 99% chance it triggers a permission prompt. Use a native tool.

---

## Why these rules and not others

The five iron laws cluster around one principle: **the foreman is a router and a state-keeper, not a maker and not a critic**. Every time a rule gets violated, what's happening underneath is one of these:

- Foreman wants to make (instinct: "I'll just write it myself") → maker should be in main thread, but as a Skill invocation, not the foreman impersonating
- Foreman wants to critique (instinct: "looks fine to me") → critic must be a fresh subagent
- Foreman wants to delegate decisions (instinct: "let me ask the user") → that's what auto-resolve is for

If you find yourself adding a 6th iron law, ask: which of these three drift directions is it preventing? If none, it's probably a phase-specific rule, not an iron law.
