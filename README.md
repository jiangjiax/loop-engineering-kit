# loop-engineering-kit

> A scaffold for building **multi-stage Claude Code agent loops** with verifier isolation, taste voting, and zero-interruption execution.

English README (this file) · [中文版](./README.zh.md)

A skill scaffold for Claude Code. Drop it into `~/.claude/skills/` or your project's `.claude/skills/`, fill in your `[MAKER]`s and `[CHECKER]`, and you get a loop that:

- **Produces → verifies → iterates → archives** without asking the user mid-flight
- **Isolates verifiers in fresh subagents** so the producer can't grade its own output
- **Routes checker findings through gate files** (your own rule book) before escalating to the human
- **Persists state across phases** in three append-only files (`STATE.md` / `loop-budget.md` / `loop-run-log.md`)

This is not a framework. It's a **set of patterns and templates** you copy and adapt to your domain.

The pattern is **domain-agnostic**. The same 9-phase structure applies to:

- **Text production** — writing, editing, copywriting, translation, summarization
- **Judgment work** — code review, design critique, research evaluation, decision support
- **Information synthesis** — research summaries, market analysis, multi-source reports
- **Structured artifacts** — plans, proposals, schemas, configurations

See [`examples/`](./examples/) for runnable loops across different domains.

---

## Why agent loops need a foreman

Most multi-step LLM workflows die at one of three places:

1. **The producer grades its own output** → quality silently drops, you don't notice until the user complains
2. **The orchestrator asks "yes or no?" 10 times** → the human becomes the bottleneck, the loop loses its value
3. **State lives in conversation context** → once context resets or the loop crashes, you can't resume

The **foreman pattern** in this kit solves all three:

- **Verifier isolation** (every checker is a fresh subagent with hardcoded read list)
- **Self-decide first, escalate last** (5 categories of "blocking questions" allowed, everything else must be auto-resolved)
- **Three-file state spine** (STATE / budget / run-log persist across phases and runs)

These aren't ideas I invented. They're battle-tested patterns from the agent engineering literature (see [INSPIRATION.md](./INSPIRATION.md)). This kit just packages them into a working Claude Code skill scaffold.

---

## What's in the box

```
loop-engineering-kit/
├── README.md                  ← you are here
├── PHILOSOPHY.md              ← the 5 iron laws + design rationale
├── INSPIRATION.md             ← lineage & credits
│
├── skills/
│   └── loop-foreman/          ← the scaffold (domain-agnostic)
│       ├── SKILL.md
│       └── references/
│           ├── phase-templates.md      ← 9 phases with [MAKER] / [CHECKER] placeholders
│           ├── iron-laws.md            ← full text of the 5 iron laws
│           ├── taste-subagent.md       ← verifier isolation pattern
│           ├── bash-no-go-zone.md      ← which Bash commands trigger permission prompts
│           └── critic-design-pattern.md ← how to design your own critic-rules.yaml
│
├── examples/                  ← runnable example loops across domains
│   ├── README.md              ← pick the example closest to your domain
│   ├── email-loop/            ← simplest: text production, 9 phases minus 3 + 3.5
│   └── code-review-loop/      ← production-grade: parallel perspectives + cross-review + polish
│
├── templates/                 ← cp these into your project root
│   ├── STATE.md.template
│   ├── loop-budget.md.template
│   └── loop-run-log.md.template
│
└── docs/
    └── how-to-build-your-own-loop.md
```

---

## 5-minute quickstart

```bash
# 1. Clone into your Claude Code skills directory
cd ~/.claude/skills/
git clone https://github.com/jiangjiax/loop-engineering-kit.git

# 2. Copy the three state files into your project root
cd /path/to/your/project
cp ~/.claude/skills/loop-engineering-kit/templates/*.template ./
mv STATE.md.template STATE.md
mv loop-budget.md.template loop-budget.md
mv loop-run-log.md.template loop-run-log.md

# 3. Pick an example closest to your domain
cat ~/.claude/skills/loop-engineering-kit/examples/README.md

# 4. Build your own
# - Copy skills/loop-foreman/ → skills/my-loop-foreman/
# - Replace [MAKER_1] / [MAKER_2] / [CHECKER] with your actual skills
# - Write your own critic-rules.yaml (see docs/how-to-build-your-own-loop.md)
```

Then invoke from Claude Code:

```
> Run my-loop on [your input]
```

The foreman takes over, runs each phase, writes state, and hands you a final report.

---

## What you need to bring

This is a **scaffold**, not a product. You'll need to write:

1. **Your producers** — the skills that actually generate the output (drafting, writing, designing, analyzing, judging). Anything that runs in the main thread.
2. **Your checker** — the skill that grades the output. Runs in a fresh subagent.
3. **Your critic rules** — a YAML file that tells the checker what counts as a fatal/serious/general violation. Without this, your loop has no gate.
4. **Your domain context** — any reference files the producer and checker need to read.

The kit handles the **wiring** (when to spawn subagents, how to vote, where to write state, when to escalate). You handle the **content** (what your loop actually does).

---

## When NOT to use this

This kit assumes:

- You have **multiple sequential stages** (≥ 3) where each stage produces a draft and the next stage refines it
- You need **verification** (some output is harder to verify than to produce)
- You want **iteration without interruption** (not a chatbot — a background producer)

If your task is single-shot ("summarize this PDF"), you don't need a loop. Just call a skill.

If your stages have **no inter-dependency**, use parallel agents instead of a sequential loop.

If verification is trivial (just check a regex), you don't need fresh subagents — inline the check.

---

## License

MIT. Build whatever you want. If you build something cool, open an issue — I'd love to see it.

---

## Credits

The foreman pattern came out of a year of building production agent loops. The patterns are domain-agnostic — they apply equally to text production, judgment work, information synthesis, and any other multi-stage workflow that needs verification. Without the work of the people listed in [INSPIRATION.md](./INSPIRATION.md), there'd be nothing to package here.
