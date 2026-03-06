---
name: task-runner
description: "Execute tasks from a structured plan document step by step, logging progress and stopping at human checkpoints. Plan files follow the naming convention *-PLAN.md (e.g., ci-hardening-PLAN.md) or plain PLAN.md. Use this skill whenever the user says 'run the plan', 'continue the plan', 'next task', 'ralph loop', 'execute the plan', 'pick up where we left off', or references any *-PLAN.md file by name. Also trigger when the user asks to 'step through tasks', 'work through the checklist', or 'run the next phase'. This skill drives autonomous task execution with human-in-the-loop checkpoints."
---

# Task Runner Skill

Execute tasks from a plan document, track progress in a paired progress file, and stop at human checkpoints.

## 1. How It Works

This skill reads a plan document, finds the next uncompleted task, executes it, logs the result, and moves to the next task. It stops when it hits a task marked `🔲 HUMAN` or when all tasks in the current phase are complete.

## 2. Plan Discovery

On each invocation, locate the plan file using this priority order:

2.1 If the user explicitly names a file (e.g., "run the ci-hardening plan"), look for `ci-hardening-PLAN.md` in the CWD.

2.2 If no file is named, scan CWD for files matching `*-PLAN.md` or exactly `PLAN.md`.
- If exactly one is found, use it.
- If multiple are found, list them and ask the user which to run. Do not guess.
- If none are found, tell the user no plan file was found and stop.

2.3 The progress file name is always derived from the plan file name:
- `foo-bar-PLAN.md` → `foo-bar-PROGRESS.md`
- `PLAN.md` → `PROGRESS.md`

Both files live in the same directory as each other (which should be the project root or the `kalpa-docs/plans/<name>/` directory).

## 3. Plan Document Format

Plan files use this structure:

```markdown
# Plan: [Project Name]

**Version:** 1.0
**Date:** DD Month YYYY
**Author:** Name
**ADR:** [optional link to ADR]
**Status:** Draft | Ready to execute | In Progress
**Branch:** (optional) feature/[branch-name] — if omitted, task-runner derives `feature/<plan-stem>` from the filename

## Related Docs
<!-- Scope the pre-flight check. List architecture docs, READMEs, ADRs, and specs relevant to this plan. -->
- `path/to/relevant-doc.md`
- `path/to/adr.md`

---

## Phase 1: [Phase Name]

### Task 1.1: [Task Title]
- **Type**: AI | HUMAN | AI+HUMAN_REVIEW
- **Input**: What files/context this task needs
- **Action**: Specific instructions for what to do
- **Output**: What files/artifacts this task produces
- **Acceptance**: How to verify the task succeeded
- **Notes**: Any additional context

### Task 1.2: [Task Title]
...

---
### 🔲 CHECKPOINT: [Description]
**Review**: What the human should check before continuing
**Resume**: What to tell Claude to continue (e.g., "continue the ci-hardening plan")
---

## Phase 2: [Phase Name]
...
```

Task types:
- **AI**: Claude executes autonomously
- **HUMAN**: Claude stops and waits for human action
- **AI+HUMAN_REVIEW**: Claude executes, then stops for human review before marking complete

## 4. Progress Document Format

The progress file is append-only. Claude creates it on first run if it doesn't exist.

```markdown
# Progress Log: [Plan Name]

## Session: [ISO timestamp]

### Phase 0: Pre-flight
- **Status**: ✅ DONE
- **Completed**: [timestamp]
- **Paths verified**: [list]
- **Issues**: None | [description]

### Task 1.1: [Task Title]
- **Status**: ✅ DONE | ⏭️ SKIPPED | ❌ FAILED | ⏸️ WAITING_HUMAN
- **Started**: [timestamp]
- **Completed**: [timestamp]
- **What was done**: [1-3 sentence summary]
- **Files modified**: [list of files created or changed]
- **Issues**: [any problems encountered, or "None"]
```

## 5. Phase 0: Pre-flight (Runs Once Per Plan)

Before executing any tasks, run Phase 0. If it already appears in PROGRESS.md as ✅ DONE, skip it entirely.

5.1 Verify all file paths referenced in the plan's **Input** fields actually exist. List any that don't.

5.2 Verify all ADR and doc links in the **Related Docs** and plan header are resolvable paths. List any that are broken.

5.3 Check that the plan's **Status** field is "Ready to execute". If it's "Draft", warn the user and stop.

5.4 If any verification fails, log it as ❌ FAILED and stop. Do not execute tasks on a broken plan. Report clearly what needs fixing.

5.5 If everything passes, log Phase 0 as ✅ DONE and proceed to Phase 1.

5.6 **Branch detection** — Determine the working branch:
- If the plan has a `**Branch:**` field: confirm with the user — "Plan specifies branch `<branch>`. Confirm this is correct before we proceed." Wait for confirmation before continuing.
- If the plan has no `**Branch:**` field: derive a name as `feature/<plan-stem>` (e.g., `cashier-standards-PLAN.md` → `feature/cashier-standards`). Announce it: "No branch specified — will use `feature/<derived-name>`."
- If git is not initialized in the repo, skip branch management entirely and note this in the log.

5.7 **Branch checkout** — Once the branch name is confirmed:
- If the branch already exists locally: `git checkout <branch>`
- If it doesn't exist locally: `git checkout -b <branch>`
- **Never execute tasks on `main` or `master`.** If the current branch is main/master after this step, stop and report an error.
- Log the branch name in the progress file under Phase 0.

## 6. Execution Rules

6.1 On each invocation, read the plan file and the progress file. Find the first task that does NOT have a corresponding entry in the progress file (or has a ❌ FAILED entry that should be retried).

6.2 If the next task is type `HUMAN`, do NOT execute it. Tell the user what they need to do and stop. Log status as `⏸️ WAITING_HUMAN`.

6.3 If the next task is type `AI`, execute it fully. Log the result. Then immediately proceed to the next task — keep going until you hit a HUMAN task, a CHECKPOINT, or the end of the current phase.

6.4 If the next task is type `AI+HUMAN_REVIEW`, execute it fully, log the result, then STOP and ask the human to review before continuing.

6.5 When you hit a `🔲 CHECKPOINT`, stop and tell the user what to review. Do not proceed past a checkpoint without explicit human instruction.

6.6 If a task fails, log it as ❌ FAILED with a clear description of what went wrong. Then STOP — do not continue past a failed task without human input.

6.7 At the start of each session, print a brief status summary:
```
📋 Plan: [Project Name]  ([filename])
🌿 Branch: [current branch]
📍 Current: Phase X, Task X.Y — [Task Title]
✅ Completed: N/M tasks
⏸️ Status: [Ready to execute / Waiting on human / Failed — needs review]
```

## 7. Important Behaviors

7.1 **Never modify the plan file.** The plan is the source of truth. If something in the plan is wrong, tell the human and stop.

7.2 **Always append to the progress file.** Never delete or overwrite previous entries. The log is an audit trail.

7.3 **Be explicit about file changes.** Every task that creates or modifies files must list them in the progress log.

7.4 **Stay in scope.** Each task has defined inputs, actions, and outputs. Don't do extra work beyond what the task specifies — the plan is sequenced deliberately.

7.5 **Commit after each task; push after each phase.** If git is initialized, commit after each completed AI task with message format: `plan: [Task X.Y] [brief description]`. When all tasks in a phase are complete (at a CHECKPOINT or phase boundary), run `git push origin <branch>` to back up progress to the remote. PRs are created manually — do not open them.

7.6 **On resume, summarize what happened.** If the user returns after a break, read the progress file and give a brief summary of where things stand before continuing.

## 8. Ralph Loop Integration

When used with the ralph-loop plugin, this skill drives the loop execution. The ralph loop repeatedly invokes Claude Code with the same prompt. This skill ensures each invocation picks up where the last one left off by reading the progress file.

Prompt template for ralph loop:
```
Read [name]-PLAN.md and [name]-PROGRESS.md. Execute the next available AI task.
After completing each task, update [name]-PROGRESS.md and commit.
Continue until you hit a HUMAN task, CHECKPOINT, or end of phase.
If all tasks are complete, output <promise>PHASE_COMPLETE</promise>.
If blocked on a human task, output <promise>WAITING_HUMAN</promise>.
```

## 9. First Run Setup

If the progress file doesn't exist when this skill is first invoked:

9.1 Discover the plan file per Section 2.

9.2 Validate the plan file exists and is parseable — count phases, tasks, and checkpoints.

9.3 Create the progress file with the header and session start timestamp.

9.4 Print the full status summary including total scope.

9.5 Run Phase 0 pre-flight (Section 5) before beginning task execution.

9.6 Begin executing from Task 1.1 (or the first AI task if 1.1 is HUMAN).
