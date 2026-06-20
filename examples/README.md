# Examples

This directory contains runnable example loops built on top of the `loop-foreman` scaffold.
Each example is a complete, self-contained skill you can read end-to-end to see how the scaffold gets filled in for a specific domain.

> **None of these examples are products.** They're calibration points showing how the same 9-phase pattern applies to different kinds of work. Pick the one closest to your domain, fork it, replace the makers and critic-rules.

---

## Pick the example closest to your domain

### 📧 `email-loop/` — text production (simplest)

**What it does**: drafts an email in 3 tones, taste-votes for the best, polishes into 2 variants, critics for tone + spam patterns, archives.

**Read this if**: you produce text outputs (emails, messages, copy, summaries) and want to see the simplest possible end-to-end loop.

**9 phases used**: 0, 1, 1.5, 2, 4, 5, 6, 7, 8 (skip 3 and 3.5 — no third maker stage needed).

**Time to read end-to-end**: ~10 minutes.

---

### 🔍 `code-review-loop/` — non-text, multi-perspective (production-grade)

**What it does**: generates code review comments from 3 perspectives (security / correctness / style) in parallel, cross-reviews them, synthesizes, critics for review-tone + technical accuracy, archives.

**Read this if**: your outputs are **not text-shaped** — they're decisions, judgments, evaluations, structured artifacts. Code review is the canonical "multi-perspective verifier" pattern.

**9 phases used**: all 9 (this is the closest to a "full" production loop).

**Time to read end-to-end**: ~15 minutes.

---

## How to use an example

```bash
# 1. Copy state templates to your project root
cp ~/.claude/skills/loop-engineering-kit/templates/*.template /path/to/your/project/
cd /path/to/your/project
mv STATE.md.template STATE.md
mv loop-budget.md.template loop-budget.md
mv loop-run-log.md.template loop-run-log.md

# 2. Pick an example, install it as a skill
cp -r ~/.claude/skills/loop-engineering-kit/examples/email-loop ~/.claude/skills/my-email-loop
# (or your domain — code-review-loop, etc.)

# 3. Edit the skill to your needs:
#    - Replace the gate files (tone-rules.yaml, etc.) with your domain rules
#    - Adjust the maker logic (inline or invoke your own skills)
#    - Tune the iteration cap in loop-budget.md
```

Then in Claude Code:

```
> Run my-email-loop on [your task]
```

---

## How the examples relate to `loop-foreman/`

All examples are **specializations of the loop-foreman scaffold** at `../skills/loop-foreman/`. They share:

- The same 9-phase structure (`loop-foreman/references/phase-templates.md`)
- The same 5 iron laws (`loop-foreman/references/iron-laws.md`)
- The same Bash red line (`loop-foreman/references/bash-no-go-zone.md`)
- The same taste vote pattern (`loop-foreman/references/taste-subagent.md`)
- The same critic design principles (`loop-foreman/references/critic-design-pattern.md`)

What changes per example:

- **The makers** — what each stage produces (drafts vs. review comments vs. analysis)
- **The critic-rules** — what counts as a violation in this domain
- **The phase count** — some domains skip stages (email-loop skips Phase 3)
- **The output paths** — emails go to `out/[recipient]-final.md`, code reviews go to `out/reviews/[PR-id].md`, etc.

---

## What's missing? (Examples we want but haven't built)

If you build any of these, open a PR. The scaffold supports them — we just haven't written them yet.

- **`research-synthesis-loop/`** — multi-source research synthesis (fetch sources → cross-verify → consolidate → critic for source attribution + claim/evidence balance)
- **`design-critique-loop/`** — multi-version visual design proposals + verifier panel + synthesis
- **`decision-support-loop/`** — multi-perspective proposals (optimist / pessimist / realist) + debate + structured vote
- **`learning-plan-loop/`** — curriculum drafting + difficulty calibration + critic for prerequisite gaps

The pattern is the same. The domain content is what you write.
