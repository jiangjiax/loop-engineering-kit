# Taste vote subagent pattern (shared by Phase 1.5 / Phase 2 step 5 / Phase 3 step 5)

## Why this exists

Every "foreman subjectively synthesizes" step in the loop has the same cognitive risk profile as critic review — and the same solution: **verifier must be fresh**.

- **Maker's predicted score is just a prediction**, not ground truth. Foreman auto-accepting the highest-scored candidate = outsourcing taste judgment to an unreliable predictor.
- **The foreman main thread is already polluted by maker reasoning** after running the maker. Foreman self-evaluating taste will bias toward "the version that flowed best while I was reasoning."
- **3-vote convergence > single foreman single perspective**.

## Subagent invocation pattern

Spawn 3 fresh `general-purpose` subagents in parallel (Agent tool, same prompt, different agent IDs).

### Before spawning · Bash list the anchor read list

This is a mechanical step. The subagent prompt hardcodes the full paths.

```bash
find [YOUR_DOMAIN_CONTEXT_DIR]/ -name "*.md"
find [YOUR_STYLE_GUIDE_DIR]/ -name "*.md"
ls [YOUR_APPROVED_PHRASING_POOL]/ 2>/dev/null  # may not exist yet
```

## Subagent prompt template

```
You are a Phase [N] Taste Vote subagent. You are NOT a creator — you are a taste evaluator.

## Required reading (Read in order, finish all before evaluating)

### User's taste anchors
1. [PATH_TO_USER_PROFILE_INDEX]      ← who is the user, what stage, what voice
2. [PATH_TO_LONGTERM_PRINCIPLE_1]    ← e.g., the core principle / framework / signature voice
3. [PATH_TO_LONGTERM_PRINCIPLE_2]
4. [PATH_TO_LONGTERM_PRINCIPLE_3]

### User's domain success patterns
5. [PATH_TO_DOMAIN_ANALYSIS_INDEX]
6. [PATH_TO_DOMAIN_PATTERN_1]        ← e.g., what makes pieces succeed in this domain
7. [PATH_TO_DOMAIN_PATTERN_2]
8. [PATH_TO_DOMAIN_PATTERN_3]
9. [PATH_TO_DOMAIN_PATTERN_4]

### Approved phrasing pool (if exists)
10. [PATH_TO_APPROVED_PHRASING_POOL]  ← list all files and Read

### Context for this run
11. [OUTLINE_PATH]
12. STATE.md's [phase_N_candidates section]  ← foreman's candidates + brief

## Your task

**Do not regenerate solutions.** Only vote on the N versions the foreman gave you.

Score each version on 3 dimensions (1-10, decimals allowed):

1. **Persona fit**: alignment with the user profile + longterm principles
2. **Stance continuity**: consistency vs. break with historical decisions in this domain
3. **Track alignment**: alignment with the outline's locked [track / theme / framing] vs. drift

**Do NOT score predicted-quality** (you are fresh on purpose, to not be polluted by the predictor).

## Output (return strict JSON)

```json
{
  "vote": "v1" | "v2" | "v3",
  "scores": {
    "v1": {"persona": 8.5, "stance_continuity": 7.0, "track_alignment": 9.0, "total": 24.5},
    ...
  },
  "reason": "[1-3 sentences, must quote a taste anchor]",
  "plan_b_note": "[if a non-winning version has irreplaceable value, one sentence]"
}
```

Default stance: **conservative beats aggressive**. When 3 versions differ by < 2 points, lean toward the version with the highest persona fit (a persona break costs more than 1 point of predicted quality).
```

## Foreman vote-tally rules (applies to all 3 phases)

- **3 votes unanimous** → directly adopt, STATE.md writes `picked_by: taste_unanimous`
- **2:1 majority** → adopt majority, minority + reason goes to Phase 8 report "🪑 unadopted candidates" section
- **1:1:1 split** → tiebreaker by average persona fit (NOT predicted quality, NOT contrarian strength), STATE.md marks `picked_by: taste_split_tiebreaker_by_persona`, all 3 reasons go to Phase 8 report

## STATE.md write format

```
phase_[N]_taste_vote:
  subagent_count: 3
  candidates: [v1, v2, v3]  # or [synthesis, original_a, original_b] for Phase 2/3
  votes: {agent_1: v1, agent_2: v1, agent_3: v2}
  ranking: {v1: 2, v2: 1, v3: 0}
  picked: v1
  picked_by: taste_majority | taste_unanimous | taste_split_tiebreaker_by_persona
  persona_scores: {v1: avg X.X, ...}
  stance_continuity_scores: {...}
  track_alignment_scores: {...}
  reasons: [agent 1: ..., agent 2: ..., agent 3: ...]
  plan_b_notes: [...]
```

## Phase adaptation differences

- **Phase 1.5**: candidates = Phase 1's 3 versions (v1/v2/v3) directly
- **Phase 2 step 5**: candidates = foreman synthesis vs original v1 vs original v2. 2:1 against synthesis = foreman rolls back to one of the originals.
- **Phase 3 step 5**: candidates = foreman synthesis vs conservative original vs sharp original. 2:1 against synthesis = foreman rolls back.

## Single-version forced mode

When budget is tight (triggered in Phase 0), still run taste vote. When tokens are tight, downgrade to 2 subagents. **Do not skip.** 3 subagents ≈ 30-50k tokens, very cost-effective vs. single-run budget of 350k.

## Decide "should this step have a taste-vote subagent?"

Ask: "Is this step a subjective foreman selection or synthesis?"
- Yes → must add taste vote
- Critic (rule-verification) → no taste vote (critic is itself the verifier)

## Why not just use the critic for taste decisions too?

The critic verifies "did you break the rules" (rule-based, deterministic-ish).
Taste vote verifies "is this the right one for this user in this context" (subjective, calibrated to taste anchors).

Same architectural pattern (fresh subagent + hardcoded read list + REJECT-default), different scoring criteria. Don't merge them — you'll lose both signals.
