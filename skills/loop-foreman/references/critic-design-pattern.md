# How to design your critic-rules.yaml

Your `critic-rules.yaml` (or whatever format you choose — JSON, TOML, plain Markdown — the format doesn't matter, the content does) is **the single highest-leverage file in your loop**. It defines:

- What counts as a fatal / serious / general violation
- What auto-fix the foreman should attempt for each violation type
- Which "approved phrasings" override the rules

A loop without a well-designed critic-rules file is a critic without teeth.

This document is about **how to design that file**, not which format to use.

---

## The 4-tier severity model

Every critic finding falls into one of four severity tiers. Map your domain-specific rules to these tiers:

| Tier | Meaning | Foreman action |
|---|---|---|
| `fatal` | Cannot ship — would actively harm the user's reputation / break the contract with the audience | Mandatory rewrite, no exceptions |
| `serious` | Significantly degrades quality but doesn't break the contract | Auto-fix if gate files have an answer, otherwise log to Phase 8 |
| `general` | Style or polish issue, would be noticed by a careful reader but won't trigger rejection | Auto-fix per rule, don't escalate |
| `escalate` | Something a fresh subagent can't judge — requires the user's voice / taste / context | Write to Phase 8 report, don't fix |

**Most domains over-use `fatal`.** A good calibration: 80% of findings should be `general`, 15% `serious`, 3% `fatal`, 2% `escalate`. If your critic is returning 50% fatal, your bar is too low — or your maker is genuinely bad and you should fix the maker, not raise the threshold.

---

## Anatomy of a critic rule

A well-formed rule has these parts:

```yaml
rule_id: avoid_em_dash_overuse
tier: general
description: Em-dashes are overused in LLM output; cap density per paragraph
trigger:
  pattern: "—"
  threshold: 2  # paragraph-level cap
exception:
  - location: "locked_punchline_positions"  # don't penalize approved punchline em-dashes
  - approved_phrasing_pool: true  # don't penalize anything in the approved pool
example:
  bad: "I want to say — and I really mean this — that — actually — never mind."
  good: "I want to say, and I really mean this, that, actually, never mind."
auto_fix:
  action: replace_or_remove
  prefer: ","
foreman_self_check:
  - "Run Grep with pattern '—' and count per paragraph. If > 2, flag."
```

Note the **`example`** field. This is what saves the loop from over-escalating per Iron Law 1 — the critic subagent can match a finding against the example and decide "yes, this is the same pattern" without escalating to the user.

---

## How to bootstrap your critic-rules

You don't write critic-rules.yaml on day 1. You **harvest it from loop failures**.

### Stage 1 · Minimum viable critic (week 1)

Write maybe 5-10 rules covering the things you already know break your output:

- Banned words / phrases (the "every LLM uses this" list)
- Hard limits (word count, sentence length, formatting rules)
- The 2-3 obvious anti-patterns in your domain

Run the loop. See what slips through.

### Stage 2 · Harvest from human feedback (weeks 2-4)

Every time you reject the foreman's output for a reason not covered by current rules:

1. Write a new rule for it
2. Add an `example` field with the actual bad output you rejected
3. Mark the corresponding tier

After 3-4 weeks, you'll have 30-50 rules covering 90% of common failures.

### Stage 3 · Approved phrasing pool (weeks 4+)

Once you've shipped 10+ pieces through the loop, you'll start noticing phrasings that **technically violate a rule but were intentional**. For example, "—" overuse might be a violation, but the phrase "—and that's the whole point" is a signature move you want to preserve.

Create an `approved_phrasings/` directory. Add the intentional violations there. Update your critic-rules to add `exception: approved_phrasing_pool: true` to relevant rules.

This is **Phase 7 sediment in action**: the foreman writes "saw approved phrasing X, suggest sedimenting to pool" to Phase 8 report, you batch-approve, and the rules learn.

---

## The verification gate pattern (Iron Law 2 in practice)

Your critic-rules file is "**the verification gate**" the foreman references in Iron Law 2.

Before the critic round, the foreman should:

1. Read critic-rules.yaml in full
2. Read any auxiliary gate files (style guide, banned word list, approved phrasing pool)
3. Then spawn the critic subagent, passing all gate file paths in the prompt's required-reading section

The critic subagent then:

1. Reads all gate files (hardcoded in prompt)
2. Scans the draft against each rule
3. For each finding, **searches the rule's `example` field** for matching patterns first
4. Only returns `ESCALATE_HUMAN` if no example matches and no example can be inferred from the outline

This way, **most findings auto-resolve via the example library** rather than going to the user.

---

## Common rule categories (cross-domain)

Even though your domain matters, these rule categories appear in nearly every loop:

### A. Lexical (word / phrase level)
- Banned words ("crucial", "leverage", "delve into")
- Banned phrases ("In conclusion", "It is important to note")
- Frequency caps (max 2 em-dashes per paragraph, max 1 three-part list per section)

### B. Structural (sentence / paragraph level)
- Sentence length distribution (% short sentences, % long)
- Parallelism overuse (3-part lists cap)
- Hedge density (qualifying words like "perhaps", "might", "could")

### C. Voice / stance level
- First-person vs second-person split (e.g., "you should..." patterns)
- Hedging vs asserting balance
- Stance consistency with locked outline

### D. Domain-specific (your unique sauce)
- Forbidden topic claims
- Required disclaimers / sources
- Approved phrasings for recurring concepts

A good critic-rules.yaml covers all four categories. If you only have A and B, you have a generic style checker. The D category is what makes the critic valuable to your domain.

---

## What the loop foreman does with your critic-rules

When the foreman routes a critic finding:

1. **Tier check** — fatal / serious / general / escalate, per the rule's `tier`
2. **Example match** — does the finding match an `example` in the rule? If yes, the foreman has a precedent → auto-fix without escalation
3. **Exception match** — is the finding in the `approved_phrasing_pool`? If yes, override the rule
4. **Auto-fix execution** — apply the rule's `auto_fix.action` to the draft
5. **Re-run critic round 2** — verify the fix doesn't introduce new findings
6. **Final report** — log every auto-fix to STATE.md + Phase 8 report, so the user can audit

The foreman never silently auto-fixes — every fix is logged with `source: [rule_id] + [line in critic-rules.yaml]` so the user can see what rule fired.

---

## How to know your critic-rules is mature

Signs you've reached "maturity":

- ✅ Most critic findings resolve via example match (≥ 80%)
- ✅ The `ESCALATE_HUMAN` rate is < 5% of findings
- ✅ Phase 6 archive runs without user intervention 9 out of 10 times
- ✅ Your `approved_phrasing_pool/` has 20+ entries (signals you've sedimented real patterns)

Signs you're still early:

- ❌ Every loop has 3+ user interruptions
- ❌ Critic verdicts are 50% REJECT or worse
- ❌ Foreman's Phase 8 "sediment suggestions" section is consistently long (each suggestion = a rule that should already exist)

The kit gives you the foreman + the routing. **Your critic-rules.yaml is the part you have to keep growing.**

---

## Example rule (from a hypothetical email-writing loop)

```yaml
rules:
  - rule_id: avoid_corporate_padding
    tier: general
    description: Cut filler phrases that delay the ask
    trigger:
      patterns:
        - "I hope this email finds you well"
        - "Just wanted to reach out"
        - "I wanted to touch base"
    auto_fix:
      action: remove
    example:
      bad: "Hi team, I hope this email finds you well. Just wanted to follow up on the timeline."
      good: "Hi team, following up on the timeline."

  - rule_id: ask_must_be_first_sentence
    tier: serious
    description: The ask is the most important part; bury it and the reader doesn't act
    trigger:
      check: "is_ask_in_first_two_sentences"
    auto_fix:
      action: restructure_paragraph
    example:
      bad: "Background: project X has been running for 3 months and we noticed Y. Could you approve Z by Friday?"
      good: "Could you approve Z by Friday? Background: project X has been running for 3 months and we noticed Y."

  - rule_id: no_apologetic_close
    tier: general
    description: "Sorry for the trouble" / "Apologies for bothering you" lowers stakes
    trigger:
      patterns:
        - "Sorry for the trouble"
        - "Apologies for"
        - "I know you're busy"
    auto_fix:
      action: remove_line
    example:
      bad: "...by Friday. Sorry for the trouble!"
      good: "...by Friday."
```

This gives the critic enough to flag a problem **and** the foreman enough to auto-fix it **without escalating to the user**.

That's the design target.
