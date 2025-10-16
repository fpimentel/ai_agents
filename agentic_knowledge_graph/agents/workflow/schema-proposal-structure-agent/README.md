
# Schema Proposal Structure Agent

This repository contains a small research/demo workspace that implements a structured schema proposal workflow for building knowledge graphs from approved import files. The primary artifact is a Jupyter notebook (`schema_proposal_structured.ipynb`) that shows how agents (powered by the Google ADK and an LLM) can propose and critique construction rules that transform CSV/text files into a property graph suitable for Neo4j.

The code here is meant for experimentation and learning rather than production use.

## High-level idea

The notebook implements a coordinated multi-agent pattern to propose a "construction plan" for a knowledge graph. The main flow is:

- Input: `approved_user_goal` (describes the target knowledge graph) and `approved_files` (list of CSV/text files that may be imported).
- Agents: a `schema_proposal_agent` proposes node and relationship construction rules; a `schema_critic_agent` validates and critiques them; a small loop coordinates refinement until the plan is acceptable.
- Tools: lightweight file inspection and Neo4j helper tools (see `tools.py`) are used by agents to read sample data, search files, and generate construction plan entries.
- Output: `approved_construction_plan` — a dictionary of construction rules (nodes and relationships) that can be used to import data into Neo4j.

## Main files

- `schema_proposal_structured.ipynb` — Notebook with the full lesson/demo and runnable cells that build the agents and run the proposal/critique loop.
- `helper.py` — Convenience helpers: environment loading, an `AgentCaller` wrapper to run ADK agents (async), and small utility getters for environment variables.
- `tools.py` — Tool functions used by agents (searching files, sampling file contents, proposing node/relationship constructions, Neo4j helper functions like `clear_neo4j_data`, `drop_neo4j_indexes`, and metadata/inspection tools).
- `requirements.txt` — Points to the top-level requirements file (this project references the parent `requirements.txt` so be sure to install the correct environment for the overall workspace).
- `agents.png`, `entire_solution.png`, `schema_proposal_structured_sequence.png` — Visual diagrams referenced in the notebook.

## Notebook overview (what each high-level section does)

- 6.1 Agent — Diagram and explanation of the multi-agent workflow (proposal, critic, and coordinator).
- 6.2 Setup — Imports, LLM setup (LiteLlm wrapper), logging, and environment loading.
- 6.3 Proposal Agent Prompts — Instruction templates and hints used to build the `schema_proposal_agent` prompt (role, hints, chain-of-thought directions).
- 6.4 Tool Definitions for Schema Proposal — Concrete implementations of tools the proposal agent uses, including `search_file`, `sample_file`, `propose_node_construction`, and `propose_relationship_construction`.
- 6.5 Define the Agent — Creation of the ADK `LlmAgent` instance wired to the tools and the instructions; `AgentCaller` is used to run it.
- 6.6 Critic Agent — Prompts and tools used by a critic agent to validate uniqueness, connectivity, and other design rules; critic returns either `valid` or `retry` with feedback.
- 6.7 & beyond — The notebook continues with the critic tool definitions, refinement loop orchestration, and examples of running the agents and inspecting the resulting `proposed_construction_plan`.

## Important modules & functions (quick reference)

- `helper.py`
	- `load_env()` — loads environment variables from a `.env` (uses `python-dotenv`).
	- `get_openai_api_key()` — convenience reader for `OPENAI_API_KEY`.
	- `get_neo4j_import_dir()` — convenience reader for `NEO4J_IMPORT_DIR`.
	- `AgentCaller` — async wrapper around an ADK `Runner` to call agents and extract final text responses.
	- `make_agent_caller(agent, initial_state={})` — create and initialize an `AgentCaller` with an in-memory session.

- `tools.py` (selected highlights)
	- `get_approved_user_goal(tool_context)` / `get_approved_files(tool_context)` — return state keys required by the agents.
	- `sample_file(file_path)` — returns up to the first 100 lines of a file in the Neo4j import directory for inspection.
	- `search_file(file_path, query)` — grep-like tool used to validate the presence/uniqueness of identifiers.
	- `propose_node_construction(...)` / `propose_relationship_construction(...)` — add construction rules into agent `tool_context.state` under the `proposed_construction_plan` key.
	- `approve_proposed_construction_plan(tool_context)` — move the proposed plan into `approved_construction_plan`.
	- Neo4j helpers: `neo4j_is_ready()`, `get_apoc_version()`, `get_neo4j_version()`, `clear_neo4j_data()`, `drop_neo4j_indexes()` (these call into a workspace helper module `neo4j_for_adk`).

## Environment variables

The notebook and helper modules expect a `.env` file (or process environment) with at least the following variables:

- `OPENAI_API_KEY` — API key for the LLM provider (the notebook uses the `LiteLlm` wrapper).
- `NEO4J_IMPORT_DIR` — path to the directory that contains CSV/text files to be inspected and potentially loaded into Neo4j. Many file tools operate relative to this directory.

Create a `.env` in the project or parent directory if one doesn't exist. Example:

```text
OPENAI_API_KEY=sk_...your_key_here...
NEO4J_IMPORT_DIR=/path/to/neo4j/import
```

Note: The project uses `python-dotenv` to locate and load `.env` automatically when `helper.load_env()` is called.

## Dependencies

This repository references the workspace-level `requirements.txt` (see `requirements.txt` which points to `../requirements.txt`). Install dependencies from the top-level file to ensure compatible versions across the workspace:

```bash
pip install -r ../requirements.txt
```

Key libraries used in the notebook and helpers include:

- google ADK / genai packages (LLM and agent abstractions used in the notebook)
- python-dotenv
- neo4j-related helper package (`neo4j_for_adk`) — a workspace helper that wraps Neo4j interactions

If you prefer a minimal quick run for the notebook without a full workspace install, you can pip-install at least `python-dotenv`, `neo4j`, and the Google ADK/GenAI client packages — but exact names and versions are determined by the parent `requirements.txt`.

## Quick run / examples

1) Create and activate a virtual environment (macOS / bash):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r ../requirements.txt
```

2) Start Jupyter and open the notebook:

```bash
jupyter lab
# open `schema_proposal_structured.ipynb` from the Jupyter UI
```

3) Example: Inspect available helper symbols from the repo root (quick check):

```bash
python -c "import helper, tools; print('helper exports:', [s for s in dir(helper) if not s.startswith('_')]); print('tools exports:', [s for s in dir(tools) if not s.startswith('_')])"
```

4) Example: create an `AgentCaller` to run the proposal agent (snippet from the notebook):

```python
from helper import make_agent_caller
from schema_proposal_structured import schema_proposal_agent  # notebook defines the agent

schema_proposal_caller = await make_agent_caller(schema_proposal_agent, {
		'feedback': '',
		'approved_user_goal': {'kind_of_graph': 'supply chain analysis', 'description': '...'},
		'approved_files': ['assemblies.csv', 'parts.csv', 'products.csv']
})

await schema_proposal_caller.call('How can these files be imported to construct the knowledge graph?')
```

Note: The notebook demonstrates these steps with more context and prints the `proposed_construction_plan` stored in agent session state.

## Safety / destructive actions

Some helper tools perform destructive operations on the Neo4j instance (for example `clear_neo4j_data()` and `drop_neo4j_indexes()` in `tools.py`). Use them with caution and only against a test/dev database. The notebook contains notes and warnings before destructive operations.

## Troubleshooting

- If the notebook fails to import `google.adk` or `neo4j_for_adk`, ensure you installed workspace dependencies from the top-level `requirements.txt` and that your PYTHONPATH includes local workspace helper modules if necessary.
- LLM output is non-deterministic — re-running the proposal/critic loop may produce different but comparable construction plans.
- Verify `NEO4J_IMPORT_DIR` points to the directory containing CSVs referenced by the notebook tools.

## Contributing and next steps

- If you'd like, I can add:
	- short example scripts that exercise the main agent flow (minimal runners),
	- a `LICENSE` file,
	- a small test that validates critical `tools.py` behaviors (e.g., `search_file` and `sample_file`).

If you want one of those, tell me which and I will add it.

