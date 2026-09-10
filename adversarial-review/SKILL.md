---
name: adversarial-review
description: Runs a three-role adversarial code review closure — a hostile Blue Team that must attach evidence chains, an independent Third Party that distrusts both sides' documents and hunts defects introduced by the fixes, and a neutral adjudicator that accepts or rejects each finding. Use when the user asks for a "blue team review", "adversarial review", "red team review", "pre-release review", "cross-check my changes", or "find bugs in my code", or wants a stronger quality gate before a release.
license: MIT
compatibility: Requires a host that can dispatch independent subagents (Claude Code, DeepSeek Harness, Cursor, Codex, or similar). Without subagent support, degrade to a single agent playing all three roles serially and state that in the report.
metadata:
  author: RevolutionLA
  version: "1.1"
---

# Adversarial Review (Blue Team / Third Party / Adjudication)

A three-role adversarial code review closure that replaces "let me look at this again" with **structured adversarial relationships**.

## When to Use This Skill

- The user says "blue team review", "adversarial review", "red team review", "cross-check my changes", "find bugs in my code", "poke holes in this", or "pre-release review".
- A feature is complete and about to be merged.
- A release is being prepared, or a large refactor just landed.
- Code was changed in response to a previous review and you want to verify nothing new broke.
- The user wants to improve code quality and is willing to spend some API budget to be genuinely challenged.

Do **not** use it for one-line copy edits, or when the user wants reassurance rather than critique — this skill's posture is hostile by design.

## What This Skill Does

Single-person or single-AI development has three failure modes that "looking more carefully" cannot fix, because the person writing and the person reviewing are the same:

1. **Unverified optimistic assumptions postpone fixes** — "surely this fallback covers it".
2. **Tests provide false security** — the suite is green but the feature broke long ago.
3. **Fixing A introduces B** — the newly written fix is itself never independently examined.

This skill splits the *positions*, not just the effort:

| Role | Stance | Hard constraint |
|---|---|---|
| **Blue Team** | Hostile review — assume it breaks, then prove it | Every finding needs an evidence chain (`file:line`, a repro command, or upstream source). No "consider improving robustness" platitudes. |
| **Third Party** | Independent audit — trusts **neither** side's documents, reads only code | Must verify claim-by-claim that "fixed" means actually fixed, and must specifically hunt defects **introduced by the remediation** |
| **Adjudicator** | Accepts / rejects / re-grades each Third Party finding | Must be a **different agent from the Blue Team**. The accused cannot be the judge. |

The point is not "get two more AIs to look at it" — it is the **ordering and mutual distrust**: the Third Party distrusts the developers, the Adjudicator distrusts both the Blue Team and the Third Party. Remove any stage and the closure breaks.

## How to Use

### Basic Usage

```
Run a blue team review on the auth module before I merge.
```

### Advanced Usage

```
This release is going out tonight — run a heavy-tier adversarial review.
Heavy tier: separate parallel subagents for compatibility, functional safety,
and ecosystem coexistence.
```

Tiers:

| Tier | Configuration | Use for |
|---|---|---|
| **Light** | 1 Blue Team subagent (correctness / compatibility / test validity) | Small changes, tight deadlines |
| **Standard** (default) | Blue Team → Third Party → Adjudicator, 3 rounds | Normal feature work |
| **Heavy** | Standard + parallel cross-verification across dimensions | Pre-release, major refactors, upstream compatibility |

### The Five Steps

**Step 0 — Pick tier and scope.** Record the baseline with `git rev-parse --short HEAD` and `git status`. Commit or at least stage the work first: a chaotic working tree makes findings unreliable.

**Step 1 — Blue Team (hostile review).** Dispatch a `subagent`. **Do not** use a forked agent — a fork inherits your reasoning and will just agree with you. The prompt must forbid trusting the developers' docs and comments, must target compatibility / error handling / test validity / declared-vs-actual mismatches / newly introduced defects / resource lifecycle / security, and must require real evidence over assertion.

**Step 2 — Third Party (independent audit).** Dispatch a **new** subagent. It must relocate all line numbers itself (they shift after remediation), verify each Blue Team item actually landed, and — its most important unique duty — find defects the remediation introduced.

**Step 3 — Adjudication.** Dispatch a **third, fresh** subagent as a neutral adjudicator. It trusts neither report, re-reads the code, and rules accept / partial / reject on each item, explicitly permitted to raise or lower severity. See the warning below.

**Step 4 — Remediation.** The main agent does the fixes personally — remediation needs global consistency judgement. Every fix needs a regression test, and tests must assert **observable behavior**, not implementation strings.

**Step 5 — Close out and archive.** Separate "fixed" / "deferred (with verified reason)" / "could not verify". Archive reports (a `docs/review/` convention is described in the full skill).

### Why the adjudicator must not be the Blue Team

An earlier version of this skill let the Blue Team adjudicate the Third Party's findings, reasoning that "the Blue Team doesn't trust the Third Party" fit the spirit. **This was wrong, and it was caught by running this skill on itself.**

The adjudication targets *the Third Party's review of the Blue Team's findings*. If the Blue Team adjudicates, it is simultaneously **the accused and the judge** — it will systematically reject findings against itself. That directly contradicts the core rule in Steps 1–2 (never reuse an agent, or it defends its own conclusions). Express "the Blue Team doesn't trust the Third Party" through the **adjudicator's prompt stance**, not by reusing the agent's identity.

## Example

**User**: "Run a blue team review on this module before I merge."

**Output** (abridged — a real run against this skill's own SKILL.md):

| ID | Severity | One-line | Dimension |
|---|---|---|---|
| B1 | 🔴 | Step 3 lets the Blue Team adjudicate findings against itself, contradicting the skill's own core rule | Governance |
| B2 | 🟠 | `description` lacks English trigger phrases, so international users never activate it | Usability |

**B1 evidence chain**: `SKILL.md:163` reads "or reuse the Blue Team agent via `send_message` to adjudicate — this better fits the 'Blue Team distrusts the Third Party' premise", while `SKILL.md:250` explicitly forbids "letting the same agent review, fix, and re-review → it will defend its own conclusions".

**Trigger**: any standard-tier review where a Third Party finding criticizes the Blue Team's original report.

**User-visible consequence**: criticism of the Blue Team is systematically rejected; the user receives a "the Blue Team is always right" adjudication and believes the closure held.

**Fix direction**: Step 3 dispatches a brand-new neutral adjudicator; the distrust posture lives in the prompt, not in a reused agent.

The full report also includes a priority roadmap (P0/P1/P2), "what was done right and should be preserved", and a declaration of **unverified items** — honesty about what was not verified is worth more than pretending it was.

**Inspired by:** the author's own development practice of running mutually distrustful review roles to catch defects that self-review structurally cannot.

## Tips

- **Never fork the Blue Team.** A fork inherits your framing and will only agree with you.
- **Never let the Blue Team adjudicate.** The accused cannot be the judge.
- **Require unverified-item declarations.** It makes uncertainty explicit instead of hiding it behind a confident tone.
- **Require reachability analysis.** Otherwise theoretical edge cases get graded critical and dilute attention away from real problems.
- **Let the adjudicator reject.** The Third Party is also fallible; without adjudication the process degenerates into "more opinions is better".
- **Demand evidence, not advice.** "Consider adding robustness" is not a finding. `file:line` plus a reproduction is.

## Common Use Cases

- Pre-release quality gate on a feature branch.
- Verifying that remediation of a previous review did not introduce new defects.
- Auditing a suspicious test suite that is green but unconvincing.
- Checking declared-vs-actual consistency before publishing (README/CHANGELOG claims vs the code).
- Adversarial review of a plugin or library before publishing it to an ecosystem where you cannot easily retract it.

## Honest Limitation (read this before trusting the output)

Subagents and the main agent are usually **the same model**, so this is not a genuinely independent Third Party. The value comes from **role constraints plus mandatory evidence**, not from "another AI's opinion".

- ✅ **Reliably catches**: code-level errors, logic holes, fabricated tests, self-contradictions, missed branches, declared-vs-actual mismatches.
- ❌ **Cannot catch**: **shared blind spots** — for example, a mutual misunderstanding of an upstream system's behavior. If a conclusion depends on external system behavior, it must be **empirically verified**, not settled by two agents nodding at each other.

When reporting, do not say "verified by an independent third party" — say "adversarially reviewed by different roles of the same model".
