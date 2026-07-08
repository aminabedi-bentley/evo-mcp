# PoC: Scalable MCP tooling — Tool Search vs. Code Mode

A runnable proof-of-concept for the [design doc](https://seequent.atlassian.net/wiki/x/_oA9kg).

It lets you **observe what the model sees** under three tool-exposure strategies, using a
self-contained demo server with mock Evo-flavored tools. **No Evo credentials or network access
required for interacting with the demo server.**

| Strategy | `list_tools()` returns | What it fixes |
|---|---|---|
| `none` | the full catalog (15 demo tools) | nothing (today's behavior) |
| `tool-search` | `search_tools`, `call_tool` | catalog bloat |
| `code-mode` | `search`, `get_schema`, `execute` | catalog bloat + round-trips + intermediate-result pollution |

## Files

- **`tool_strategy.py`** — the shared factory (`apply_strategy`, `ToolStrategy`,
  `SearchEngine`). Reads `MCP_TOOL_STRATEGY` / `MCP_SEARCH_ENGINE`. This is the *real* module wired
  into `src/mcp_tools.py` (imported from here), so the PoC exercises production wiring.
- **`demo_server.py`** — a FastMCP server with 15 mock Evo tools (tagged `read`/`write`/`admin`/
  `destructive`) that return canned data. Provides `build_demo_server(strategy, ...)` and an env-var
  `__main__` entrypoint.
- **`explore.ipynb`** — the interactive walkthrough. Run it top to bottom.

## Setup

```bash
# From the repo root
uv sync --extra poc
```

## Run the notebook

```bash
uv run --extra poc jupyter lab src/poc/explore.ipynb
```

The notebook covers:
1. What the model sees upfront (`list_tools`) in each mode, plus an approximate **context-cost**
   measurement of the upfront catalog.
2. `none` — the baseline (full catalog, one raw result per call).
3. `tool-search` — discovery on demand; BM25 vs. regex engines.
4. `code-mode` — a multi-step workflow collapsed into a single sandboxed `execute` that returns only
   the final answer.
5. **Guardrails** — hiding `destructive`-tagged tools from discovery via a `Visibility` transform.
6. **Against the live Evo MCP server (discovery only)** — the real ~46-tool catalog collapsing to
   2–3 synthetic tools, with real `search_tools` / `search` / `get_schema` calls. **No Evo
   credentials and no Evo API calls** — discovery reads the catalog only. The multi-step workflow is
   *presented* as the code the model would write, not executed.
7. A side-by-side summary table.

> Sections 1–5 use `demo_server.py` (deterministic, safe, no creds) so the executable workflow and
> destructive-guardrail cells actually run. Section 6 targets the real catalog but stays discovery-only
> so it needs no credentials and touches no live tenant.

## Run the demo server standalone

```bash
MCP_TOOL_STRATEGY=none        uv run python src/poc/demo_server.py
MCP_TOOL_STRATEGY=tool-search uv run python src/poc/demo_server.py
MCP_TOOL_STRATEGY=code-mode   uv run python src/poc/demo_server.py
```

## The same switch on the real server

The factory is wired into `src/mcp_tools.py`, applied after all tools are registered:

```bash
MCP_TOOL_STRATEGY=code-mode   uv run python src/mcp_tools.py   # 46 tools -> search/get_schema/execute
MCP_TOOL_STRATEGY=tool-search uv run python src/mcp_tools.py   # 46 tools -> search_tools/call_tool
# MCP_TOOL_STRATEGY unset / "none" -> unchanged behavior (default)
```

## Notes

- `code-mode` requires the `fastmcp[code-mode]` extra (pydantic-monty sandbox), already declared in
  the project dependencies.
- Inside the `code-mode` sandbox, `await call_tool(name, params)` returns `{"result": <value>}` — see
  the `["result"]` unwrapping in the notebook.
- These are stock FastMCP transforms (`>=3.1.0`; repo runs `3.3.1`). Code Mode is flagged
  **experimental** upstream. For production cloud use, harden the sandbox (isolated runtime, no
  network egress) as described in the design doc §7.
