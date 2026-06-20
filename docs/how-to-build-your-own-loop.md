# How to build your own loop

This doc walks through building a custom loop from the `loop-foreman` scaffold, using a small new example: a **code-review loop**.

By the end you'll have a working skill that:
1. Drafts code review comments in 2 perspectives
2. Cross-reviews them
3. Synthesizes via taste vote
4. Critics for tone + correctness
5. Outputs final review

The walkthrough mirrors the structure of the `_example_email_loop` skill — read that first if you haven't.

---

## Step 1 · Decide what your makers and checker are

The foreman pattern needs you to identify three things:

1. **Stages where content is produced** (the makers)
2. **The single point where you verify the produced content** (the checker)
3. **What your rule book looks like** (the critic-rules file)

For a code-review loop, a natural decomposition:

| Stage | Maker / Checker | Output |
|---|---|---|
| 1 | maker: review-draft (multi-perspective) | 3 review drafts: security-focused / correctness-focused / style-focused |
| 2 | maker: review-merge | foreman picks the 1 best draft, then generates 1 polished version |
| 3 | (skip — no third maker stage needed for code reviews) | — |
| 4 | checker: review-critic | verdict on review tone (avoid harshness) + technical accuracy |

So you have 2 maker stages, 1 critic. That fits the scaffold cleanly.

---

## Step 2 · Set up the skill directory

```bash
mkdir -p ~/.claude/skills/code-review-loop/references
cd ~/.claude/skills/code-review-loop
```

You'll create these files (filling in your domain):

```
code-review-loop/
├── SKILL.md           # the loop foreman, your skill
└── references/
    ├── review-rules.yaml      # tone rules for reviews
    └── technical-rules.yaml   # correctness checks
```

---

## Step 3 · Copy & adapt loop-foreman/SKILL.md

Open `loop-engineering-kit/skills/loop-foreman/SKILL.md` and copy its structure to your new `SKILL.md`. Make these substitutions:

| In scaffold | Replace with |
|---|---|
| `[MAKER_1]` | `review-draft` (your stage-1 skill or inline logic) |
| `[MAKER_2]` | `review-merge` (your stage-2 skill or inline logic) |
| `[MAKER_3]` | — (skipped for this domain) |
| `[POLISHER]` | — (skipped) |
| `[CHECKER]` | `review-critic` (your critic skill) |
| `[CRITIC_RULES_PATH]` | `references/review-rules.yaml` + `references/technical-rules.yaml` |
| `[topic name]` | `code-review-for-[file/PR]` |

Also adjust the phase table: since you have 2 maker stages, **delete Phase 3 and its step-5 taste vote section**.

---

## Step 4 · Write your critic rules

This is the highest-leverage step. Spend at least 30 minutes on it before running the loop.

Use `loop-foreman/references/critic-design-pattern.md` as your design guide.

For our code-review loop, start with maybe 15 rules covering:

- **Tone (general)**: avoid "you should have done X" (passive-aggressive)
- **Tone (general)**: avoid bare imperatives without explanation
- **Tone (serious)**: never personally attack ("this is bad code")
- **Tone (fatal)**: never reveal frustration ("I've told you this before")
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
Run code-review-loop on file: src/auth.ts (PR #482)
```

The foreman starts at Phase 0, reads STATE.md, then walks through phases.

### What you should see (zero-interruption loop)

- Phase 0 report: STATE initialized
- Phase 1 report: 3 review drafts generated
- Phase 1.5: 3 subagents vote, 1 draft selected
- Phase 2 report: refined version generated, step-5 taste vote runs
- Phase 4 report: critic flagged X findings
- Phase 5 report: N auto-fixes applied, critic round 2 verdict
- Phase 6 report: review file archived
- Phase 7-8: final report with summary + suggestions

### Total tool calls: ~30 / Total time: 5-15 minutes / User interruptions: 0

If you see permission prompts mid-loop, check `references/bash-no-go-zone.md` — you probably used a forbidden Bash pattern.

---

## Step 7 · Sediment learnings

After your first few loops, look at the Phase 8 reports' "🌱 sediment suggestions" section.

For each suggestion, ask:
- Is this a new rule I should add to `review-rules.yaml`?
- Is this a phrasing I want to add to my approved-phrasings pool?

Within 2-3 weeks, your critic-rules will cover 80%+ of the failures the foreman catches, and ESCALATE rate should drop below 10%.

---

## Common adaptations by domain

### Article-writing loop (3 makers)

Like `_example_email_loop` but with a Phase 3 main-body stage. Use the scaffold's Phase 3 template (conservative + sharp dual version + cross-review + step-5 taste vote).

### Slide-deck loop

- Phase 1: outline structure (3 candidates with different narrative arcs)
- Phase 2: per-slide title + key bullet generation
- Phase 3: visual recommendations per slide
- Phase 4: critic for narrative consistency + slide density

### Research-summary loop

- Phase 1: extract key claims (3 candidates with different levels of skepticism)
- Phase 2: counter-claim generation
- Phase 3: synthesis paragraph
- Phase 4: critic for source attribution + claim/evidence balance

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
