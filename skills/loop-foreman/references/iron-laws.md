# The 5 Iron Laws + Bash red line (enforcement reference)

This file is the authoritative source for the rules referenced from `SKILL.md`. Read this whenever a phase transition involves user-facing output or routing decisions.

---

## Iron Law 1 · ESCALATE is not a delegation excuse

When the checker (critic) returns `ESCALATE_HUMAN`, the foreman's instinct is to forward it. **Don't.**

Most "escalate" findings have answers in:
- A documented `example` in your critic-rules.yaml
- A rule in your style-guide
- A note in the input brief

Before escalating, the foreman **must search the gate files for a matching example**. Only the following truly need human:

1. Gate files + outline + memory all silent on this issue
2. The fix requires muscle memory the LLM can't simulate (voice, taste, embodied knowledge)
3. The human has explicitly said "this category always goes to me"

For categories 2 and 3, write the finding into the final report as a note ("worth checking when recording / shipping") — **don't block the loop**.

### Self-check before escalating

For each finding the critic flagged as `ESCALATE_HUMAN`:

```
□ Did I Grep [critic-rules-path] for matching example? Y/N
□ Did I Grep [style-guide-path] for matching rule? Y/N
□ Did I Grep [outline-path] for matching note? Y/N
```

If any of the three is N, do that grep before deciding whether to escalate.

---

## Iron Law 2 · The gate files ARE the verification gate

Your critic-rules.yaml, your style guide, your "don't say these words" list — these aren't documentation. They're the **gate the foreman must pass every output through**.

Before routing any critic finding, the foreman **must Read every readable gate file**, auto-resolve everything answerable from them, and only escalate the truly unanswerable.

### What "every readable gate file" means

Depending on your domain, this might be:

- `critic-rules.yaml` (or `.json` / `.toml` — whatever you use)
- A style guide
- A list of forbidden words / phrases / patterns
- A list of "approved phrasings" the user has previously locked
- The outline / brief / spec being executed against
- Any project memory file (e.g., `MEMORY.md`)

Define this list once per phase, hardcode it into the routing logic.

---

## Iron Law 3 · Inside the loop, do NOT ask the user (except 5 blocking categories)

**This is why the loop exists.**

### Allowed (5 blocking categories, exhaustive)

1. **Outline / brief missing** — can't find the spec the loop is supposed to execute
2. **State conflict** — another loop is already running in `current_loop`
3. **Infrastructure missing** — STATE.md / loop-budget.md / loop-run-log.md don't exist
4. **Budget exceeded 100%** — would burn past today's token cap
5. **Iteration cap hit** — same topic looped N times (default 6) without convergence

### Forbidden (must auto-resolve)

- ❌ "V1 or V2 or V3?" → run taste voting (see `taste-subagent.md`)
- ❌ "Should I save this learning?" → decide yes/no, write the suggestion into Phase 8 report, let the user batch-review
- ❌ "Critic returned REJECT — should I roll back to outline?" → follow Phase 5 routing matrix in `phase-templates.md`
- ❌ "Should I re-run?" → follow Phase 5 routing matrix
- ❌ "Should I use the locked v1 cover or re-run?" → read the outline contract + the user's pre-flight instructions, auto-decide
- ❌ "Is this word count OK?" → check the gate files

### Self-check before any 4-line phase report

Before writing the phase summary, scan:

```
□ Does the message contain "?" or "？"?
□ Does it end with "you choose" / "you decide" / "should I"?
□ Behavior check: does the next response action invoke the next phase's tool? If no, that's an implicit "say continue" question = violation.
```

If any answer is wrong:
1. Delete the question
2. Auto-resolve per the foreman routing rules
3. Log the decision to STATE.md
4. Write the rationale into Phase 8 report for user review later

### Iron Law 3.1 · Skill-tool continuity (added v4.3)

After invoking a Skill tool, the same response **must immediately invoke the next phase's first tool call** (Bash find / Agent spawn / Read / Edit / next Skill). Stopping after a 4-line report creates an implicit "say continue" prompt = Iron Law 3 violation.

**Test**: Does the assistant's last message in this turn end with a tool call?
- Yes → correct
- No → violation

---

## Iron Law 4 · Don't skip phases via "v1 already locked"

If v1 of a stage passed verification, the instinct is to skip the verification on v2 because "the upstream change is small." **Don't.**

| Source of next-version artifact | Foreman action |
|---|---|
| Outline contract explicitly says "stage X locked, skip Phase Y" | Skip Phase Y, copy contract into STATE.md, **still run taste vote at step 5** |
| Inherited from a previous v1 / v2 run | **DO NOT skip** — re-verify, mark `re_verified: true` in STATE.md |
| User pasted a seed assertion | Run Phase normally, verify the assertion is preserved + meets gate criteria |
| No prior artifact | Run full phase |

Mark `re_verified: true` in STATE.md so you can audit later. The verification cost is small. The regression cost is large.

---

## Iron Law 5 · Skill tool ≠ Read SKILL.md

**Skill tool invocation** = the skill's template **takes over** the current turn. Rules have enforcement power.

**Read SKILL.md as a file** = the main thread treats it as documentation. The thread will instinctively cut corners.

Every maker phase requires three steps, none skippable:

1. **Bash find** to list reference files in the skill directory
2. **Skill tool invocation** (NOT a Read of SKILL.md)
3. **Foreman Reads all references** (every file in the listing except SKILL.md, no cherry-picking, no prejudgment)

In STATE.md, log each maker phase:
```
phase_N_skill_invocation:
  skill_invoked: true/false
  invocation_method: Skill_tool / Read_only  # Read_only = violation
  references_read: [list of paths]
```

`Read_only = violation`. Phase 0 of the next loop scans for this and halts.

---

## Bash command red line (added v4.4)

Independent of the 5 iron laws, certain Bash patterns **trigger Claude Code permission prompts**, which break the zero-interruption promise of the loop. Forbidden in all phases.

### Forbidden patterns

| Pattern | Used for | Use instead | Why |
|---|---|---|---|
| `cat >> file << 'EOF' ... EOF` | Append to log file | Read + Edit (append) | heredoc triggers prompt |
| `sed -i 's/X/Y/' file` | In-place edit | Edit tool | sed -i triggers prompt |
| `sed -n 'M,Np' \| grep ...` | Range-then-search | Grep tool with `path` + `multiline: true` | piped sed/grep triggers prompt |
| `awk '...' file` | Statistical processing | Read then process in main thread / Grep tool | awk triggers prompt |
| `find . -name "*.md" -exec ...` | Batch operation | Glob tool + main thread loop | find -exec triggers prompt |
| `wc -m file` | Character count | Read + main thread count | sometimes triggers prompt |

### Allowed patterns (no prompt)

- `find <path> -type f` — list files (used before spawning subagents to hardcode read list)
- `cp src dst` — file copy (used in Phase 6 archive)
- `date -u +"..."` — timestamp (used in Phase 0 and Phase 6)
- `ls -la <path>` — directory listing

### Heuristic

If a Bash command contains:
- a pipe (`|`)
- a redirect (`>>` `>` `<<`)
- a stream editor (`sed -i` `awk`)

99% chance it triggers a permission prompt. Use a native Claude Code tool (Edit, Read, Write, Grep, Glob) instead.

---

## Why these rules and not others

The five iron laws cluster around one principle:

> **The foreman is a router and a state-keeper, not a maker and not a critic.**

Every time a rule gets violated, what's happening underneath is one of these drift directions:

| Drift | Symptom | Fix |
|---|---|---|
| Foreman wants to make | "I'll just write it myself" | Maker should be in main thread but invoked via Skill tool, not by foreman impersonation |
| Foreman wants to critique | "Looks fine to me" | Critic must be a fresh subagent |
| Foreman wants to delegate decisions | "Let me ask the user" | Auto-resolve using gate files + taste-vote subagents |

If you find yourself adding a 6th iron law, ask: which of these three drifts is it preventing? If none, it's probably a phase-specific rule, not an iron law. Add it to `phase-templates.md` instead.
