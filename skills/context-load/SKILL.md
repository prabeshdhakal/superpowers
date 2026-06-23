---
name: context-load
description: "Load project context at session start: check for a knowledge graph and query project memory for information relevant to the current task. Invoke before reading code files."
---

# Context Load

Load available project context before reading code. Two sources, both optional. Skip silently if either is absent.

## Session Guard

If you have already invoked this skill in the current session, do not invoke it again. Once per session is enough.

## Step 1: Knowledge Graph

Check whether `.understand-anything/` exists in the repo root.

```bash
ls .understand-anything/ 2>/dev/null && echo "EXISTS" || echo "ABSENT"
```

**If ABSENT:** Skip to Step 2.

**If EXISTS:** Read `.understand-anything/summary.md` if present, otherwise read the first file found under `.understand-anything/` that contains a module or component listing. Surface the module names and one-line responsibilities to yourself. Do not read all files under `.understand-anything/` — one summary file is enough.

```bash
ls .understand-anything/
```

Read whichever of these exists (check in order, stop at first hit):
- `.understand-anything/summary.md`
- `.understand-anything/graph.md`
- `.understand-anything/index.md`

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
- Do not read every file under `.understand-anything/`
- Do not store anything during context-load — this skill only reads
- Do not invoke this skill more than once per session
