# adk-agent-patterns

**Five production-style AI agents built with Google's Agent Development Kit (ADK) and the Model Context Protocol (MCP).**
Deep research, CLI orchestration, bug triage, code review and a guarded e-commerce assistant, in one notebook that runs
free on Google Colab.

[Open the notebook in Colab](https://colab.research.google.com/drive/1q0j3CU3qO-ZnLwGpDXS7Ab4jvWdEyDGK?usp=sharing)
 · [Watch the demo video](https://youtu.be/sEl_ia_Z_h4)

---

## Why this project

Most agent demos show a single prompt and a single tool. This repo shows how real agent systems are put together: planning loops, tools
that other agents can verify, tool servers behind a protocol, deterministic checks around an LLM, and guardrails that don't depend on the
model behaving. Each agent demonstrates a different pattern, and all of them run on the Gemini API free tier with no paid services.

## The five agents

| | Agent | What it does | Patterns demonstrated |
|---|---|---|---|
| **A** | **Lead-generation deep research** | Turns a target-customer description into a ranked, sourced lead list (CSV + report) | `SequentialAgent`, `LoopAgent`, critique-and-refine loop, state passing with `output_key`, loop exit via `escalate` |
| **B** | **Gemini CLI as a tool** | Plans a coding task, runs Gemini CLI as a tool, and independently verifies the result | Wrapping a CLI agent as a `FunctionTool`, plan → execute → verify |
| **C** | **MCP bug assistant** | Triages bugs from an internal tracker, Stack Overflow and GitHub issues | Custom MCP server, `MCPToolset` over stdio, MCP and function tools together |
| **D** | **Code review assistant** | Finds bugs and risks, runs generated tests and returns corrected code | Deterministic tools around an LLM, sandboxed execution, unit-tested tools |
| **E** | **E-commerce agent** | Product search, cart, confirmed checkout and order support | Multi-agent transfer, `tool_filter`, `before_tool_callback` guardrails, DB transactions |

## How it fits together

```
                          ADK Runner  +  session state
                                      |
   +-----------+--------------+-------+-------+--------------+-----------+
   |           |              |               |              |           |
   A           B              C               D              E
 Lead gen   Gemini CLI    Bug assistant   Code review    E-commerce
 plan>loop  plan>run>      one agent       4-stage        shop <> support
 >write     verify         + MCP           pipeline       + MCP
   |           |              |               |              |
 web search  gemini CLI   MCP server     AST, ruff,      MCP server
                          (SQLite +     sandboxed       (SQLite)
                           public APIs)  pytest
```

### Design principles
- **Deterministic tools for facts, LLMs for judgement.** Parsing, linting, database queries and test runs are plain code.
- **Least privilege.** Each agent sees only the tools it needs, and user identity is injected by the system, never chosen by the model.
- **Verify, don't trust.** Agent B re-checks the CLI's work and agent D runs generated tests instead of just reading code.
- **Fail safely.** Timeouts, input-size limits, path-traversal guards, structured errors and rate-limit retries throughout.

## Quick start (Google Colab, free)

1. Create a free API key at <https://aistudio.google.com/apikey>.
2. Open `adk_agent_patterns.ipynb` in Colab (link above, or **File → Upload notebook**).
3. In Colab, open **Secrets**, add `GOOGLE_API_KEY`, and enable *Notebook access*.
4. Run the cells top to bottom, **one section at a time** (not *Run all*): setup first, then sections A to E.

Each section shows the agent's code, then a run cell that prints every step of the agent:

| Symbol | Meaning |
|---|---|
| `USER` | the message sent to the agent |
| `calls` | the agent invoked a tool, with its arguments |
| `got` | the tool's result |
| `[agent_name]` | the agent's answer |

The code files are created inside the Colab VM by `%%writefile` cells, so the notebook is fully self-contained.
ADK prints `UserWarning: [EXPERIMENTAL] ...` lines, which are harmless library notices.

### Models and free-tier limits

| Setting | Default | Notes |
|---|---|---|
| `ADK_MODEL` | `gemini-3.5-flash-lite` | Used by every ADK agent; chosen for generous free daily limits |
| `GEMINI_CLI_MODEL` | `gemini-3.8-flash` | Used only by agent B; a stronger model drives the CLI's file and shell tools more reliably |

Google retires models regularly and free-tier limits change, so check <https://ai.google.dev/gemini-api/docs/rate-limits> and the usage
page in AI Studio. The runner waits and retries automatically on `429 RESOURCE_EXHAUSTED`. Daily quotas reset at midnight Pacific time.

## Agent details

### A. Lead-generation deep research
`research_planner → LoopAgent[lead_researcher ⇄ research_evaluator] → lead_report_writer`

The planner builds an ideal-customer profile, search queries and a scoring rubric. The researcher gathers leads with a free
DuckDuckGo search tool. The evaluator checks completeness (company count, sources, target roles, duplicates) and either sends
feedback back or ends the loop. The writer scores the leads and saves `outputs/leads.csv` and `outputs/lead_report.md`.
Only public company-level information is collected: job titles, never personal emails or phone numbers.

### B. Gemini CLI pipeline
`task_planner → cli_executor (gemini -p ...) → verifier`

Gemini CLI runs non-interactively in a `workspace/` sandbox folder. The verifier lists the real files, reads them and re-runs
pytest itself. During development the CLI once claimed to have created files and passed tests while the workspace was empty, so the
tool now reports `no_files_created` and the verifier fails anything it cannot confirm.

### C. MCP bug assistant
`software_bug_assistant` ⟷ MCP server

Tools exposed by the server: `search_bugs`, `get_bug`, `create_bug`, `update_bug_status`, `search_stackoverflow`,
`search_github_issues`. The agent checks the internal tracker first, then public sources, then the web for current docs. It writes to
the tracker only when asked. The server logs to stderr because stdout is the MCP transport.

### D. Code review assistant
`structure_analyzer → style_checker → test_runner → review_synthesizer`

The AST analyzer reports complexity and risky patterns (mutable defaults, bare `except`, `eval`, `shell=True`). Ruff adds bug and
security rules. LLM-written pytest tests run in a temp directory with a 30-second timeout and no API keys in the environment. The
synthesizer returns a verdict, a findings table and corrected code, saved to `outputs/code_review.md`. The tools have their own unit tests.

### E. E-commerce agent
`shopping_agent` → transfer → `order_support_agent`, both backed by a SQLite MCP server

The store supports search, cart, confirmed checkout, order status and cancellation. A `before_tool_callback` injects the user ID
from session state and blocks checkout until the user explicitly confirms. Checkout and cancellation are atomic database
transactions, so stock, orders and carts stay consistent. Each agent gets only its own subset of tools through `tool_filter`.

## Repository contents

```
adk_agent_patterns.ipynb   the notebook, with saved outputs
README.md
```

When the notebook runs it creates these files in the Colab VM:

```
common.py  search_tool.py  run.py
a_lead_gen_agent/          agent.py
b_gemini_cli_agent/        agent.py
c_bug_assistant/           agent.py, bug_mcp_server.py
d_code_review_assistant/   agent.py, tools.py, samples/, tests/
e_ecommerce_agent/         agent.py, mcp_server.py
```

## Limitations
- Lead research is summarised from web-search snippets by an LLM, so **verify company facts before using them** for outreach.
- DuckDuckGo search can rate-limit; the tool retries and returns a clear error when it does.
- Gemini CLI runs with `--yolo` (auto-approve) inside the workspace, so use it only on a disposable VM such as Colab.
- Generated tests execute inside a temp directory with a timeout. That is not a hardened sandbox; production use needs containers.
- LLM answers vary between runs, so outputs will differ from the saved ones.

## Troubleshooting

| Problem | Fix |
|---|---|
| `GOOGLE_API_KEY` is missing | Add the Colab secret, enable notebook access, re-run the key cell |
| `404 ... model is no longer available` | Set `ADK_MODEL` to a current model (list them with `client.models.list()`) |
| `429 RESOURCE_EXHAUSTED` | Wait a minute, run one section at a time, or retry after the midnight-Pacific quota reset |
| `No module named 'mcp.server.fastmcp'` | `pip install "mcp<2"` |
| `gemini CLI not found` | Re-run the agent B install cell (Node 20+ is required) |
| MCP `Connection closed` | Run the server file directly and read the traceback |
| An agent prints no answer | Look for a `NO text` warning; re-run the section or try another model |

## Roadmap
- Swap SQLite for Postgres or AlloyDB through MCP Toolbox
- Deploy the agents to Cloud Run with the ADK API server
- Add ADK evaluation sets and regression tests for agent behaviour
- Add a chat UI with `adk web`

## Acknowledgements
Built with [Google ADK](https://google.github.io/adk-docs/) and the [Model Context Protocol](https://modelcontextprotocol.io/).
Patterns adapted from Google's deep-research ADK guide, `derrickchwong/gemini-cli-on-adk`, `google/adk-samples` (software bug
assistant), `ayoisio/adk-code-review-assistant` and `mtoscano84/sports-agent-adk-mcp-alloydb`.

## Author
**Soham Raj Jain**: M.S. Software Engineering (Data Science), San Jose State University.
