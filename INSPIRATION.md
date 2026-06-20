# Inspiration & lineage

Nothing in this kit is original. Everything here is a packaging of ideas from others, often combined in ways that turned out to work in production.

If your work is listed here and I've got the attribution wrong, or you'd like it removed, please open an issue.

---

## Direct influences

### Anthropic Claude Code Skills

The entire framework runs on Claude Code's [Skills system](https://docs.claude.com/en/docs/claude-code/skills). The `SKILL.md` format, the `references/` directory pattern, the Skill tool invocation semantics — all of that comes from Anthropic's design.

The foreman pattern in this kit is one specific way to use Skills: composing multiple skills into a sequential pipeline with verification gates between them. There are many other ways to use Skills that this kit doesn't cover (single-skill workflows, parallel skill fan-out, etc.).

### Anthropic — "Building Effective Agents"

Anthropic's [post on building effective agents](https://www.anthropic.com/engineering/building-effective-agents) describes several composition patterns: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer. The foreman pattern is essentially **orchestrator-workers + evaluator-optimizer chained sequentially**, with explicit verifier isolation rules.

If you haven't read that post, read it before reading this kit. It's the conceptual foundation.

### Cobus's loop-verifier

The principle **"verifier must be a fresh subagent, default stance: REJECT until proven otherwise"** comes from Cobus Greyling's writing on agent verifier patterns. Specifically the insight that **you cannot let the maker grade its own output** because the maker has already convinced itself the output is good (that's why it produced it).

This is iron law of verifier isolation: every checker in this kit spawns as a fresh subagent with a hardcoded read list, and the foreman cannot see the checker's reasoning before the verdict.

### Daniel Kahneman — System 1 / System 2

The framing of "the maker writes with System 1, the checker reads with System 2" is from Kahneman's *Thinking, Fast and Slow*. It's the psychological justification for why verifier isolation isn't paranoid — it's recognizing that the maker mode and checker mode are different cognitive tasks that need different "agents" running them.

This is also why the kit uses 3 fresh subagents for "taste voting" decisions: a single System 2 reader can be biased; three independent readers approximate a jury.

---

## Indirect influences

### Test-driven development (TDD)

The pattern **"define your gate before you build, then iterate against the gate"** is just TDD applied to LLM outputs. Your `critic-rules.yaml` is your test suite. Your maker is the implementation. The loop runs until tests pass.

This is why `critic-design-pattern.md` exists in `references/` — designing your critic rules well is the highest-leverage thing you can do to improve loop output quality.

### Continuous integration (CI)

The three-file state spine (`STATE.md` / `loop-budget.md` / `loop-run-log.md`) is borrowed from CI systems: pipeline state, budget caps, and a run log. Same idea — make the loop's state durable, auditable, and resumable, not ephemeral conversation context.

### Toyota Production System — andon cord

The 5 blocking categories in Iron Law 3 are an **andon cord** for the loop. Anything not in those 5 categories must be auto-resolved by the foreman. Anything in those 5 stops the line and pulls the human in.

This is the only way to keep the loop's promise of "zero interruption" without losing the ability to halt on real problems.

---

## What I added

To be honest about authorship:

- **The "foreman" framing** — packaging orchestrator-workers + evaluator-optimizer + verifier isolation into a single role with named responsibilities, written for Claude Code Skills specifically
- **The 5 iron laws as enforced rules** rather than guidelines (each one came from a specific failure I hit)
- **The Bash red line** — the observation that certain Bash commands trigger Claude Code permission prompts and break zero-interruption (I haven't seen this written up elsewhere)
- **The "taste voting" pattern with 3 fresh subagents** — voting subagents for subjective decisions where a single critic could be biased
- **Phase 6's "write Read + Edit, not Bash heredoc"** rule — domain-specific to Claude Code, easy to miss

None of these are big inventions. They're glue.

---

## What you should cite if you use this

If you fork the kit and ship something based on it, please:

1. **Cite Anthropic's "Building Effective Agents" post** — that's the conceptual foundation
2. **Cite Claude Code Skills documentation** — that's the runtime
3. **Cite Cobus's verifier work** — that's the verifier isolation principle
4. **Optionally cite this kit** if you find the foreman packaging useful

You don't need to cite me. Cite the people whose ideas are doing the heavy lifting.

---

## Open questions I haven't solved

These are things I tried but couldn't make clean. If you have ideas, open an issue.

1. **Cross-loop learning** — when loop N's critic finding could have been prevented by loop N-1's lesson, how do we propagate? Currently the kit writes "sediment suggestions" to the final report and asks the user to manually fold them back into critic-rules. That's the wrong end of the system.

2. **Resumable state after process crash** — STATE.md persists, but if the foreman crashes mid-phase, recovering "which phase was I in, what subagents were in flight" is manual. CI systems solved this 20 years ago, I haven't ported the pattern over yet.

3. **Multi-foreman coordination** — what if you want to run 3 loops in parallel on different topics? They'd all want to write to STATE.md. Currently the kit serializes (one `current_loop` at a time). A proper fix would need actual locking.

4. **Cost-aware iteration cap** — Iron Law allows 6 iterations / 2M tokens per topic, but those numbers are arbitrary. A smarter cap would adapt based on convergence rate (if rounds 1-3 are reducing critic findings monotonically, allow more; if they're oscillating, halt earlier).

If any of these resonate with work you're doing, I'd love to compare notes.
