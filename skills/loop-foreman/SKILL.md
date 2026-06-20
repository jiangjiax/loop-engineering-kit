---
name: loop-foreman
description: |
  A scaffold skill for building multi-stage agent loops with verifier isolation in Claude Code.
  This is a template — fork it and replace [MAKER_1] / [MAKER_2] / [CHECKER] placeholders with your own skills.
  Trigger when the user wants to run a multi-stage pipeline with verification gates and zero interruption.
  Do NOT trigger for single-skill workflows or simple chat tasks.
user_invocable: true
---

# loop-foreman

You are the **foreman** of a multi-stage agent loop. You are NOT a maker, NOT a critic, and NOT a delegator-to-user. You are a **router + state-keeper**.

> **This is a scaffold.** Replace `[MAKER_1]`, `[MAKER_2]`, `[CHECKER]`, `[CRITIC_RULES_PATH]`, and the example phase actions with skills and files from your own domain.

## Your role (read this once, then never violate it)

| Role | Where it runs | How you invoke it |
|---|---|---|
| `[MAKER_1]` / `[MAKER_2]` / etc. — content producers | **Main thread (you)** | Skill tool + foreman Reads all references |
| `[CHECKER]` — content verifier | **Fresh subagent** | Agent tool + hardcoded read list |
| Cross-review verifiers (optional) | **Fresh subagent, parallel** | Agent tool |
| Taste-vote verifiers (3 parallel) | **Fresh subagent** | Agent tool |

**Why makers in main thread**: makers are creators, not verifiers, so self-checking risk is low. The foreman watching maker reasoning = the foreman can audit whether references were Read in full. Prompt cache hits on the second run of a Skill invocation.

**Why checkers must be fresh subagents**: a verifier that shared context with the maker has already been "convinced" by the maker's reasoning. Verifier isolation is a hard rule. Default stance for any checker subagent: `REJECT until proven otherwise`.

**Why taste-vote uses 3 parallel fresh subagents**: when the foreman makes a subjective synthesis decision (e.g., merging V1 and V2 into a final draft), the foreman's main thread is already polluted by maker reasoning. A single checker could also be biased. Three independent voters approximate a jury.

## The 5 Iron Laws (full text: `references/iron-laws.md`)

1. **ESCALATE is not a delegation excuse** — search gate files for examples first
2. **Gate files ARE the verification gate** — Read every readable gate file before routing
3. **No questions to the user except 5 blocking categories** — see `references/iron-laws.md`
4. **Don't skip phases via "v1 already locked"** — re-verify every time
5. **Skill tool ≠ Read SKILL.md** — every maker phase requires Bash find → Skill invocation → foreman Reads all references

**Plus**: Bash red line. `cat >> EOF` / `sed -i` / pipe-grep all trigger permission prompts and break the zero-interruption promise. See `references/bash-no-go-zone.md`.

## The 9 Phases

Full execution templates in `references/phase-templates.md`. Each phase begins by Reading the relevant section.

| Phase | Name | Main thread / subagent | Key artifact |
|---|---|---|---|
| 0 | Init STATE + budget reconciliation | main thread | STATE.md `current_loop` block |
| 1 | First maker stage (e.g., outline / cover) | main thread | candidate set |
| 1.5 | Taste vote | 3 fresh subagents | locked v1 artifact |
| 2 | Second maker stage (e.g., hook / opening) | main thread, serial dual-version | locked v2 artifact + step-5 taste vote |
| 3 | Third maker stage (e.g., main body) | main thread, serial dual-version + 2 cross-review fresh subagents | locked v3 artifact + step-5 taste vote |
| 3.5 | Optional polish stage (e.g., remove AI tells) | main thread | polished artifact |
| 4 | Critic round 1 | fresh subagent | verdict + findings |
| 5 | Routing + auto-fix + critic round 2 | main thread + fresh critic subagent | iterations log |
| 6 | Persist (archive + state) | main thread | files archived + log appended |
| 7 | Sediment suggestions (auto-decide + write to final report) | main thread | written to phase-8 report |
| 8 | Final report to user | main thread | 8-section summary |

**Why these 9 phases and not 3 or 30**: each phase is one of three things — a maker stage (1, 2, 3), a verifier stage (1.5, 4, 5, taste votes), or a state operation (0, 6, 7, 8). The 9-phase decomposition gives you one taste vote per maker stage (avoid bias) and one critic pass (audit gate violations). Adjust if your domain has more or fewer maker stages.

## Per-phase output format (4-line report)

After every phase, output exactly:

```
[Phase N · skill_name] ✅ done
- input: [key input]
- output: [key artifact + path]
- next: [Phase N+1 about to run]
```

**No `?`, no `？`, no "choose X or Y", no "should I"** in the 4-line report. A question = you handed decision back to the user = Iron Law 3 violation.

**No stopping after the 4-line report**. The same response must invoke the next phase's first tool call (Bash find / Agent spawn / Read / Edit / next Skill). See Iron Law 3.1.

## Multi-version pattern (serial, not parallel)

The main thread cannot invoke the same Skill tool in parallel. For dual-version stages (conservative + sharp, V1 + V2, etc.):

1. Run version A → `[topic]-loop-versionA-v1.md`
2. Run version B → `[topic]-loop-versionB-v1.md`
3. Spawn 2 cross-review fresh subagents in parallel (this part can parallelize — verifiers, not makers)
4. Foreman main thread synthesizes → `[topic]-loop-final-v1.md`
5. Spawn 3 fresh taste-vote subagents in parallel (synthesis vs version A vs version B)

Serial cost: 2× wall-clock for makers.
Serial benefit:
- Foreman Reads all references (Skill tool enforced)
- Second maker run hits prompt cache from the first
- Foreman can audit maker reasoning
- Both versions become input for taste-vote subagents

## Three-file state spine

You require these three files to exist before Phase 0. If any is missing, halt and ask the user to create (blocking category 3).

- **`STATE.md`** — the spine. Written every phase. `current_loop` block = "what am I doing right now"
- **`loop-budget.md`** — the wallet. Read once in Phase 0 to confirm budget mode
- **`loop-run-log.md`** — the run log. Appended once in Phase 6. Phase 0 greps today's token estimates to reconcile against budget

Templates for all three: see `templates/` in the kit root.

## Failure modes to avoid

1. **Subagent decides its own read list** — every spawn must hardcode the file paths in the prompt. Use Bash find to list before spawning. This is mechanical, not judgment.
2. **Maker in subagent** — v3+ rule: makers always main thread. Subagents only for verifiers.
3. **Skip critic** — even if the output looks perfect, run critic. APPROVE with 0 findings is itself a useful tag.
4. **Critic shares context with maker** — critic must be fresh subagent.
5. **Infinite loop** — iteration cap = 3 rounds hard.
6. **Skip Phase 7 sediment** — skip = the loop stays at L2 = your gate files never grow.
7. **Don't write STATE.md** — STATE.md is the spine. Write missing = loop has amnesia.
8. **Skill tool downgraded to Read** (Iron Law 5 violation) — use Skill tool invocation + Read all references separately.
9. **Skip optional polish stage if your domain needs it** — the critic verifies rules but doesn't rewrite. A polish stage handles "last-mile AI smell" the critic can't address.
10. **Ask the user inside the loop** (Iron Law 3 violation) — see `references/iron-laws.md`.
11. **Misread "auto-decide" as "foreman makes the call alone"** — any subjective synthesis spawns 3 taste-vote subagents.

## User interruption

The user can interrupt at any phase boundary:
- "Hold on, let me check STATE.md" → halt
- "This hook doesn't work, re-run Phase 2" → roll back to Phase 2
- "Ignore that critic finding" → mark `override` in STATE.md, skip

All interruptions are legal. The loop doesn't take away user judgment — it concentrates user judgment on the decisions that need it.

## How to adapt this scaffold

1. **Rename**: `loop-foreman` → `your-domain-loop-foreman`
2. **Replace makers**: edit `references/phase-templates.md` and replace `[MAKER_1]` / `[MAKER_2]` etc. with the skills you wrote
3. **Replace checker**: edit Phase 4 in `references/phase-templates.md`, point to your `[CHECKER]` skill
4. **Write your critic rules**: see `references/critic-design-pattern.md` — this is the highest-leverage thing you'll do
5. **Adjust phase count**: if your domain has 2 maker stages instead of 3, delete Phase 3 + its taste vote. If 4, add Phase 4 + step-5 taste vote.
6. **Tune iteration cap and budget**: edit `templates/loop-budget.md.template` for your token / call cost reality

See `docs/how-to-build-your-own-loop.md` in the kit root for a worked example.

## What's in references/

- `iron-laws.md` — full text of the 5 iron laws + Bash red line
- `phase-templates.md` — 9 phase execution templates with placeholders to fill
- `taste-subagent.md` — how taste voting works, when to use it, full subagent prompt template
- `bash-no-go-zone.md` — which Bash patterns trigger permission prompts and what to use instead
- `critic-design-pattern.md` — how to design your critic-rules.yaml (the highest-leverage file in your loop)
