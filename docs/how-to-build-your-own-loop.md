# How to build your own loop

This doc walks through building a custom loop from the `loop-foreman` scaffold, using a hypothetical domain: **academic paper summarization with citation verification**.

By the end you'll have a working skill that:
1. Drafts paper summaries from 3 perspectives (claim-focused / methodology-focused / impact-focused)
2. Cross-reviews them for source attribution
3. Synthesizes via taste vote
4. Critics for citation accuracy + claim-evidence balance
5. Outputs final summary

> **Note**: this is a *thought experiment* walkthrough, not a runnable skill. Two runnable examples ship in `examples/`:
> - `examples/email-loop/` — simplest end-to-end loop, text production
> - `examples/code-review-loop/` — multi-perspective judgment work
>
> Read whichever is closest to your domain, then come back here for the meta-pattern.

---

## Step 1 · Decide what your makers and checker are

The foreman pattern needs you to identify three things:

1. **Stages where output is produced** (the makers)
2. **The single point where you verify the produced output** (the checker)
3. **What your rule book looks like** (the critic-rules file)

For a paper-summary loop, a natural decomposition:

| Stage | Maker / Checker | Output |
|---|---|---|
| 1 | maker: extract-claims (multi-perspective) | 3 candidate summaries: claim-focused / methodology-focused / impact-focused |
| 2 | maker: synthesize-summary | foreman picks the strongest, then generates a polished single-paragraph version |
| 3 | (skip — no third maker stage needed for summaries) | — |
| 4 | checker: citation-and-balance-critic | verdict on citation accuracy + claim-evidence balance |

So you have 2 maker stages, 1 critic. That fits the scaffold cleanly.

---

## Step 2 · Set up the skill directory

```bash
mkdir -p ~/.claude/skills/paper-summary-loop/references
cd ~/.claude/skills/paper-summary-loop
```

You'll create these files (filling in your domain):

```
paper-summary-loop/
├── SKILL.md           # the loop foreman, your skill
└── references/
    ├── balance-rules.yaml      # claim-evidence balance rules
    └── citation-rules.yaml     # source attribution checks
```

---

## Step 3 · Copy & adapt loop-foreman/SKILL.md

Open `loop-engineering-kit/skills/loop-foreman/SKILL.md` and copy its structure to your new `SKILL.md`. Make these substitutions:

| In scaffold | Replace with |
|---|---|
| `[MAKER_1]` | `extract-claims` (your stage-1 skill or inline logic) |
| `[MAKER_2]` | `synthesize-summary` (your stage-2 skill or inline logic) |
| `[MAKER_3]` | — (skipped for this domain) |
| `[POLISHER]` | — (skipped) |
| `[CHECKER]` | `citation-and-balance-critic` (your critic skill) |
| `[CRITIC_RULES_PATH]` | `references/balance-rules.yaml` + `references/citation-rules.yaml` |
| `[topic name]` | `paper-summary-for-[paper-id]` |

Also adjust the phase table: since you have 2 maker stages, **delete Phase 3 and its step-5 taste vote section**.

---

## Step 4 · Write your critic rules

This is the highest-leverage step. Spend at least 30 minutes on it before running the loop.

Use `../skills/loop-foreman/references/critic-design-pattern.md` as your design guide.

For our paper-summary loop, start with maybe 15 rules covering:

- **Citation (fatal)**: claim attributed to author X but author X didn't make that claim
- **Citation (fatal)**: claim presented as paper's conclusion when it's actually a hypothesis
- **Balance (serious)**: presents methodology without limitations
- **Balance (serious)**: extrapolates the paper's claim beyond its stated scope
- **Correctness (serious)**: claim of "this is a bug" needs evidence
- **Correctness (general)**: praise without specifics is filler
- **Structure (general)**: each comment needs at least one of {observation, question, suggestion}
- **etc.**

Each rule needs the canonical fields: `rule_id`, `tier`, `description`, `trigger`, `auto_fix`, `example.bad`, `example.good`.

The `example.bad/good` pair is the most important field — it's what saves the foreman from over-escalating per Iron Law 1.

---

## Step 5 · Copy state templates to project root

```bash
cp ~/.claude/skills/loop-engineering-kit/templates/*.template /path/to/your/project/
cd /path/to/your/project
mv STATE.md.template STATE.md
mv loop-budget.md.template loop-budget.md
mv loop-run-log.md.template loop-run-log.md
```

---

## Step 6 · Run the loop

```
Run paper-summary-loop on paper: arxiv.org/abs/2024.12345
```

The foreman starts at Phase 0, reads STATE.md, then walks through phases.

### What you should see (zero-interruption loop)

- Phase 0 report: STATE initialized
- Phase 1 report: 3 candidate summaries generated
- Phase 1.5: 3 subagents vote, 1 summary selected
- Phase 2 report: synthesized version generated, step-5 taste vote runs
- Phase 4 report: critic flagged X findings
- Phase 5 report: N auto-fixes applied, critic round 2 verdict
- Phase 6 report: summary archived
- Phase 7-8: final report with summary + suggestions

### Total tool calls: ~30 / Total time: 5-15 minutes / User interruptions: 0

If you see permission prompts mid-loop, check `references/bash-no-go-zone.md` — you probably used a forbidden Bash pattern.

---

## Step 7 · Sediment learnings

After your first few loops, look at the Phase 8 reports' "🌱 sediment suggestions" section.

For each suggestion, ask:
- Is this a new rule I should add to `balance-rules.yaml` or `citation-rules.yaml`?
- Is this a phrasing I want to add to my approved-phrasings pool?

Within 2-3 weeks, your critic-rules will cover 80%+ of the failures the foreman catches, and ESCALATE rate should drop below 10%.

---

## Common adaptations by domain

### Article-writing loop (3 makers)

Like `examples/email-loop/` but with a Phase 3 main-body stage. Use the scaffold's Phase 3 template (conservative + sharp dual version + cross-review + step-5 taste vote).

### Slide-deck loop

- Phase 1: outline structure (3 candidates with different narrative arcs)
- Phase 2: per-slide title + key bullet generation
- Phase 3: visual recommendations per slide
- Phase 4: critic for narrative consistency + slide density

### Code review loop

Already shipped as `examples/code-review-loop/`. Multi-perspective in parallel (security / correctness / style) + cross-review + tone polish. The closest the kit has to a "full" production loop.

### Decision support loop

- Phase 1: proposals from N perspectives (optimist / pessimist / realist) in parallel
- Phase 2: structured debate (each perspective steel-mans the others)
- Phase 3: synthesis with explicit trade-off analysis
- Phase 4: critic for steel-man quality + missing considerations

The pattern is the same. The domain content is what you write.

---

## When you've outgrown this scaffold

The scaffold is good for sequential pipelines with 2-5 maker stages. Signs you need something else:

- **Parallel maker stages**: if your "makers" don't depend on each other (e.g., generating subtitles + audio + visual descriptions all from the same input), use parallel agents, not a sequential loop
- **Long-running workflows that span days**: STATE.md doesn't have task scheduling. You'd want to integrate with a real workflow engine
- **Multi-foreman coordination**: 3 loops needing to coordinate on shared state would need real locking. STATE.md serializes (one `current_loop` at a time)

For those cases, the scaffold can still be a building block, but you'd compose it differently. The `INSPIRATION.md` doc lists some references for those richer patterns.

---

## Glossary recap

| Term | Meaning |
|---|---|
| **Foreman** | The orchestrator skill that routes between phases. Stays in main thread. |
| **Maker** | A content-producing skill (drafting, generating, writing). Runs in main thread. |
| **Checker** | A verification-only skill that grades output. Runs in fresh subagent. |
| **Taste vote** | 3 fresh subagents voting on subjective synthesis decisions. |
| **Cross-review** | 2 fresh subagents reviewing each other's work in dual-version stages. |
| **Critic-rules** | The YAML/JSON/Markdown file that defines what counts as a violation and how to fix it. The highest-leverage file in your loop. |
| **Gate file** | Any file the foreman or critic reads to determine "is this OK or not." Critic-rules is one. Style guide is another. |
| **Sediment** | Phase 7 action where the foreman writes suggestions to grow the gate files based on this run's lessons. |
| **Iron Law** | A non-negotiable rule that prevents a known failure mode. |
| **Bash red line** | Bash patterns that trigger Claude Code permission prompts and break zero-interruption. |
