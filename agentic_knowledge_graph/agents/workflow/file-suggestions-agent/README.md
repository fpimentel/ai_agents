Project Overview

A focused utility module for working with a Neo4j-backed agent workflow. This folder contains a small Neo4j wrapper, a couple of ADK-compatible helper tools for sampling files and running agents, and minimal environment helpers. This README documents only what's present in the codebase — no feature assumptions or extrapolations.

Short summary of what's actually implemented

- `neo4j_for_adk.Neo4jForADK` (instantiated as `graphdb`) —
  - Exposes `send_query(cypher_query, parameters=None)` — runs a Cypher query and returns a ADK-friendly JSON-like dict via `result_to_adk`.
  - Exposes `get_driver()` and `close()`.
  - `get_import_directory()` currently returns the literal string `"../../neo4j/import"` (it does not inspect the DB configuration).
  - A module-level `graphdb = Neo4jForADK()` instance is created and registered to close at process exit.

- `tools.sample_file(file_path: str, tool_context)` —
  - Attempts to read the file at (import_dir / file_path) as text and returns up to the first 100 lines.
  - Returns a dict with `status: 'success'` and a `content` field on success, or `status: 'error'` with `error_message` on failure.
  - Note: the function assumes a `ToolContext` (from `google.adk.tools`) is provided; if `approved_files` or `approved_user_goal` are missing from state, helper functions in the same file return an error payload.

- `helper.py` —
  - `get_openai_api_key()` and `get_neo4j_import_dir()` read respective environment variables after calling `load_env()`.
  - `AgentCaller` and `make_agent_caller(...)` are small wrappers around `google.adk` runner/session types to run an ADK agent asynchronously and extract the final response.


Agent: purpose, inputs, outputs
--------------------------------

This project provides building blocks (not a full packaged agent). The following are the concrete contracts and behavior implemented in the codebase.

- Purpose
  - Provide small utilities to run an ADK agent turn (`helper.AgentCaller`), query a Neo4j database (`Neo4jForADK`), and sample file contents from a configured import directory (`tools.sample_file`). These pieces are intended to be composed by a higher-level agent or script.

- Neo4jForADK.send_query(cypher_query: str, parameters: dict | None) -> dict
  - Input: a Cypher query string and optional parameters dict.
  - Output (success): dict with keys `status: 'success'` and `query_result`, where `query_result` is a list of record dictionaries (converted from Neo4j records).
  - Output (error): dict with keys `status: 'error'` and `error_message` describing the failure.
  - Error modes: connection or query exceptions are caught and returned as an error dict; the caller should check `status` before using `query_result`.

- tools.sample_file(file_path: str, tool_context: ToolContext) -> dict
  - Input: `file_path` (path relative to Neo4j import directory) and a `ToolContext` from `google.adk.tools`.
  - Behavior: resolves the import directory via `graphdb.get_import_directory()` and attempts to open the file as UTF-8 text, reading up to the first 100 lines.
  - Output (success): dict with `status: 'success'` and `content` containing the sampled text.
  - Output (error): dict with `status: 'error'` and `error_message` (for missing file, IO errors, or missing import directory).

- helper.AgentCaller.call(query: str, verbose: bool=False) -> str
  - Input: a textual query to send to the ADK agent.
  - Behavior: runs the agent asynchronously via `Runner.run_async`, iterates agent events, and captures the event marked by `is_final_response()` (prefers the agent's authored final message). It prints progress when `verbose=True`.
  - Output: the final response text (string). If the agent produces no final textual response, the function returns the default string "Agent did not produce a final response." or an escalation message if present in the event.

- make_agent_caller(agent: Agent, initial_state: Optional[dict]) -> AgentCaller
  - Input: an ADK `Agent` object and an optional initial session state dict.
  - Output: an `AgentCaller` instance that is ready to call the agent with `AgentCaller.call`.

Edge cases and notes
- `neo4j_for_adk` creates a module-level `graphdb` on import; ensure environment variables for Neo4j are set before importing to avoid connection errors.
- `get_import_directory()` currently returns a hard-coded path (`"../../neo4j/import"`); if the resolved path doesn't exist, `tools.sample_file` will return an error.
- Errors are surfaced as dicts (for tools/Neo4j) or as default strings (for the AgentCaller); callers should check return shapes and `status` fields.


Agent: `file_suggestion_agent` (concrete notebook agent)
---------------------------------------------------

This repository includes a notebook-defined agent called `file_suggestion_agent` (created in the notebook as `file_suggestion_agent_v1`). The README now documents the agent's concrete behavior as defined in `file_suggestion_agent.ipynb`.

- Agent identity
  - Name in the notebook: `file_suggestion_agent_v1` (constructed via `Agent(name="file_suggestion_agent_v1", ...)`).
  - Description: "Helps the user select files to import." The agent uses an instruction string (`file_suggestion_agent_instruction`) that guides it to suggest files for constructing a knowledge graph.

- High-level purpose
  - Review available files in a Neo4j import directory, evaluate relevance to an approved user goal (a description of the kind of graph), and produce a curated list of suggested files for import.

- Tools the agent uses (as defined in the notebook)
  - `get_approved_user_goal` — reads the approved goal from session state.
  - `list_available_files` — enumerates files under the Neo4j import directory and stores them in state under the key `ALL_AVAILABLE_FILES`.
  - `sample_file` — reads up to 100 lines of a file (relative to import dir) to help determine relevance.
  - `set_suggested_files` / `get_suggested_files` — set and retrieve the `SUGGESTED_FILES` list in session state.
  - `approve_suggested_files` — move `SUGGESTED_FILES` into `APPROVED_FILES` in state when the user confirms.

- Inputs (what the agent expects)
  - The notebook runs the agent via `make_agent_caller(file_suggestion_agent, initial_state={...})` and passes an `initial_state` that should include `approved_user_goal` (dict with keys like `kind_of_graph` and `description`).
  - At runtime the agent calls tools; `sample_file` and `list_available_files` operate on the filesystem path returned by `get_neo4j_import_dir()`.

- Outputs (what the agent produces)
  - Session state entries (made available via the runner's session service):
    - `ALL_AVAILABLE_FILES` — list of files discovered under the import dir.
    - `SUGGESTED_FILES` — the agent's curated list of files to consider for import.
    - `APPROVED_FILES` — after user confirmation, the approved set saved by `approve_suggested_files`.
  - The agent also produces a final textual response (the standard ADK final message) summarizing the suggestions or next steps.

- Typical flow (as showcased in the notebook)
  1. A caller constructs `file_suggestion_caller = await make_agent_caller(file_suggestion_agent, {"approved_user_goal": {...}})`.
  2. Caller runs: `await file_suggestion_caller.call("What files can we use for import?")`.
  3. The agent uses `list_available_files`, samples files via `sample_file` as needed, and sets `SUGGESTED_FILES` via `set_suggested_files`.
  4. The user (or a scripted caller) can then approve with `await file_suggestion_caller.call("Yes, let's do it")`, after which `APPROVED_FILES` is set.

Note: the agent is defined inside a Jupyter notebook; it isn't exported as a standalone Python module in this folder. To use it programmatically, follow the notebook pattern: import the agent object or re-create it in code and use `make_agent_caller` / `AgentCaller` to run it.



Environment / dependencies

- The code imports: `neo4j`, `python-dotenv`, and `google.adk` / `google.genai` packages. Install dependencies via your project's `requirements.txt`. Note that the `requirements.txt` in this folder contains a single line referencing `../requirements.txt` — ensure you install the actual requirements file listed there (or install dependencies globally in your venv).
Example (local install in this folder):

```bash
# from the repository root or this directory (adjust path as necessary)
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

If the `requirements.txt` points at `../requirements.txt`, run `pip install -r ../requirements.txt` instead.


Environment variables used by the code (observed in source)

- NEO4J_URI — URI for Neo4j (e.g., bolt://localhost:7687)

Minimal usage examples (based on actual functions/classes)

1) Run a Cypher query and get the ADK-shaped result:

```python
from neo4j_for_adk import graphdb

resp = graphdb.send_query("MATCH (n) RETURN count(n) AS c")
print(resp)  # { 'status': 'success', 'query_result': [...] } or an error dict

graphdb.close()
```

2) Sample a file via `tools.sample_file` (note: requires a `ToolContext` instance):

```python
from google.adk.tools import ToolContext
from tools import sample_file

# Construct or obtain a ToolContext (this example assumes you have one available)
# tool_context = ...

# result = sample_file('path/relative/to/import/dir.txt', tool_context)
# print(result)
```

3) Use `helper.AgentCaller` to run an ADK agent asynchronously (pseudo-code):

```python
import asyncio
from helper import make_agent_caller

# agent = <your ADK Agent object>
# caller = await make_agent_caller(agent, initial_state={})
# await caller.call("Hello agent")
```


Known notes / concrete observations (no guessing)

- `neo4j_for_adk.get_import_directory()` returns the hard-coded path `"../../neo4j/import"`.
Potential quick fixes / suggestions (explicit and minimal)

- If you see NameError for `Path` or `islice` when calling `sample_file`, add these imports at the top of `tools.py`:
```python
from pathlib import Path
from itertools import islice
```

- If you want `get_import_directory()` to return a configurable value, update `neo4j_for_adk.get_import_directory()` to read an environment variable (for example `NEO4J_IMPORT_DIR`) instead of returning the literal string.

If you'd like I can now:

- Update `neo4j_for_adk.get_import_directory()` to read from an environment variable and fall back to the current literal.
Tell me which of those you'd like and I'll implement it. 

