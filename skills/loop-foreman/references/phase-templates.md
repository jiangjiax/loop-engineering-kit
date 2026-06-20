# Phase execution templates (9 phases + STATE.md write format)

> SKILL.md is the skeleton. This file is the meat. Each phase starts by Reading the relevant section.
>
> **All `[BRACKETED]` tokens are placeholders. Replace them with skills and paths from your domain.**

---

## Phase 0 · Init STATE + budget reconciliation

Read three files to confirm state:
- `STATE.md` / `loop-budget.md` / `loop-run-log.md` (project root)
- `loop-run-log.md` — sum today's `tokens_estimate` entries, reconcile against budget

Behavior branches:
- `STATE.md` `current_loop: [other topic] in progress` → **Blocking category 2**, ask user whether to archive / abort
- Today's tokens ≥ 80% of daily budget → **Force single-version mode**, mark `single_version_mode: true` in STATE.md
- Today's tokens ≥ 100% of daily budget → **Blocking category 4**, ask user whether to override budget
- Any of the three files missing → **Blocking category 3**, ask user whether to create

Write to STATE.md:
```
current_loop:
  selection: [topic name]
  started_at: [Bash date -u +"%Y-%m-%dT%H:%M:%SZ"]
  outline_path: [outline path]
  status: phase_1_in_progress
  today_tokens_so_far: [N]k
  budget_mode: full_dual / single_version_forced
```

---

## Phase 1 · First maker stage ([MAKER_1] · main thread)

**Replace `[MAKER_1]` with your first content production skill** (e.g., outline generator, cover designer, draft writer).

Check whether the outline contract has a "locked v1 artifact" block:
- Locked + user didn't say "re-run" → skip Phase 1, write contract to STATE.md, proceed to Phase 1.5 (still run taste vote)
- Not locked / user said re-run → run the full flow below

Steps (per Iron Law 5):
1. Bash `find .claude/skills/[MAKER_1] -type f` to list files
2. **Skill tool invocation**: `Skill(skill="[MAKER_1]", args="[your task description] / input: [outline path] / output: 3 candidate versions ranked by [your scoring criterion]")`
3. Foreman manually Reads all references (every file except SKILL.md)
4. Main thread generates 3 candidates based on outline + references
5. **Don't directly accept the highest-scored**, go to Phase 1.5 for taste vote

STATE.md:
```
phase_1_skill_invocation:
  skill_invoked: true / false
  invocation_method: Skill_tool / Read_only  # Read_only = violation per Iron Law 5
  references_read: [...]
phase_1_candidates:
  v1: {...artifact metadata + predicted score}
  v2: {...}
  v3: {...}
  score_ranking: [v1, v2, v3]  # maker's predicted ranking, not the adopted decision
status: phase_1_5_taste_vote_in_progress
```

---

## Phase 1.5 · Taste vote

Full template: `references/taste-subagent.md`.

3 fresh subagents vote in parallel. Candidates = Phase 1's 3 versions.
Adopt the majority, write to STATE.md `phase_1_5_taste_vote` + `locked_v1_artifact`. Runner-up candidates go to Phase 8 report.

`status: phase_2_in_progress`

---

## Phase 2 · Second maker stage ([MAKER_2] · main thread, serial dual-version)

**Replace `[MAKER_2]` with your second content production skill** (e.g., hook writer, opening optimizer, headline generator).

### Iron Law 4: don't skip via "v1 already locked"

| Source of v2 artifact | Foreman action |
|---|---|
| Outline contract explicitly says "v2 generation already run + locked" | Skip Phase 2, write contract to STATE.md (still run step-5 taste vote) |
| Inherited from a previous v1/v2 run | **DO NOT skip** — re-run Phase 2 to verify gate criteria, mark `re_verified: true` |
| User pasted a seed assertion | Run Phase 2 to expand seed into full v2 artifact + verify gate criteria |
| No prior artifact | Run full flow |

### Steps

1. **Run [MAKER_2] (main thread, per Iron Law 5: Skill tool + Read all references)**:
   - `Skill(skill="[MAKER_2]", args="task: [...] / input: [outline] + [locked v1 artifact] / output: 3 candidates + top 1 + [your gate criteria checklist]")`

2. **Optional · foreman self-review**: against your gate files. Any fail → request rewrite.

3. **Foreman synthesizes** (this is where the taste-vote pattern earns its keep):
   - Preserve any user-provided seed assertion
   - Prefer candidates that pass all gate criteria
   - Respect word count / format constraints

4. **Strength self-check** (domain-specific — for example, if your gate is "no hedge words"):
   - `grep` for hedge words in key assertions; any hit = rewrite
   - Apply any other domain-specific scoring

5. **Taste vote** (see `references/taste-subagent.md`):
   - Candidates = foreman's synthesis vs original v1 vs original v2
   - 2:1 against synthesis = foreman rolls back to one of the originals + writes reason

STATE.md:
```
phase_2_skill_invocation: {...}
phase_2_taste_vote: {subagent_count: 3, candidates, votes, picked, picked_by}
locked_v2_artifact:
  content: [...]
  word_count: N
  picked_by: foreman_synthesis_then_taste_voted
  synthesis_logic: [...]
  taste_revision: [whether rolled back to an original, one sentence]
status: phase_3_in_progress
```

User-provided seed branch: mark `picked_by: human_seed_then_[MAKER_2]_verified_then_taste_voted`.

Single-version forced mode: only run version 2 (broader coverage), skip cross-review. **Still run step-5 taste vote**.

---

## Phase 3 · Third maker stage ([MAKER_3] · main thread, serial dual-version)

**Replace `[MAKER_3]` with your third content production skill** (e.g., main body writer, full article draft, full code generation).

Default pattern: conservative version + sharp version, serial dual run + cross-review + foreman synthesis + taste vote.

### Steps

1. **Run conservative version (main thread, per Iron Law 5)**:
   - Bash find `.claude/skills/[MAKER_3]` to list all references
   - `Skill(skill="[MAKER_3]", args="generate conservative version / input: [outline] + [locked v1 artifact] + [locked v2 artifact] / hard constraints: strict to outline word counts / preserve tension sources / steady pacing / output to [topic]-loop-conservative-v1.md")`
   - Foreman Reads all references (not cherry-picked)

2. **Run sharp version (same pattern)**:
   - `Skill(skill="[MAKER_3]", args="generate sharp version / hard constraints flipped: short sentence stacking / high punchline density / output to [topic]-loop-sharp-v1.md")`
   - Prompt cache hits on the first run

3. **Spawn 2 cross-review fresh subagents in parallel** (Agent tool):
   - Conservative perspective reviews sharp version / sharp perspective reviews conservative version
   - Each gives top-3 dangerous issues + 1-2 "the other side did right and we should learn"

4. **Foreman synthesizes**:
   - Use sharp version as the base (stronger structure + clearer punchline peaks)
   - Apply local tweaks per conservative reviewer's 3 dangerous issues (don't rewrite)
   - Output `01-publishable/short-video/[topic]-loop-final-v1.md` (adapt the path to your domain)

5. **Taste vote**:
   - Candidates = synthesis vs original conservative vs original sharp
   - 2:1 against synthesis = foreman rolls back to one of the originals

### Shared hard constraints for both versions

- Phase 2 locked hook = inserted verbatim into the first paragraph
- Phase 1.5 locked v1 artifact = used in the format/title spots
- Sign-off / closing block = if you have a locked closing, insert verbatim

STATE.md:
```
phase_3_skill_invocation_conservative: {skill_invoked, invocation_method}
phase_3_skill_invocation_sharp: {skill_invoked, invocation_method}
draft_paths: {conservative, sharp, final}
draft_word_count: [N]
draft_synthesis: [one sentence: sharp base + conservative local fixes]
phase_3_taste_vote: {subagent_count: 3, candidates, votes, picked, picked_by, taste_revision}
status: phase_3_5_polish_in_progress
```

Single-version forced mode: only run sharp, skip cross-review. **Still run step-5 taste vote + Phase 3.5**.

---

## Phase 3.5 · Optional polish stage ([POLISHER] · main thread)

**Replace `[POLISHER]` with your polish skill** (e.g., a "remove AI smell" pass, a tone normalizer, a length trimmer).

### Why this matters

The critic verifies rules but **doesn't rewrite**. Without a polish layer, the "last-mile residue" — patterns that don't trip critic rules but feel mechanical — only gets caught when the human reviews.

A polish skill scans for whatever your domain considers "machine smell" (overused conjunctions, em-dash overuse, three-part parallelism, stock phrases, etc.) and rewrites them in natural style.

### Steps

1. **Skill tool invocation (per Iron Law 5)**:
   ```
   Skill(skill="[POLISHER]", args="
   draft path: [Phase 3 final path]

   Preserve unchanged:
   ① [your locked closing block]
   ② YAML frontmatter
   ③ Section headings + metadata
   ④ Any pre-approved phrasings (cite the source rule for each)

   Focus on:
   - Overused em-dashes outside of locked punchline positions
   - AI-frequent words like "crucial" / "intricate" / "delve into" (adapt to your domain)
   - Three-part parallelism outside approved peaks
   - Sentence-ending sentiment phrases

   Output: Edit the file directly + write a change log
   ")
   ```

2. **Foreman reviews polisher's changes** against your approved-phrasing list:
   - Changes that touched a locked phrase → foreman Edits back
   - Changes covered by critic-rules approved examples → Edit back
   - Other changes default to accepted

STATE.md:
```
phase_3_5_polish:
  skill_invoked, invocation_method
  changes_count: N
  rolled_back_count: N
  final_path: [post-polish path]
status: phase_4_critic_in_progress
```

A polisher cannot replace the critic (it doesn't know your domain-specific rules). When budget is tight, **still run it** — it's high-ROI for the token cost (10-20k tokens for a major AI-smell pass).

---

## Phase 4 · Critic round 1 ([CHECKER] · fresh subagent)

**Replace `[CHECKER]` with your verifier skill** (e.g., a script critic, a code reviewer, a copy QA).

**Must use Agent tool to spawn a fresh subagent**. Foreman has been polluted by maker reasoning = verifier design collapses.

### Steps

1. Bash `find .claude/skills/[CHECKER] -type f` to list files (e.g., SKILL.md + critic-rules.yaml)
2. Agent tool spawns `general-purpose` subagent, prompt includes:
   - **Hardcoded read list** (full file paths, one-line description of what each file rules)
   - **Explicit instruction**: "Verdict rules live in [CRITIC_RULES_PATH]. SKILL.md is just the flow entry point. Don't read SKILL.md only and start reviewing."
   - **Default stance**: "Default stance: REJECT until proven otherwise"
   - Context: draft path (Phase 3.5 final) + outline path
   - Output: one of 5 verdicts (APPROVE / APPROVE_WITH_NOTES / REJECT_LIGHT / REJECT / ESCALATE_HUMAN) + finding list (fatal / serious / general / escalate)

STATE.md:
```
critic_verdict: [verdict]
critic_findings: {fatal: N, serious: N, general: N, escalate: N}
status: phase_5_decision
```

---

## Phase 5 · Routing (Iron Law 1 + Iron Law 2 main battleground)

### Step 1 · Bucket every critic finding

For each finding, three buckets:

- **A. Gate file has the answer**: critic-rules.yaml example / anti-pattern checklist / [MAKER_2] SKILL.md has an example / "user-approved" original / banned word list → Step 2 auto-resolve
- **B. Context can be inferred**: outline punchline / user's seed assertion / [MAKER_1] locked contract → Step 2 auto-resolve
- **C. Truly unanswerable**: A/B both empty, involves human muscle memory / voice / taste → Step 4 escalate

### Step 2 · Auto-fix bucket A/B

Edit the draft directly (or run [MAKER_3] in "only-fix" mode, only fixing A/B items, not rewriting the whole thing).

Each auto-fix logged to STATE.md `iterations[N].auto_fixes`:
```
auto_fixes:
  - issue: "[description]"
    source: "[gate file path + line]"
    fix: "[changed to]"
```

### Step 3 · Critic round 2 (fresh subagent, not reused)

### Step 4 · Only escalate true C-bucket findings

After A/B auto-fixed and round-2 critic passes, any remaining findings are pure muscle-memory C-bucket.
Action: directly verdict `APPROVE_WITH_NOTES`, archive, write C findings into Phase 8 report as "user check during recording / shipping" notes. **Let the user resolve at recording time, don't block the loop.**

### Verdict matrix

| Critic verdict | Foreman action |
|---|---|
| APPROVE | Proceed to Phase 6 archive |
| APPROVE_WITH_NOTES | Step 1 bucket. Auto-fix A/B; log C to Phase 7 report. Then Phase 6 |
| REJECT_LIGHT | Steps 1-3. Max 3 iteration rounds |
| REJECT | **Do not halt immediately, do not ask the user**. First Step 1 bucket. **≥2 fatal findings in C bucket** → set `escalate_to_outline_in_phase8_report: true`, proceed to Phase 6 archive (with escalate flag), write "suggest rolling back to outline stage" into Phase 8 report |
| ESCALATE_HUMAN | Step 1 bucket. Most ESCALATEs are actually A/B. **Only true C goes to Step 4** |

### Iteration cap

3 rounds hard cap. Round 4 auto-escalates as "3 rounds didn't converge = upstream outline has structural issues" → write into report suggesting user roll back to outline stage (don't ask in real time).

```
iterations:
  - round: 1
    critic_verdict: REJECT
    auto_fixes: [...]
    escalates: [...]
    next_action: rerun_critic
  - round: 2
    critic_verdict: APPROVE_WITH_NOTES
    next_action: phase_6_archive
```

---

## Phase 6 · Persist (archive + state)

Reached here = critic APPROVE / APPROVE_WITH_NOTES.

1. **Rename / copy the final draft**: use Bash `cp` (not heredoc) to copy `[topic]-loop-final-v1.md` to `[topic].md` (strip suffix). Keep the loop work files as the work log. Simple `cp` usually auto-allows; no permission prompt.

2. **Update your topic log** (e.g., `topics-log.md`): use **Read + Edit** to find the entry, append `- loop completed (YYYY-MM-DD)`. **Don't use Bash sed/awk** — those trigger permission prompts and break loop automation.

3. **Append to loop-run-log.md** — one JSON entry:
   - **Forbidden: `cat >> file << 'EOF'` heredoc** (triggers permission prompt, breaks loop automation)
   - **Correct**: Read loop-run-log.md → Edit the file end to append the JSON block
   - JSON fields:
   ```json
   {
     "run_id": "[ISO timestamp]",
     "skill": "[YOUR_LOOP_NAME]",
     "selection": "[topic name]",
     "phases_completed": ["state_init", "phase_1", "taste_phase_1_5", "phase_2_v1+v2", "phase_3_conservative+sharp", "critic_roundN", "archive"],
     "iterations": [round count],
     "final_verdict": "[APPROVE / APPROVE_WITH_NOTES / HUMAN_OVERRIDE]",
     "draft_word_count": [N],
     "duration_minutes": [estimate],
     "tokens_estimate": [estimate],
     "escalations_to_human": [count],
     "auto_fixes_count": [total auto-fix count]
   }
   ```

4. **Clear STATE.md's `current_loop` block** — use **Read + Edit** (not Bash sed/awk) to mark complete (move to comment block as history).

### Bash tool red line (Phase 6 + Phase 4/5 review)

`loop-foreman` is a zero-interruption loop — from user trigger to Phase 8 report there should be **no permission prompts**. The following Bash patterns are **all forbidden**; use native tools instead:

| Forbidden Bash | Replacement | Why |
|---|---|---|
| `cat >> file << 'EOF' ... EOF` (writing logs) | Read + Edit (append) | heredoc triggers permission prompt |
| `sed -i 's/X/Y/' file` (editing file) | Edit | sed -i triggers prompt |
| `sed -n '32,148p' file \| grep -nE "..."` (review grep) | Grep tool + path + multiline | piped sed/grep triggers prompt |
| `awk '...' file` (stats/filter) | Read then process in main thread / Grep tool | awk triggers prompt |
| `find . -name "*.md" -exec ...` (batch operation) | Glob tool + main thread loop | find -exec triggers prompt |
| `wc -m file` (character count) | Read then count in main thread / Grep tool with `head_limit` | simple read but wc also triggers |

**Allowed Bash** (no permission prompt):
- `find <path> -type f` (list files, used before spawning subagents)
- `cp src dst` (file copy, Phase 6 step 1)
- `date -u +"..."` (timestamp, Phase 0 / Phase 6)
- `ls -la <path>` (directory listing)

Heuristic: if a Bash command has a pipe (`|`), redirect (`>>` `>` `<<`), stream editor (`sed -i` `awk`) → 99% triggers permission prompt. **Find a native tool replacement.**

---

## Phase 7 · Sediment (foreman auto-decides + writes suggestion list · no longer asks user)

### Foreman auto-decides on 3 dimensions

**Dimension 1: critic-rules.yaml example gaps**
- Scan Phase 5 `iterations[*].auto_fixes`
- grep `source` hits in `critic-rules.yaml` → already covered, **don't suggest appending**
- Misses (fixed by inference + memory / outline fallback) → **suggest appending example**

**Dimension 2: approved phrasing pool candidates**

Scan the locked v1/v2/v3 artifacts + closing + punchlines. Pass through 4 gates:
- Gate 1: passes your domain-specific quality criteria (e.g., 5 axiom checks, all 5 pass)?
- Gate 2: meets some quality threshold (e.g., strength ≥ tier 2)?
- Gate 3: critic APPROVE / APPROVE_WITH_NOTES?
- Gate 4: has differentiation beyond the generic version your maker would produce by default?

All 4 gates pass → suggest sediment to `approved-phrasing-pool/[type]/`

**Dimension 3: project memory candidates**

Per your project's memory schema (e.g., `MEMORY.md` triggers): new long-term judgment / user characteristic change / prior judgment overturned. Hit → suggest memory update, name file path + section + change.

### Phase 7 output (don't ask user in real time)

```
phase_7_sediment_self_decided:
  asked_at: [timestamp]
  suggested_critic_rules_additions: [N items · brief + location]
  suggested_patterns: [N items · pattern name + 4-gate score]
  suggested_memory_updates: [N items · file path + section + change]
  rationale: [foreman 1-2 sentence reasoning]
  ↑ All goes into Phase 8 report "🌱 sediment suggestions for user review" section
status: phase_8_report
```

### User's batch-review pattern

User reviews the "🌱 sediment suggestions for user review" section in the Phase 8 report later:
- "sediment X into critic-rules" → foreman Edits `critic-rules.yaml` to append example
- "sediment Y into pattern pool" → foreman Writes to `approved-phrasing-pool/[type]/[pattern_name].md`
- "update memory" → execute per your memory schema
- "skip" → foreman logs `lessons_skipped: true` to loop-run-log.md
- User silent → default skip, next time the same topic hits the same finding → report adds "[N] repeated suggestion not adopted, suggest batch decision"

---

## Phase 8 · Final report to user

```markdown
# Loop report · [topic]

**Total time**: [minutes]  **Iterations**: [N]  **Final verdict**: [verdict]

## ✅ Completed
- v1 artifact: [...]
- v2 artifact: [...]
- v3 final draft path: [path]
- Word count: [N] (target range [...])

## 🪑 Unadopted candidates (taste-vote + foreman synthesis runners-up, for user review)
- v1 v2 / v3: [brief + score + taste-vote rationale + foreman's reason for not adopting]
- v2 v1 / v2 pre-synthesis candidates: [brief + gate-check + synthesis rationale]
- v3 conservative version differences: [3-5 key differences from the adopted version]

## 🟡 Critic ESCALATE notes for user (non-blocking, address at production time)
[List C-bucket remaining items]

## 🌱 Sediment suggestions for user review (Phase 7 foreman auto-decided products)

### A. Critic-rules.yaml example gap suggestions
- Suggest appending [N] examples:
  - [issue description] → suggested location: [yaml field] / suggested example: [text]
- User reviews and replies "sediment A1 / A2" or "skip"

### B. Approved phrasing pool candidates
- [N] items passing all 4 gates:
  - [pattern name] · quality score X/10 · strength tier X · suggested path: [file path]
- User reviews and replies "sediment B1 / B2" or "skip"

### C. Memory update suggestions
- [file path] / section / change suggestion
- User reviews and replies "update memory" or "skip"

### D. Repeated unadopted suggestions (if any)
- [N]th repeated suggestion "[X]" not adopted — suggest batch decision (accept / permanently reject / continue watching)

## 📌 Next step suggestions
- [record / manual tweak / roll back to outline stage etc.]

## 📊 Loop evolution progress
- Cumulative loop runs: [N] (from loop-run-log.md)
- Cumulative sediment into critic-rules: [N items]
- Estimated distance to full automation: [critic-rules coverage X%, target ≥30 items]
```
