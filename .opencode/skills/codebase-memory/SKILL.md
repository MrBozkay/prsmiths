---
name: codebase-memory
description: Query this repository through the codebase-memory knowledge graph (architecture, call traces, symbol snippets, structural search) before falling back to grep and file reads. Use when asked to inspect, map, trace or explain the codebase, find callers/callees of a function, get a module outline, check dead code or cycles, or when the MCP tools search_graph, get_architecture, trace_path, get_code_snippet, query_graph, detect_changes are available.
---

# codebase-memory for this repository

The repository is indexed in a codebase-memory knowledge graph (tree-sitter
symbols, call graph, routes). Prefer one graph query over many text scans.

## 1. Reach the graph

Try in order, stop at the first that works:

1. **MCP tools** in this session: `search_graph`, `get_architecture`,
   `trace_path`, `get_code_snippet`, `get_file_outline`, `query_graph`,
   `search_code`, `detect_changes`, `get_graph_schema`, `list_projects`.
2. **Local binary** (installed on this machine):
   `codebase-memory-mcp cli <tool> [flags]`
   or, if it is not on `PATH`, the binary the global OpenCode config points at
   (`/home/kurtar/Desktop/cbm/codebase-memory-mcp`).
3. **npm wrapper** (portable): `npx -y codebase-memory-mcp cli <tool> [flags]`.
4. Neither works, times out, or reports no project: say so once and fall back
   to `rg` and direct file reads. Graph availability is never a prerequisite.

Resolve the project id with `list_projects` (it mirrors the checkout path, e.g.
`prsmiths`) and pass it as `--project`.

## 2. Index when the graph is missing

```
codebase-memory-mcp cli index_repository --repo-path <checkout> --mode full
```

`--mode fast` for a smoke index; `full` also builds semantic embeddings. Verify
with `get_graph_schema` plus one focused `search_code` before relying on it.

## 3. Query order

1. `get_architecture --aspects all` — languages, packages, entry points,
   routes, hotspots, boundaries, layers, clusters, file tree. Add
   `--aspects cycles` for circular call dependencies (cycles are opt-in).
2. `search_graph` — symbols by `--name-pattern`, `--label`, degree filters;
   `--min-degree`/`--max-degree` find hotspots and isolated code.
3. `trace_path --function-name <f> --direction inbound|outbound|both --depth N`
   — who calls a function and what it calls.
4. `get_code_snippet --qualified-name <sym> --source-mode full` — source of one
   symbol; `get_file_outline --file-path <path>` — a whole file's symbols.
5. `query_graph` — Cypher-like `MATCH (f:Function)-[:CALLS]->(g:Function)`
   patterns. Single-hop relationships only; `WHERE` accepts expressions but not
   relationship patterns (filter in `MATCH` instead).
6. `detect_changes` — map a git diff to affected symbols and blast radius.
7. `search_code --pattern <text>` — graph-augmented grep when a name is unknown.

## 4. This repository's anchors

- Graph construction: `app/pipeline.py::build_workflow` (bench) and
  `build_workflow(models, live=True)` (live mode).
- Deterministic nodes: `app/nodes/{routing,verify,gate,finish,intake}.py`.
- Sandbox contract: `app/environment/base.py` + `docker.py` / `agent_runtime.py`,
  contract suite `tests/integration/test_environment_contract.py`.
- Entry points and every command: `AGENTS.md` (authoritative), README for the
  landing page, `docs/architecture.md` for the whole system, `docs/adr/` for
  decisions.

## 5. Boundaries

- Read-only: never run `index_repository` with another repository's path, and
  never delete projects you did not create (`delete_project` needs a reason).
- Held-out tasks (`bench/tasks/*-h[0-9][0-9]/`) are sealed by `AGENTS.md`
  hard rule 5: do not open, print or query their contents, in the graph or
  anywhere else.
- The graph is a cache, not a source of truth: when code and graph disagree,
  read the file.
