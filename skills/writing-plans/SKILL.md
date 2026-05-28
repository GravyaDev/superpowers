---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

<KLOUDIFY-AUTO-CONFIRM>
In Kloudify-flavored projects (those with `.claude/kloudify/`), this skill operates in **auto-confirm mode**:

- The full plan-authoring flow runs unattended without intermediate "looks right?" pauses. Each phase is announced for visibility, then the AI proceeds.
- The user retains the right to interrupt at any moment.
- The **Execution Handoff** at the end becomes 3-way (Subagent-Driven / Inline Execution / **Defer**) — the Defer option is Kloudify-specific and uses the Task Board + `/schedule` infrastructure to register a trigger condition for later auto-resume. See the "Execution Handoff" section below for full protocol.

**Per-phase self-review specs** (apply in auto-confirm mode only). Each phase has its own pass/fail criteria — these are NOT a generic "looks ok?" rubric. If a phase fails, fix inline and re-run the criterion before proceeding.

**Phase 0 — Prior-work surfacing (Kloudify work-graph, v2.55.0+)**
Before scoping, surface prior plans / open-threads / shipped-work that overlap this plan's topic, so you extend or supersede rather than author a duplicate (the `surface-prior-work-before-authoring-plan` universal rule). Run, if the work-graph scripts are installed:
```bash
if [ -d "$CLAUDE_PROJECT_DIR/.claude/kloudify/bin" ]; then WGB="$CLAUDE_PROJECT_DIR/.claude/kloudify/bin"; else WGB="$CLAUDE_PROJECT_DIR/bin"; fi
if [ -f "$WGB/reconcile-work-graph.py" ]; then
  python "$WGB/reconcile-work-graph.py" >/dev/null 2>&1 || true
  python "$WGB/surface-work-context.py" --author "<this plan's topic or title>" 2>/dev/null || true
fi
```
If prior work surfaces, review it before writing — extend or supersede the existing plan/thread. If a prior plan already covers this topic, STOP and surface to your human partner. Skips silently when the work-graph scripts are absent (pre-v2.55.0 Kloudify) or outside a Kloudify project. Detection: `[ -d .claude/kloudify ]` (same gate as the rest of this block).

**Phase A — Scope check**
1. *Multi-subsystem detection*: does the spec describe more than one independently-shippable subsystem (e.g. "auth + billing + notifications")? If yes, this is a decomposition failure that should have been caught during brainstorming. STOP and surface to the user — do NOT silently write a plan that conflates them.
2. *Single-subsystem confirmation*: state in one sentence which subsystem this plan covers. If you cannot state it crisply, the spec itself is ambiguous and step 5 self-review (in brainstorming) should have caught it. STOP and surface.

**Phase B — File structure**
1. *Path concreteness*: every file in the structure section has an absolute or repo-relative path. No `<path>` placeholders, no "the auth module" without a file.
2. *Responsibility statement*: each file gets a one-sentence statement of what it owns. If two files have nearly-identical statements, they should probably be one file — split is wrong.
3. *Existing-vs-new*: each file is marked Create / Modify / Delete. Modify entries name the line range or symbol being changed when known.

**Phase C — Task decomposition (per task)**
1. *Bite-sized*: each task is 2-5 minutes of work as documented. A "task" that is really 30 minutes of work hidden in one bullet is a fail — split.
2. *TDD shape*: TDD-shaped tasks have at least 5 steps (Write failing test → Run to confirm fails → Implement minimal → Run to confirm passes → Commit). Tasks missing the failing-test step or the confirm-fails step fail this check.
3. *Self-containment*: a task can be executed by a subagent reading only the task block (no need to scroll up to "see Task 3 for context"). If you find yourself writing "see Task N", inline the relevant context into THIS task.
4. *Verification command*: every task ends with at least one explicit verification command + expected outcome. "Manually verify" or "make sure it works" → fail; rewrite as a command.

**Phase D — Write tasks (post-write check)**
1. *No placeholders* in any task: TBD, TODO, "implement later", "fill in details", "similar to Task N", "add appropriate error handling" — all fail. (See "No Placeholders" section below for the full anti-pattern list.)
2. *Type consistency*: a function named `clearLayers()` in Task 3 is not named `clearFullLayers()` in Task 7. Run a quick name-collision scan across tasks before proceeding to Phase E.
3. *Code completeness*: every task that changes code shows the actual code in a fenced block. "Add a function that does X" without showing the function → fail.

**Phase E — Self-Review (existing 4-check + 1 extra in Kloudify mode)**
The Self-Review section below already documents 4 sub-checks (spec coverage, placeholder scan, type consistency, edge-case coverage). In auto-confirm mode the AI runs them inline AND adds:
5. *Reachability*: every file path, function name, and flag referenced in the plan exists in the codebase OR is created by an earlier task in this same plan. If a task references something undefined, either add a creating task or rewrite to remove the reference.

**Phase F — Execution Handoff**
This is **always user-gated** (Subagent / Inline / Defer choice). Auto-confirm does NOT collapse this gate — the choice between "implement now" and "defer" is operational, not procedural. The skill terminates either by handing off to executing-plans / subagent-driven-development OR by registering a deferred trigger and stopping. See the Execution Handoff section below.

**Cross-cutting**: at any point during phases A-E, if the user types a message in chat, treat it as a potential override. Read before continuing.

Detection: `[ -d .claude/kloudify ]`. Outside Kloudify projects this skill behaves exactly as upstream Superpowers documents — 2-way handoff, no defer, user-gated phase transitions where upstream defines them.
</KLOUDIFY-AUTO-CONFIRM>

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits
- Edge-case rows from the spec (Kloudify-flavored projects) become tasks in the plan; never silently drop them

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

**4. Edge-case coverage** (Kloudify-flavored projects only): If the spec contains a `## Edge cases triaged` section (per Kloudify edge-case-sweep mandate from brainstorming step 6), verify that every row marked `implemented` has a corresponding task in this plan that implements + tests it. List gaps. If a gap is found, the fix is to add a task — not to downgrade the row's status to `out-of-scope` (status downgrades require explicit user sign-off, since `out-of-scope` carries a documented trigger to revisit; downgrading silently loses that trigger). This is a SOFT check: the spec already enforced the HARD-GATE at brainstorming time. Here we're verifying the plan honors what the spec committed to. If the spec has no edge-case section (e.g. brainstorming was skipped or the project is non-Kloudify), this check is a no-op.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Execution Handoff

After saving the plan, offer execution choice. **In Kloudify-flavored projects** (those with `.claude/kloudify/`) the offer is **3-way** (Subagent / Inline / Defer). **Outside Kloudify projects** the offer remains 2-way (Subagent / Inline) for parity with upstream Superpowers.

### Kloudify projects (3-way handoff)

> "Plan complete and saved to `<path>` (commit `<sha>`). Three execution options:
>
> **1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
>
> **2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.
>
> **3. Defer** — Don't implement now. I'll register this plan as `TRIGGER-DRIVEN` in the Task Board with a trigger I'll propose based on what I know about the project. You can accept the proposal or counter-propose.
>
> Which approach?"

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Fresh subagent per task + two-stage review

**If Inline Execution chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:executing-plans
- Batch execution with checkpoints for review

**If Defer chosen:**
1. **Propose a trigger.** Pick from the 5-flavor taxonomy below based on signals in the project state. Lead with a single concrete proposal; mention 1-2 alternatives only if multiple are roughly equivalent. The user can accept ("ok"), counter-propose ("Tuesday instead"), or refine ("only after PR #45 merges, not the calendar date").

   **Trigger taxonomy** (Kloudify A2 — calendar | event | manual | deadline | custom):
   - **calendar** — fixed date/time. *Example proposal*: *"Propongo: lunedì 2026-05-04 09:00. Posso schedulare un agente background che apre la PR di implementazione."*
   - **event** — wait for an external state change to fire (PR merge, CI pass, deploy completion). *Example proposal*: *"Propongo: dopo che PR #45 merge su main. Verifico con `gh pr view 45 --json state` daily."*
   - **manual** — no automatic trigger; the plan stays in the board, the user picks it up when ready. *Example proposal*: *"Resta `TRIGGER-DRIVEN` nella Task Board, riprendilo tu con `/execute-plan <path>` quando vuoi."*
   - **deadline** — must be done by date X; if not started by then, escalates. *Example proposal*: *"Propongo: entro venerdì 2026-05-08. Se non avviato entro quel giorno, riemerge in `/start` come priorità top."*
   - **custom** — user describes the trigger in natural language. The AI translates it to a concrete mechanism (typically `/schedule` cron or a state-poll command). *Example*: user says *"quando il dashboard tornerà sotto 100 errori al giorno"* → AI proposes a daily polling agent that watches the metric and auto-resumes when threshold met.

2. **Update the Task Board.** Add or modify the row for this plan in the `## Plans` section:
   - **Status**: `TRIGGER-DRIVEN`
   - **Phase**: `0/N pending` (where N is the task count from this plan)
   - **Owner**: `claude` (or whoever the user assigns)
   - **Trigger / next**: the trigger expression in machine-parseable shape if possible
     - calendar: `2026-05-04 09:00`
     - event: `gh pr #45 merged`
     - deadline: `by 2026-05-08`
     - manual: `manual — user resume via /execute-plan`
     - custom: a one-line summary + a pointer to the scheduling artifact (e.g. `/schedule cron 'errors<100' → resumes plan`)

3. **Offer to schedule mechanically (Kloudify C2 protocol).** If the chosen trigger is **calendar**, **event**, or **deadline**, proactively offer to register a `/schedule` agent that fires when the trigger condition is met. If the trigger is **manual**, skip this step (no schedule needed). If the trigger is **custom**, propose the most plausible `/schedule` translation but explicitly flag the translation as best-effort.

   **Offer wording** (single message, after the Task Board update):
   > *"Trigger registered: `<trigger>`. Vuoi che schedulo un agente di background che <concrete action> al firing? (Yes / No / Modify trigger)"*

   On **Yes**: invoke `/schedule` (or the equivalent `ScheduleWakeup` mechanism) with the trigger as input. The agent's role at firing is to read the plan, then invoke `superpowers:subagent-driven-development` (the recommended path) — same as if the user had picked option 1 immediately.

   On **No**: leave the Task Board entry as-is. The user manually picks up the plan when ready (`/execute-plan <path>` or equivalent).

   On **Modify trigger**: re-propose. Loop until the user accepts or selects No.

4. **Confirm and stop.** End the writing-plans session with:
   > *"Plan deferred. Task Board updated, status `TRIGGER-DRIVEN`, trigger `<trigger>`. <schedule status if applicable.> The plan stays at `<path>` until the trigger fires or you resume it manually."*

   Do NOT invoke `executing-plans` or `subagent-driven-development`. The skill terminates here.

### Non-Kloudify projects (2-way handoff, upstream behavior)

After saving the plan, offer execution choice:

> "Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Two execution options:
>
> **1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
>
> **2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.
>
> Which approach?"

Same dispatch as before based on choice. No defer option — defer requires the Task Board / `/schedule` infrastructure, which is Kloudify-specific.
