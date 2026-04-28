---
name: codex-pair
description: Run the codex-pair collaboration pipeline (architecture → plan with interface contract → consensus loop → codex implements → codex code-tests → claude behavior-verifies → claude commits) when the user wants Claude and Codex to work together on a project. Triggers on phrases like "codex pair this", "two-AI mode", "use codex on this", "have codex pair with you", or "let codex collaborate". NOT for solo tasks, simple one-off codex consultations, or when the user explicitly says "skip codex" / "I'll do it myself".
---

# Codex Pair Workflow

A pipeline for projects where Claude and Codex collaborate, each playing to their strengths. Designed for users who cannot or do not want to review code themselves — both AIs catch each other's mistakes before code touches main. Follow the pipeline strictly.

## Why this exists

When one AI works alone, its mistakes go uncaught unless the user can review code. When two AIs work without coordination, they duplicate work, drift on interfaces, and over-engineer in different directions. This pipeline solves both: forced consensus before code, forced cross-review after code, and a single user-decision point (architecture).

## Role split

- **Claude** — front-of-house: user dialogue, architecture, written plan, behavior-level verification, git operations. Has access to git, browser/UI, and the user.
- **Codex** — back-of-house: deep code synthesis, code-level testing, implementation. Runs in sandbox; cannot write to `.git/`.

## Pipeline (8 steps)

### Step 1 — Architecture sketch (Claude) → user check-in

Claude proposes an architecture sketch. Cover: file/module decomposition, key interfaces, data flow, dependency choices. If 2+ sketches are roughly equal, present them and ask the user to pick a direction. Otherwise present one and proceed.

**Wait for user to confirm direction before continuing.** This is the user's only scheduledcheckpoint point in the entire pipeline.

### Step 2 — Plan v1 with interface contract (Claude)

Expand the chosen sketch into a written plan. **Must include the interface contract**, written at the top:

```
parse_pdf(path: str) → ParsedDoc
  errors: FileNotFound, ParseError
ParsedDoc: { title, sections[], metadata }
render_html(doc: ParsedDoc) → str  (HTML5)
```

Function signatures, parameters, return types, and error conditions for every cross-module function. Plan also includes test strategy (what behaviors verified, by whom) and implementation order.

**Do NOT write code yet.** Plan only.

### Step 3 — Codex reviews the plan

Send Codex the v1 plan. Codex returns a structured review:

- **Agreed parts** — quick acknowledgment.
- **Disagreed parts** — concrete reasoning + suggested revisions.
- **Missing considerations** — edge cases, test gaps, integration risks.

Codex MUST NOT write code in this step. Plan review only.

### Step 4 — Consensus loop v2..vN (Claude ↔ Codex)

For each disagreement Codex raised:

- **Accept** → revise plan accordingly.
- **Counter-argue** → explain why Codex's revision is wrong; send back to Codex.

**Hard ceiling: 3 rounds.** If round 3 ends without consensus, stop the loop and escalate to the user with a tiebreak card:

```
Disagreement: <one-sentence summary>
Codex position: <stance + 1 reason>
Claude position: <stance + 1 reason>
My lean: <one sentence>
Your call: A / B / "neither, here's a third option"
```

This is the **only mid-pipeline user interruption** outside of step 1.

Once consensus is reached, proceed straight to step 5. **No additional user approval gate.**

### Step 5 — Codex implements

Send Codex the locked plan + interface contract. Codex implements in sandbox.

While Codex works:

- **Long tasks emit heartbeats** — every 3-5 minutes, Codex appends a status line to the worklog so the orchestrator can confirm it isn't stalled.
- **Use `Monitor` to tail the worklog**. Don't trust "I'll notify you" callbacks — file-based polling is the source of truth.

**Local errors** (syntax, import, typo) → Codex iterates in place. Does not trigger arollback.

### Step 6 — Codex code-level testing

Codex runs unit and integration tests against its own implementation. Reports pass/fail in the worklog.

**If unit tests fail** → Codex iterates code in place, re-runs tests. Does not trigger arollback. Only escalates when stuck after 2 attempts on the same failure.

### Step 7 — Claude behavior-level verification

**Do not skip this step even if Codex's tests passed.** Codex's tests verify code-level correctness; Claude must verify user-perceptible behavior.

- Pull Codex's diff. Read the code. Look for: drift from the plan, dead code, contradictions, over-engineering signals.
- Run a behavior check appropriate to the project:
  - UI changes → click through in a real browser.
  - Script → run it once, inspect real output.
  - API → make a real call, verify the response.
  - Data pipeline → run on a real sample, look at the rows.

**If verification fails, route based on root cause:**

| Symptom | Route to |
|---|---|
| Interface is correct, implementation is wrong (code bug) | **Step 5** with a specific bug report |
| Plan didn't account for this scenario (interface insufficient or design gap) | **Step 4** with new constraints, redo consensus |
| Cannot determine root cause | Delegate diagnosis to Codex; route based on Codex's diagnosis |

**Anti-loop rule:** if the same failure triggers arollback three times in a row, escalate to the user with a clear summary instead of looping again.

### Step 8 — Claude commits

Codex writes its commit intent to `.claude/pending-commits.jsonl` (see [PROTOCOLS.md](../../PROTOCOLS.md) for the format). Claude:

1. Reads the intent.
2. Validates it (message accuracy, file scope, no destructive flags).
3. Executes the actual `git commit`.
4. Pushes only if explicitly authorized — pushing has its own safety rules outside this pipeline.

Then ends with a user-facing report.

## Hard rules

- **Only Claude does git.** Codex sandbox blocks writes to `.git/`. Don't waste a round trying.
- **Step 2 must include the interface contract.** Without locked signatures, "consensus" in step 4 is meaningless — Codex drifts in step 5, integration breaks in step 7.
- **3-round ceiling on consensus.** Then escalate to user.
- **Anti-loop onrollback:** 3 consecutive same-failurerollback → escalate to user.
- **Safety gates still apply.** If any step proposes a destructive operation, the orchestrator's standard safety check (e.g. a `risk-check` step in the user's CLAUDE.md) still fires. The pipeline does not bypass safety.
- **Codex writes code; Claude reviews every diff before commit.** Per the standard delegation discipline.

## User interaction timeline

```
Step 1 → user picks architecture direction      (scheduled, mandatory)
Step 2-6 → fully autonomous; user can leave
Step 4 round-3 fail → tiebreak card             (exception 1)
Step 7 third consecutiverollback → escalate         (exception 2)
Any destructive op → safety check               (orthogonal safety net)
Step 8 done → user-facing report
```

## Quick branch: bug-fix shortcut

If the user invokes this on a bug fix (not a new feature or system):

- **Skip steps 1-2** — no architecture discussion for a bug fix.
- **Step 3 alt** — if root cause is unclear, Codex investigates first and returns a diagnosis. If root cause is obvious, Claude proposes a minimal fix sketch.
- **Step 4 alt** — consensus on the fix sketch (1-2 rounds usually enough).
- **Steps 5-8** — same as full pipeline.

## User overrides (mid-pipeline)

The user can issue any of these to redirect:

- **`skip review`** — bypass Codex review, Claude solo from this point.
- **`I'll do it myself`** / **`skip codex`** — cancel the pipeline.
- **`run them in parallel`** — switch step 5 to worktree-parallel mode (only when slices are truly independent).
- **`skip grill`** — disable the grill-me path at step 1 if the user has it configured to fire automatically.

## When NOT to use this skill

- One-line tweaks, copy/style changes → solo, no Codex.
- Simple bug fixes with one obvious answer → simpler review pattern (Claude fixes, Codex audits at end if at all).
- Pure consultation / "what do you think about X?" → no pipeline, just discussion.
- User explicitly opts out (`skip codex`, `I'll do it myself`).

## Related docs

- [PROTOCOLS.md](../../PROTOCOLS.md) — file-based protocols (worklog, pending commits, venv).
- [README.md](../../README.md) — project overview, install, philosophy.
