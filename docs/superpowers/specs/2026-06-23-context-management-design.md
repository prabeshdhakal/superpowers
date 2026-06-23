# Context Management: Knowledge Graph + Project Memory

**Date:** 2026-06-23
**Status:** Approved

## Problem

Agents start every session cold. They re-read the codebase, re-read specs, and have no memory of decisions made in prior chat sessions that never made it into a file. This wastes tokens and loses context that matters.

Two distinct layers are missing:

| Layer | Problem |
|---|---|
| **Structural** | Agent doesn't know the codebase shape without re-reading it |
| **Episodic** | Agent doesn't know what was decided verbally in past sessions |

## Scope

This is a fork-local plugin (not contributed to superpowers core). Target harness: OpenCode only.

## What Gets Stored in Memory

Memory stores **project knowledge that exists in the agent's head but isn't in the repo**:

- Architectural decisions made in chat that weren't written into a spec or commit message
- Constraints stated verbally ("we can't upgrade that dependency", "this module is being deprecated")
- Design rationale that influenced a decision but didn't land in the spec doc
- Known gotchas discovered during sessions ("this API returns 200 on failure", "that test is flaky under load")
- Inter-module contracts agreed to verbally

**Explicit exclusions — never stored:**
- Anything already in a README, code file, spec doc, or plan doc (it's in the repo — read it)
- User preferences, communication style, or personal information of any kind
- Session-level working notes (those belong in the SDD progress ledger)

## Components

### 1. `context-load` skill

A new skill invoked at the **start of any session on an existing project** (or on first use within a session). Two steps, both optional and non-blocking:

**Step 1 — Knowledge graph check:**
If `.understand_anything/` exists in the repo root, call the `understand` MCP or read the graph summary to surface top-level module names, their responsibilities, and key relationships. If absent, skip silently.

**Step 2 — Project memory check:**
Call `search_nodes` on the `project-memory` MCP server with a query derived from the current task (e.g., the first user message or the current feature being worked on). Pull in matching entities and their observations. If no memory file exists yet, note it will be initialized on first write and continue.

The skill does **not**:
- Load the full graph wholesale
- Surface recent spec/plan files (the user references those explicitly when needed)
- Store anything itself — that is the write-back step in `brainstorming`

### 2. Hook into `brainstorming` skill

Two additive insertions:

**Before "Explore project context" step:**
> Before reading code files, invoke `context-load` to surface the knowledge graph summary and relevant project memory for this repo.

**After "Write design doc" step:**
> Write-back: review the session conversation for any architectural decisions, constraints, or gotchas that were stated verbally and are not already captured in the spec doc or any existing repo file. Write these to project memory using `create_entities` or `add_observations`. Never write user preferences or personal information.

### 3. Hook into `writing-plans` skill

One additive insertion at the top of the Overview section:
> Before writing the plan, invoke `context-load` if it has not already been invoked in this session. Use the knowledge graph and project memory to inform which files to reference and which gotchas to call out in task steps.

### 4. Per-repo OpenCode MCP config

`server-memory` is declared in an `opencode.json` at the **project root** of each repo that uses this plugin. This gives each repo its own isolated memory file.

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

`.superpowers/memory.jsonl` is added to `.gitignore` alongside `.superpowers/sdd/`. The file is local to the machine and never committed.

## Data Flow

```
Session start
    └─ context-load invoked
          ├─ .understand_anything/ present? → surface module summary
          └─ search_nodes(current task) → pull relevant entities

During brainstorming
    └─ agent uses surfaced context instead of re-reading code

After spec written
    └─ write-back: new decisions/constraints → add_observations / create_entities

Session start (next time)
    └─ context-load → search_nodes returns those observations
```

## Memory Storage

- **File location:** `<repo-root>/.superpowers/memory.jsonl`
- **Server:** `@modelcontextprotocol/server-memory` via npx (no install required)
- **Format:** JSONL knowledge graph (entities, relations, observations)
- **Gitignored:** yes — machine-local, never committed
- **Isolation:** per-repo by virtue of per-repo `opencode.json` config

## What Is Not in Scope

- Global/cross-repo memory
- Semantic/vector search (server-memory uses keyword search; sufficient for this use case)
- Automatic memory loading on every agent turn (on-demand via skill only)
- Spec/plan file surfacing at session start (user-initiated)
- Upstream contribution to superpowers core (fork-local only)

## Files Changed

| File | Change |
|---|---|
| `skills/context-load/SKILL.md` | New skill |
| `skills/brainstorming/SKILL.md` | Two additive hook insertions |
| `skills/writing-plans/SKILL.md` | One additive hook insertion |
| `opencode.json` (repo root) | New file — MCP server config for this repo |
| `.gitignore` | Add `.superpowers/memory.jsonl` |
