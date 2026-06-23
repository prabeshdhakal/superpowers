# Context Management: Knowledge Graph + Project Memory — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `context-load` skill and hook it into `brainstorming` and `writing-plans` so agents start sessions with relevant project context instead of re-reading the codebase from scratch.

**Architecture:** A new `context-load` skill checks for a `.understand_anything/` knowledge graph and queries a per-repo MCP memory server at session start. `brainstorming` invokes it before exploring code and writes new project knowledge back to memory after writing the spec. `writing-plans` invokes it if not already done. Each repo opts in by adding an `opencode.json` that configures the memory MCP server pointing at `.superpowers/memory.jsonl`.

**Tech Stack:** Markdown skills (plain text), `@modelcontextprotocol/server-memory` (npx, no install), OpenCode `opencode.json` config.

## Global Constraints

- Zero new npm dependencies in the superpowers package itself — `server-memory` runs via `npx -y` at the repo level, not installed into superpowers
- `.superpowers/` is already gitignored in this repo — no `.gitignore` change needed
- Skill files use YAML frontmatter (`name`, `description`) followed by Markdown body — match this format exactly
- All skill insertions are additive only — do not restructure or reword existing skill content outside the insertion points
- `opencode.json` at repo root uses the OpenCode config schema: `https://opencode.ai/config.json`
- No placeholders in any plan step

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `skills/context-load/SKILL.md` | **Create** | New skill: check knowledge graph + query project memory |
| `skills/brainstorming/SKILL.md` | **Modify** | Insert context-load hook at step 1; insert write-back step after step 6 |
| `skills/writing-plans/SKILL.md` | **Modify** | Insert context-load hook at top of Overview section |
| `opencode.json` | **Create** | Per-repo MCP server config for project memory |

---

## Task 1: Create the `context-load` skill

**Files:**
- Create: `skills/context-load/SKILL.md`

**Interfaces:**
- Consumes: nothing (entry point skill)
- Produces: a skill that agents invoke before reading code; calls MCP tools `search_nodes` (from `project-memory` server) and optionally reads `.understand_anything/` summary

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/context-load
```

- [ ] **Step 2: Verify the directory exists**

```bash
ls skills/context-load
```

Expected: empty directory (no error)

- [ ] **Step 3: Write the skill file**

Create `skills/context-load/SKILL.md` with this exact content:

```markdown
---
name: context-load
description: "Load project context at session start: check for a knowledge graph and query project memory for information relevant to the current task. Invoke before reading code files."
---

# Context Load

Load available project context before reading code. Two sources, both optional. Skip silently if either is absent.

## Session Guard

If you have already invoked this skill in the current session, do not invoke it again. Once per session is enough.

## Step 1: Knowledge Graph

Check whether `.understand_anything/` exists in the repo root.

```bash
ls .understand_anything/ 2>/dev/null && echo "EXISTS" || echo "ABSENT"
```

**If ABSENT:** Skip to Step 2.

**If EXISTS:** Read `.understand_anything/summary.md` if present, otherwise read the first file found under `.understand_anything/` that contains a module or component listing. Surface the module names and one-line responsibilities to yourself. Do not read all files under `.understand_anything/` — one summary file is enough.

```bash
ls .understand_anything/
```

Read whichever of these exists (check in order, stop at first hit):
- `.understand_anything/summary.md`
- `.understand_anything/graph.md`
- `.understand_anything/index.md`

## Step 2: Project Memory

Query the `project-memory` MCP server for nodes relevant to the current task.

The current task is the most recent user message or, if at session start, the first user message of the session.

Call:
```
search_nodes(query: "<3-5 word summary of the current task>")
```

Example: if the user said "add rate limiting to the API", call `search_nodes(query: "rate limiting API")`.

**If the `project-memory` MCP tool is unavailable** (server not configured in this repo's `opencode.json`): skip silently and note that project memory is not configured for this repo.

**If `search_nodes` returns no results:** note that no relevant project memory exists yet for this task. Memory will be initialized on first write-back.

**If `search_nodes` returns results:** surface the entity names and their observations to yourself. These are project-specific facts that are not in the codebase — treat them as ground truth alongside the code.

## What Not To Do

- Do not call `read_graph` — it loads the entire graph and wastes context
- Do not read every file under `.understand_anything/`
- Do not store anything during context-load — this skill only reads
- Do not invoke this skill more than once per session
```

- [ ] **Step 4: Verify the file exists and has correct frontmatter**

```bash
head -5 skills/context-load/SKILL.md
```

Expected output:
```
---
name: context-load
description: "Load project context at session start: check for a knowledge graph and query project memory for information relevant to the current task. Invoke before reading code files."
---
```

- [ ] **Step 5: Commit**

```bash
git add skills/context-load/SKILL.md
git commit -m "feat(skill): add context-load skill for knowledge graph + project memory"
```

---

## Task 2: Hook `context-load` into `brainstorming`

**Files:**
- Modify: `skills/brainstorming/SKILL.md` (lines 24 and 29–30 are the insertion points)

**Interfaces:**
- Consumes: `context-load` skill (Task 1)
- Produces: modified `brainstorming` skill with two new steps in the checklist

Two insertions:

1. **After checklist item 1** ("Explore project context"): add a sub-note that `context-load` must be invoked before reading files.
2. **After checklist item 6** ("Write design doc"): add a new step 7 for memory write-back, shifting the existing steps 7–9 to 8–10.

- [ ] **Step 1: Read the current checklist section to confirm line numbers**

```bash
grep -n "Explore project context\|Write design doc\|Spec self-review\|User reviews written spec\|Transition to implementation" skills/brainstorming/SKILL.md
```

Expected output (approximate line numbers):
```
24:1. **Explore project context** — check files, docs, recent commits
29:6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
30:7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
31:8. **User reviews written spec** — ask user to review the spec file before proceeding
32:9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 2: Insert the context-load hook at checklist item 1**

In `skills/brainstorming/SKILL.md`, find this exact line:

```
1. **Explore project context** — check files, docs, recent commits
```

Replace it with:

```
1. **Explore project context** — invoke `context-load` skill first, then check files, docs, recent commits. `context-load` surfaces the knowledge graph summary and relevant project memory so you read code with context instead of from scratch.
```

- [ ] **Step 3: Insert the memory write-back step and renumber**

Find this exact block (items 6–9):

```
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

Replace it with:

```
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Memory write-back** — review the session conversation for any architectural decisions, constraints, or gotchas stated verbally that are not already captured in the spec doc or any existing repo file. Write these to project memory using `create_entities` or `add_observations` on the `project-memory` MCP server. Never write user preferences or personal information. If `project-memory` is not configured for this repo, skip silently.
8. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
9. **User reviews written spec** — ask user to review the spec file before proceeding
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 4: Verify the checklist now has 10 items and the two insertions are correct**

```bash
grep -n "^\([0-9]\+\)\." skills/brainstorming/SKILL.md | head -12
```

Expected: 10 numbered items, item 1 mentions `context-load`, item 7 mentions `Memory write-back`.

- [ ] **Step 5: Verify no other content was changed**

```bash
git diff skills/brainstorming/SKILL.md
```

Review the diff. Only two regions should be changed: the item 1 line and the items 6–9 block. Nothing else.

- [ ] **Step 6: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): hook context-load at start and add memory write-back after spec"
```

---

## Task 3: Hook `context-load` into `writing-plans`

**Files:**
- Modify: `skills/writing-plans/SKILL.md` (insertion after the Overview opening paragraph)

**Interfaces:**
- Consumes: `context-load` skill (Task 1)
- Produces: modified `writing-plans` skill with a context-load hook in the Overview

- [ ] **Step 1: Read the current Overview section to confirm the insertion point**

```bash
grep -n "Write comprehensive\|Announce at start\|Context:" skills/writing-plans/SKILL.md
```

Expected output:
```
10: Write comprehensive implementation plans assuming ...
14: **Announce at start:** "I'm using the writing-plans skill to create the implementation plan."
16: **Context:** If working in an isolated worktree ...
```

- [ ] **Step 2: Insert the context-load hook**

In `skills/writing-plans/SKILL.md`, find this exact line:

```
**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."
```

Replace it with:

```
**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context load:** Before writing the plan, invoke `context-load` if it has not already been invoked in this session. Use the knowledge graph and project memory to inform which files to reference and which gotchas to call out in task steps.
```

- [ ] **Step 3: Verify the insertion is correct and nothing else changed**

```bash
git diff skills/writing-plans/SKILL.md
```

Review the diff. Only the two added lines after "Announce at start" should appear.

- [ ] **Step 4: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): invoke context-load before writing the plan"
```

---

## Task 4: Add per-repo OpenCode MCP config

**Files:**
- Create: `opencode.json` (repo root)

This file configures the `project-memory` MCP server for this repo. It is also the template other repos will copy when adopting this plugin.

**Interfaces:**
- Consumes: `@modelcontextprotocol/server-memory` via npx
- Produces: `project-memory` MCP tools available to the agent in OpenCode sessions opened in this repo

- [ ] **Step 1: Verify no `opencode.json` already exists at repo root**

```bash
ls opencode.json 2>/dev/null && echo "EXISTS" || echo "ABSENT"
```

Expected: `ABSENT`

- [ ] **Step 2: Create `opencode.json`**

Create `opencode.json` at the repo root with this exact content:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "project-memory": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-memory"],
      "environment": {
        "MEMORY_FILE_PATH": ".superpowers/memory.jsonl"
      }
    }
  }
}
```

- [ ] **Step 3: Verify the file parses as valid JSON**

```bash
node -e "JSON.parse(require('fs').readFileSync('opencode.json', 'utf8')); console.log('valid')"
```

Expected: `valid`

- [ ] **Step 4: Verify `.superpowers/` is already gitignored (covers `memory.jsonl`)**

```bash
grep "\.superpowers" .gitignore
```

Expected: `.superpowers/` — already present, no change needed.

- [ ] **Step 5: Commit**

```bash
git add opencode.json
git commit -m "feat(opencode): add project-memory MCP server config"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Covered by |
|---|---|
| `context-load` skill: knowledge graph check | Task 1 Step 3 (`.understand_anything/` check) |
| `context-load` skill: project memory query via `search_nodes` | Task 1 Step 3 (`search_nodes` call) |
| `context-load` skill: session guard (invoke once only) | Task 1 Step 3 (Session Guard section) |
| `context-load` skill: no `read_graph`, no wholesale loading | Task 1 Step 3 (What Not To Do) |
| `brainstorming` hook: invoke context-load before exploring | Task 2 Step 2 |
| `brainstorming` hook: write-back after spec | Task 2 Step 3 (item 7) |
| `writing-plans` hook: invoke context-load if not done | Task 3 Step 2 |
| Per-repo `opencode.json` with `MEMORY_FILE_PATH` | Task 4 Step 2 |
| `.superpowers/memory.jsonl` gitignored | Task 4 Step 4 (already covered by `.superpowers/`) |
| No user preferences stored | Task 1 Step 3 (What Not To Do) + Task 2 Step 3 (item 7 explicit exclusion) |

All spec requirements covered. No gaps.

**Placeholder scan:** No TBDs. All steps contain exact content (file contents, exact commands, expected outputs).

**Type consistency:** No shared types across tasks — each task produces a Markdown file or JSON config. No consistency issues.
