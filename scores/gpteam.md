# Defense-in-Depth Score: GPTeam

**Repo:** https://github.com/101dotxyz/gpteam · **Commit:** `a216364ea2dbb208e6501468c5e5b791594cc3d5` · **Reviewed:** 2026-10-04
**What it is:** Multi-agent simulation where GPT-4 agents with memory and plans move between locations and talk to each other to pursue configured goals.
**Category:** AI Assistants
**Scored configuration:** `poetry run world` with the default sqlite database, GPT-4, config.json as shipped, and no optional keys (Discord, SerpAPI, Wolfram Alpha unset).
**Agent surface (default):** code execution no · filesystem write no · network egress opt-in · external credentials opt-in · persistent memory yes · untrusted input opt-in · third party extensions no · sub agents yes · external communication opt-in

## Score: 4.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L2 | L3 | 0.47 | — | **0.47** | High |
| C2 | Approval gates | L0 | L0 | L0 | L3 | 0.15 | — | **0.15** | High |
| C3 | Tool & action scoping | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L3 | 0.15 | — | **0.15** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L1 | 0.33 | — | **0.33** | High |
| C10 | Limits & kill switch | L1 | L0 | L0 | L0 | 0.07 | — | **0.07** | High |

Controls where a risk surface exists: 2.02 / 8.0 (25%); 2 criteria scored SA (surface absent).

GPTeam is safe mostly because it can do little: by default its agents only talk to each other and write to a local sqlite file, with no shell, file or network tools. It has essentially no controls of its own - no approval step, an authorization mechanism that is not enforced on every path, unvalidated persistent memory, and no step or spending limit, so a run calls GPT-4 forever until killed. Enabling the Discord integration lets any channel member inject text into agent memory while agents post back unattended.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.47 (high)

In the default setup the agents' tools touch only the local simulation database; the OpenAI key lives in the process environment but no tool can read it or spawn a subprocess. When Discord is enabled each agent posts with its own bot token, which is a per-agent identity, but nothing scopes those tokens beyond what the operator configured. The project has a per-tool authorization flag, but every tool sets it to false, and the authorized-tools list is not enforced on every path.

- **S L2:** Tools use no credentials by default; with Discord enabled each agent uses a dedicated per-agent bot token selected by its own agent id, but all keys sit in one process environment and are not narrowed per tool. — [src/tools/send_message.py:45-50](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/send_message.py#L45-L50); [src/utils/database/seed.py:50-52](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/seed.py#L50-L52); [src/main.py:24](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/main.py#L24) (verified)
  - *To reach the next level:* No per-tool or read/write credential separation and no deterministic authorization gate before credentials are attached.
- **C L1:** An authorized-tools filter exists, but it is not enforced on every execution path. — [src/agent/executor.py:360](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L360) (verified)
  - *To reach the next level:* Every tool call does not pass through an effective authorization layer in code.
- **D L2:** Default install has no external credentials wired to tools (Discord, SerpAPI and Wolfram enable only when their env keys are set); widening is an operator env change with no warning. — [src/utils/parameters.py:43-48](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/parameters.py#L43-L48); [src/tools/base.py:176-177](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L176-L177) (verified)
  - *To reach the next level:* Default is not explicitly read-only; write tools (save-document, speak) are always on and adding credentials needs no elevation step.
- **B L3:** If the agent loop is hijacked in the default config it can only insert rows (messages, documents) into the local sqlite database; with Discord enabled it can additionally post to configured channels as its bot. — [src/tools/document.py:18-30](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L18-L30); [src/tools/send_message.py:45-50](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/send_message.py#L45-L50) (verified)
  - *To reach the next level:* Credentials are long-lived and not revocable per run; with Discord on, writes reach an external system.
- **Cap:** none

### C2 Approval gates — 0.15 (high)

There is no human approval step for any tool. A requires_authorization flag exists but is false on every tool, so whatever the model picks runs immediately. In the default configuration the tools only write simulation state (messages and documents) into a local database, which limits the damage; once Discord is enabled, agents post to real Discord channels with no gate.

- **S L0:** No approval mechanism: the executor runs the model-selected tool directly. — [src/agent/executor.py:360](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L360); [src/tools/base.py:39](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L39) (verified)
  - *To reach the next level:* No per-call human approval of any kind.
- **C L0:** No tool, including the external Discord-posting speak tool when enabled, crosses a gate. — [src/agent/executor.py:322-327](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L322-L327); [src/tools/base.py:251-260](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L251-L260) (verified)
  - *To reach the next level:* Consequential tools are not routed through any gate.
- **D L0:** No approval is on by default; every tool is seeded with requires_authorization=False and agents with an empty authorized list. — [src/utils/database/seed.py:46](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/seed.py#L46); [src/tools/base.py:39](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L39) (verified)
  - *To reach the next level:* Approval is not on by default.
- **B L3:** Default tools only insert simulation rows into a local sqlite file that db-reset wipes; no external action exists unless the operator sets Discord or search keys. — [src/tools/document.py:18-30](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L18-L30); [src/utils/parameters.py:43-48](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/parameters.py#L43-L48); [src/tools/send_message.py:45-50](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/send_message.py#L45-L50) (verified)
  - *To reach the next level:* No checkpoint/rollback of individual writes and no preview or rate limit on Discord posts when that integration is enabled.
- **Cap:** none

### C3 Tool & action scoping — 0.50 (high)

The tool set is narrow by design: speak, wait, ask-a-human, a company directory and save/read document, plus web search and Wolfram Alpha only when their API keys are set. Inputs are typed with pydantic schemas, recipients are resolved against the known agent list, and database queries are parameterized, but there are no length or quantity bounds. Per-location and per-agent tool scoping is not enforced on every path.

- **S L2:** Typed pydantic schemas, recipient lookup against known agents, and parameterized SQL; no size or count bounds on messages or documents. — [src/agent/message.py:60-71](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/message.py#L60-L71); [src/utils/database/sqlite.py:61](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/sqlite.py#L61); [src/tools/base.py:251-260](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L251-L260) (verified)
  - *To reach the next level:* No numeric bounds (message length, document size, number of writes) enforced in code.
- **C L2:** All tool calls flow through CustomTool.run and the same parameterized DB layer; free-text tools (wait, human, directory) take unvalidated strings but are low-power. — [src/tools/base.py:83-89](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L83-L89); [src/agent/executor.py:322-327](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L322-L327) (verified)
  - *To reach the next level:* No central policy layer; tool scoping is not enforced on every path.
- **D L2:** No exec tool and network tools off unless keys are set, but the default set always includes write tools and per-location/per-agent selection is not enforced on every path. — [src/tools/base.py:176-177](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L176-L177); [src/utils/database/seed.py:34](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/seed.py#L34) (verified)
  - *To reach the next level:* Default tool set is not read-only.
- **B L2:** A misused tool can write arbitrary content to the shared local simulation database (documents are global across agents) but nothing outside it by default. — [src/tools/document.py:18-30](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L18-L30); [src/tools/document.py:44-48](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L44-L48) (verified)
  - *To reach the next level:* Writes are not quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

No model-reachable code-execution path exists at this commit. A langchain bash tool is defined in src/tools/built_in.py but nothing imports or calls it, and the only subprocess call is the operator's database-reset command. Model output is parsed as JSON and used as parameterized database values, never evaluated.

- **Structural absence:** searched `rg -n 'subprocess|os\.system|os\.popen|eval\(|exec\(|BashProcess|PythonREPL|compile\('` in `src` → 5 hits (built_in.py:1,6 define a langchain BashProcess inside get_built_in_tools, which has no callers (see next search); main.py:3 is an unused import; reset.py:3,17 is the operator's db-reset CLI running 'supabase db reset', not reachable from the agent loop.); searched `rg -n 'BashProcess|get_built_in_tools'` in `src` → 3 hits (All three hits are the import and the definition in src/tools/built_in.py; no module imports or calls get_built_in_tools, so the bash tool is dead code.)

### C5 Untrusted input blast radius — 0.15 (high)

Nothing structurally limits a hijacked agent: tool results and other agents' messages enter the prompt as plain text alongside instructions. In the default configuration the only content agents read comes from other agents in the same simulation and the operator's own stdin answers, there is no outbound channel beyond the model provider, and the only state change is local database rows. If the operator enables Discord, any human in a mapped channel can inject text that becomes agent memory, and agents reply to Discord unattended.

- **S L0:** No provenance tracking, quarantine or taint-based gating; peer messages and tool observations are concatenated into the prompt. — [src/agent/executor.py:57-59](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L57-L59); [src/agent/base.py:994-1002](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L994-L1002) (verified)
  - *To reach the next level:* No approval or capability restriction triggered by reading untrusted content.
- **C L0:** Untrusted sources (peer agents, Discord humans when enabled, search results when enabled) are not distinguished from instructions. — [src/utils/discord.py:152-164](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/discord.py#L152-L164); [src/agent/executor.py:57-59](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L57-L59) (verified)
  - *To reach the next level:* Sources are not distinguished at all.
- **D L0:** No control exists to be on by default. — [src/agent/executor.py:57-59](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L57-L59) (verified)
  - *To reach the next level:* No untrusted-input control exists.
- **B L3:** Default config: no egress tool and no sensitive systems; a hijack can only write local simulation rows. With Discord enabled a hijacked agent could post to channels, but that is opt-in. — [src/tools/base.py:176-177](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/base.py#L176-L177); [src/utils/parameters.py:43-48](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/parameters.py#L43-L48); [src/tools/document.py:18-30](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L18-L30) (verified)
  - *To reach the next level:* Sessions that read peer content can still change state (DB writes), and the opt-in Discord path adds an unattended outbound channel.
- **Cap:** none

### C6 Memory, context & configuration integrity — 0.10 (high)

Every observation and model-generated reflection is written to a persistent sqlite memory table with no validation and is loaded back on the next run, where it drives planning and tool use. Memories are filtered per agent in queries, but documents saved by any agent are readable by all agents, and there is no provenance, expiry or per-entry review. The whole store can be inspected through the agents/*.txt dumps and wiped with db-reset, but nothing finer-grained exists.

- **S L0:** The model writes reflections and observations straight to persistent memory, which is re-injected as context. — [src/agent/base.py:615-619](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L615-L619); [src/agent/base.py:328](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L328); [src/agent/base.py:172-174](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L172-L174) (verified)
  - *To reach the next level:* No validation, gating or provenance on memory writes.
- **C L0:** No memory store (memories, global documents, LLM response cache.json) is controlled. — [src/agent/base.py:328](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L328); [src/tools/document.py:18-30](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L18-L30); [src/utils/cache.py:13](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/cache.py#L13) (verified)
  - *To reach the next level:* No store has write controls.
- **D L1:** Memories are queried per agent_id, but the documents table is global across agents and isolation is not a security control in this single-operator app. — [src/agent/base.py:172-174](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L172-L174); [src/tools/document.py:44-48](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/tools/document.py#L44-L48) (verified)
  - *To reach the next level:* Not every store is namespaced; documents are read without an agent filter.
- **B L1:** Poisoned memory persists across runs until a full db-reset and feeds plans that select tools. — [src/agent/base.py:172-174](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L172-L174); [src/agent/base.py:889-891](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L889-L891) (verified)
  - *To reach the next level:* Persistence is not limited to text-only influence or gated actions, and there is no per-entry purge or rollback.
- **Cap:** none
- **Notes:** config.json and .env are loaded from the project checkout (operator scope); the app does not operate on a third-party workspace, so C6-REPOCONFIG was not applied.

### C7 Third-party extensions — 1.00 (high)

The agent loads no third-party code at runtime: there is no plugin system, no MCP client, no model download and no package installation. The only deserialization is the app's own local vector store file, which this process writes itself.

- **Structural absence:** searched `rg -n -i 'importlib|__import__|pip install|trust_remote_code|torch\.load|pickle|mcp|plugin|entry_points'` in `src` → 5 hits (All hits are the app's own local HyperDB vector store file vectors.pickle.gz (save/load in sqlite.py, delete in reset.py); it is written by this process, not a third-party extension. No plugin, MCP, hub-download or dynamic-import path exists.); searched `rg -n 'load_tools'` in `src` → 4 hits (langchain load_tools is used only in dead code (built_in.py, and load_built_in_tool in tools/base.py which has no callers); both load langchain's bundled tools, not third-party code.)

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from environment variables or a .env file and are never placed in prompts. Nothing is masked: Discord bot tokens are copied into a plaintext column of the local sqlite database, and the full agent transcript is written unredacted to a log file that an unauthenticated local websocket streams to any connecting page. There is no telemetry.

- **S L1:** Secrets are read from env vars; Discord tokens are stored in plaintext in database.db; no redaction helpers exist. — [src/main.py:24](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/main.py#L24); [src/utils/database/seed.py:50-52](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/seed.py#L50-L52); [src/utils/database/sqlite.py:234](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/sqlite.py#L234) (verified)
  - *To reach the next level:* No type-level masking or log filters; tokens persisted in plaintext DB.
- **C L1:** Keys are kept out of model-bound prompts by design, but logs, transcript dumps and the DB have no protection. — [src/agent/executor.py:364](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L364); [src/utils/database/sqlite.py:234](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/sqlite.py#L234) (verified)
  - *To reach the next level:* Logs and transcripts are not protected.
- **D L1:** No telemetry, but verbose agent logs are always on, unredacted, and served over an unauthenticated websocket with Quart debug enabled. — [src/web/__init__.py:31-55](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/web/__init__.py#L31-L55); [src/web/__init__.py:24](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/web/__init__.py#L24); [src/utils/logging.py:107](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/logging.py#L107) (verified)
  - *To reach the next level:* Logs are not minimized or access-restricted by default.
- **B L1:** Long-lived OpenAI and Discord bot keys; not reachable by the model (no exec or env-reading tool) but stored/kept for the process lifetime. — [src/main.py:24](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/main.py#L24); [src/utils/database/seed.py:50-52](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/database/seed.py#L50-L52) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none

### C9 Audit & traceability — 0.33 (high)

Each tool call and its result are written as plain text lines to src/web/logs/agent.txt, and plan scratchpads with tool name and input are saved to the database. The log has no timestamps or structure, consecutive wait steps overwrite each other in the scratchpad, and the log file is truncated every time the world starts, so earlier runs leave no record.

- **S L1:** Unstructured INFO log lines of thought, action and response; scratchpad stores tool and input but no per-step timestamps. — [src/agent/executor.py:364](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L364); [src/agent/executor.py:374-379](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L374-L379) (verified)
  - *To reach the next level:* No structured per-call record with timestamps and result status.
- **C L2:** All tool calls go through the single executor path that logs them; there are no extensions or approvals to cover. — [src/agent/executor.py:364](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L364); [src/agent/executor.py:338-340](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/executor.py#L338-L340) (verified)
  - *To reach the next level:* Approvals/denials and sub-agent attribution are not recorded (no approval path exists).
- **D L1:** Logging is on by default but the log is truncated on every launch. — [src/utils/logging.py:89](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/logging.py#L89) (verified)
  - *To reach the next level:* Log is not preserved outside the process's control; it is wiped at start.
- **B L1:** Best-effort file logging; records from previous runs are destroyed on restart. — [src/utils/logging.py:89](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/logging.py#L89); [src/utils/logging.py:107](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/utils/logging.py#L107) (verified)
  - *To reach the next level:* Records are not durable across restarts.
- **Cap:** none

### C10 Limits & kill switch — 0.07 (high)

The world runs agents in an infinite loop with no step, time or spending limit. Plan durations are only suggested to the model in the prompt, and the only hard bound is a per-request timeout on some model calls. A runaway simulation keeps calling GPT-4 until the operator kills the process.

- **S L1:** Only advisory limits (max_duration_hrs told to the model) and per-request LLM timeouts; no iteration, wall-clock or cost cap. — [src/agent/plans.py:148](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/plans.py#L148); [src/agent/base.py:717](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/agent/base.py#L717); [src/world/base.py:98-101](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/world/base.py#L98-L101) (verified)
  - *To reach the next level:* No enforced iteration cap plus wall-clock or cost cap.
- **C L0:** No enforced limit applies to the agent loop or tools. — [src/world/base.py:98-101](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/world/base.py#L98-L101) (verified)
  - *To reach the next level:* No limit covers the top-level loop.
- **D L0:** Unlimited by default. — [src/world/base.py:98-101](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/world/base.py#L98-L101); [src/world/base.py:111](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/world/base.py#L111) (verified)
  - *To reach the next level:* No default limits exist.
- **B L0:** No ceiling: the loop spends model tokens indefinitely until killed. — [src/world/base.py:98-101](https://github.com/101dotxyz/gpteam/blob/a216364ea2dbb208e6501468c5e5b791594cc3d5/src/world/base.py#L98-L101) (verified)
  - *To reach the next level:* No run-level time or cost ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Default: peer-agent messages only (src/agent/executor.py:57-59); opt-in Discord humans (src/utils/discord.py:152-164) · [B] sensitive data/systems: None reachable by tools by default; API keys stay in process env (src/main.py:24) · [C] state change / egress: Local DB writes (src/tools/document.py:18-30); opt-in Discord posting (src/tools/send_message.py:45-50) · Same default session? No

## Highest-impact improvements
1. Add an enforced per-run step cap, wall-clock limit and token/cost budget to World.run_agent_loop. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)
2. Stop truncating agent.txt at startup; write a per-run, timestamped JSONL log of every tool call. — C9 D L1→L2, +0.050 before caps (Playbook 1 step 3)
3. Harden enforcement of requires_authorization/authorized_tools on the execution path. — C1 C L1→L2, +0.075 before caps (Playbook 4)
4. Require per-call human approval for speak when it posts to Discord, showing the exact message and channel. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Keep Discord tokens out of the database (read from env at send time) and redact logs. — C8 S L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of third-party libraries (langchain 0.0.144 SerpAPIWrapper/WolframAlphaAPIWrapper, hikari, Quart's default bind host, HyperDB's pickle-based load) was inferred from their documented behaviour, not read in source.
- The Supabase backend (src/utils/database/supabase.py) and the Window AI model path were not scored in depth; they are opt-in.
- No text aimed at AI reviewers was found in the repository.
