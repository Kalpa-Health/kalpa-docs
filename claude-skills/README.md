# Claude Skills — Team Setup Guide

**Version:** 1.0
**Date:** 06 March 2026
**Maintained by:** Alex

---

## 1. What These Skills Are

Claude Code skills are plain markdown files that teach Claude how to behave in specific situations. They live in `~/.claude/commands/` on each developer's machine and apply globally to all Claude Code sessions.

This directory contains two shared skills:

- **`task-runner.md`** — plan execution engine. Trigger with: "run the plan", "continue the plan", "next task", or any reference to a `*-PLAN.md` file.
- **`markdown-style.md`** — document style guide for PRDs, plans, and structured docs. Trigger with: "create a plan", "draft a PRD", "write this up as a document".

---

## 2. Prerequisites

- [Claude Code](https://claude.ai/code) installed (`npm install -g @anthropic-ai/claude-code` or via the desktop app)
- `~/.claude/commands/` directory exists — Claude Code creates this automatically on first run

---

## 3. Installation

Clone `kalpa-docs` (if you haven't already):

```bash
git clone git@github.com:kalpa-health/kalpa-docs.git ~/Projects/WellMed/kalpa-docs
```

Symlink the skills into your Claude commands directory:

```bash
ln -s ~/Projects/WellMed/kalpa-docs/claude-skills/task-runner.md ~/.claude/commands/task-runner.md
ln -s ~/Projects/WellMed/kalpa-docs/claude-skills/markdown-style.md ~/.claude/commands/markdown-style.md
```

That's it. Skills take effect immediately — no restart needed.

---

## 4. Staying Up to Date

Because the files are symlinked, pulling the latest `kalpa-docs` automatically updates the skills:

```bash
cd ~/Projects/WellMed/kalpa-docs && git pull
```

No re-linking required.

---

## 5. How to Use

### 5.1 task-runner

Create a `[topic]-PLAN.md` file in your working directory (see the plan format in `task-runner.md` Section 3). Then tell Claude:

```
run the [topic] plan
```

Claude will execute AI tasks sequentially, log progress to `[topic]-PROGRESS.md`, and stop at human checkpoints. To resume after a checkpoint:

```
continue the [topic] plan
```

### 5.2 markdown-style

When creating or updating any structured document, Claude will automatically follow the style conventions. You can also invoke it explicitly:

```
draft a PRD for [feature]
write this up as a plan document
```

---

## 6. The PRD to Plan Workflow

```
[claude.ai console, Opus]        [Claude Code, local]
      |                                 |
      v                                 v
 Draft PRD                       Create Plan from PRD
 (markdown-style §9)    ---->    (markdown-style §8 + task-runner)
      |                                 |
      v                                 v
 kalpa-docs/development/        kalpa-docs/plans/[topic]/
   prds/[feature]-PRD.md          [topic]-PLAN.md
```

1. In console (Opus): `create a PRD for [feature]`
2. Save to `kalpa-docs/development/prds/[feature]-PRD.md`
3. In Claude Code: `create a plan from [feature]-PRD.md`
4. Run: `run the [topic] plan`
5. On completion: task-runner moves plan + progress to `kalpa-docs/plans/archive/`

---

## 7. Troubleshooting

**Skill not triggering**
Verify the symlink is not broken: `ls -la ~/.claude/commands/`. If the link target shows in red, the source path is wrong — re-run the `ln -s` command with the correct path to your kalpa-docs clone.

**Wrong clone location**
If you cloned kalpa-docs somewhere other than `~/Projects/WellMed/kalpa-docs`, adjust the path in the `ln -s` commands accordingly.
