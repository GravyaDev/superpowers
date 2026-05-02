---
name: executing-plans
description: Use when you have a written implementation plan to execute in a separate session with review checkpoints
---

# Executing Plans

<KLOUDIFY-AUTO-CONFIRM>
In Kloudify-flavored projects (those with `.claude/kloudify/`), this skill operates in **silent-checkpoint mode**:

- Inter-task pauses for "review this checkpoint?" are removed.
- After each task completes, the AI emits an **observability log line** in chat in the form: `✓ Task N: <title> — <files changed> (+<insertions>/-<deletions>) — committed <sha>`. This preserves the user's ability to interrupt if something looks wrong.
- The AI does NOT pause to ask permission to continue between tasks. The default is **proceed to next task**.
- Hard-stop conditions (the "When to Stop and Ask for Help" section below) are unchanged: missing dependencies, repeated verification failures, ambiguous instructions, plan defects all halt execution and surface to the user.

**Per-task self-review specs** (applied silently after each task, before logging the ✓ line). Criteria are diversified by task class so the right thing is checked for each kind of task.

**Universal checks (every task)**:
1. *Verifier exit code*: did the verification command for this task return 0? If not → STOP, do not log success, raise the failure.
2. *Diff sanity*: did the commit actually touch the files the task said it would touch? If unrelated files changed unintentionally → STOP, raise the diff drift.
3. *Commit hygiene*: the commit message follows the project's commit-identity rule (Kloudify projects: `Co-Authored-By: Kloud <kloud@gravya.it>` per `.claude/knowledge-base.md`). If the trailer is wrong or missing on a project that requires it → STOP, fix the commit (amend) before proceeding.

**Task-class-specific checks**:

| Task class | Detection | Extra check |
|---|---|---|
| **TDD task** | task block contains "write the failing test" + "run to confirm fails" steps | Did a new test file land in this commit? If implementation diff has no `tests/` change → STOP, raise missing-test signal. |
| **Code edit task** | task modifies a code file (not docs / config) | Did the modified function/symbol still pass a syntax check? Run language-specific quick syntax verification (`bash -n`, `python -c "import ast; ast.parse(open('file').read())"`, `tsc --noEmit` if available) — if syntax fail → STOP. |
| **Doc edit task** | task modifies only `.md` / `.rst` / docs paths | Lint check: are all relative links in the new content resolvable? (`grep -E '\]\((?!http)' file.md` then test each path.) Broken internal links → STOP. |
| **Config edit task** | task modifies `.json`, `.yaml`, `.toml`, `settings*.json`, `*.conf` | Schema validity: parse the file with the appropriate tool (`jq . file.json`, `yq file.yaml`). Parse failure → STOP. |
| **Migration / DB task** | task modifies a migration file or DB schema | Reversibility: the task block must show a `down` / rollback script OR document why irreversible is acceptable. Missing rollback → STOP, raise missing-down signal. |
| **Security-sensitive task** | task touches auth, secrets, env vars, crypto | The Kloudify "SECURITY ACCEPTED" / "SECURITY MITIGATED" annotation: any new code path that bypasses a control needs a `// SECURITY ACCEPTED YYYY-MM-DD: <reason>` comment + entry in `.claude/security-acceptances.md`. Missing → STOP. |
| **Refactor task** | task says "rename X to Y" / "extract Y from X" / "consolidate X+Y" | All call sites updated: grep for the old name across the repo; if any non-test reference to the old name remains → STOP, raise stale-reference signal. |

Self-review is silent (no chat output) when all applicable checks pass. When any fail, the AI emits the failure inline AND the proposed remediation, then stops — same surface as a "blocker" today. The user can approve the proposed remediation, propose a different one, or unblock manually.

**Why diversify**: a TDD-task's success criterion (test landed) is meaningless for a doc-edit task; a config-edit task's success criterion (schema parses) is meaningless for a migration task. Generic checks miss class-specific failures. The cost of one extra grep / parse per task is negligible; the cost of a config edit that lands invalid YAML and breaks every downstream agent is hours.

Outside Kloudify projects this skill behaves exactly as upstream Superpowers documents — checkpoints with explicit user review between tasks (or batches).

Detection: `[ -d .claude/kloudify ]`. The check fires once at skill start.
</KLOUDIFY-AUTO-CONFIRM>

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

**Note:** Tell your human partner that Superpowers works much better with access to subagents. The quality of its work will be significantly higher if run on a platform with subagent support (such as Claude Code or Codex). If subagents are available, use superpowers:subagent-driven-development instead of this skill.

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create TodoWrite and proceed

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Complete Development

After all tasks complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use superpowers:finishing-a-development-branch
- Follow that skill to verify tests, present options, execute choice

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference skills when plan says to
- Stop when blocked, don't guess
- Never start implementation on main/master branch without explicit user consent

## Integration

**Required workflow skills:**
- **superpowers:using-git-worktrees** - Ensures isolated workspace (creates one or verifies existing)
- **superpowers:writing-plans** - Creates the plan this skill executes
- **superpowers:finishing-a-development-branch** - Complete development after all tasks
