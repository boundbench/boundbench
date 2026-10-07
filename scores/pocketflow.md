# Defense-in-Depth Score: PocketFlow

**Repo:** https://github.com/The-Pocket/PocketFlow · **Commit:** `f74d023f93607b8c3268133339a5e532a949898c` (v0.0.0-185-gf74d023) · **Reviewed:** 2026-10-04
**What it is:** A 100-line Python graph framework for building LLM workflows and agents from nodes and action-labelled transitions.
**Category:** Agent Frameworks
**Scored configuration:** The pocketflow core library (pocketflow/__init__.py) used with its public constructors' default arguments; cookbook examples are not the product but are footnoted.
**Agent surface (default):** code execution no · filesystem write no · network egress no · external credentials no · persistent memory no · untrusted input yes · third party extensions no · sub agents yes · external communication no

## Score: 3.4 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C10 | Limits & kill switch | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |

Controls where a risk surface exists: 0.40 / 7.0 (6%); 3 criteria scored SA (surface absent).

PocketFlow is a tiny orchestration core with almost no surface of its own: it runs no code, loads no extensions, and stores nothing, which is where most of its points come from. Everything safety-relevant is left to the developer: there is no approval step, no input validation, no record of what ran, and no step or time limit, so an agent loop can run forever. The dominant risk is the documented agent pattern itself, where model output picks the next action and tool results flow back unmarked, letting a prompt injection drive any action the developer wired in with the process's full credentials.

## Critical gaps
- No identity or authorization layer: every node runs with the host process's full ambient credentials, so a hijacked agent holds all of them. (ASI03, T3; C1) — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49)
- Node output loops back into prompts unmarked and model output selects the next action with no approval, so a prompt injection can leak data and take irreversible actions unattended (C5-WORSTCASE). (ASI01, LLM01, T6; C5) — [pocketflow/__init__.py:42-45](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L42-L45); [docs/design_pattern/agent.md:117-118](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/design_pattern/agent.md#L117-L118)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

PocketFlow has no identity or authorization layer. Nodes are plain Python classes that run inside the developer's process with whatever credentials that process holds, and nothing checks an action against a policy or against the user who asked for it. The official docs and examples read provider keys from the environment of the same process. If an agent built on it is hijacked, the attacker gets the full authority of the host process.

- **S L0:** No credential scoping: the library passes only the shared dict and params to nodes, which use the host process's ambient credentials. — searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.); [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49) (verified)
  - *To reach the next level:* No dedicated or scoped identity for node actions; L1 needs at least a dedicated identity.
- **C L0:** No authorization check on any path; the flow runs whichever wired node the previous node's action string selects. — [pocketflow/__init__.py:42-45](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L42-L45); searched `rg -n -S -i -e 'approv|confirm|interrupt|human|input\('` in `pocketflow` → 0 hits (No approval, interrupt, or human-in-the-loop primitive exists in the library.) (verified)
  - *To reach the next level:* No authorization layer on the main path; L1 needs the main action path checked.
- **D L0:** The default run carries the developer process's full authority; narrowing is left entirely to the developer. — searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.) (verified)
  - *To reach the next level:* No narrower default identity; L1 needs a narrower default even if easily widened.
- **B L0:** A hijacked agent can use every credential and system the host process can reach, and the framework prevents none of it. — searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.); [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49) (verified)
  - *To reach the next level:* Nothing bounds what the process credentials can reach; L1 needs reach limited below the full user account.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval primitive in the library: no interrupt, pause, or confirmation step before a node runs. Whatever action string the model's node returns selects the next node, and that node's code runs immediately. Human-in-the-loop exists only as cookbook examples the developer writes by hand. A wrongly chosen or injected action can therefore run any consequential operation the developer wired in, with no undo.

- **S L0:** No approval mechanism: Flow._orch runs each successor as soon as the previous node returns its action. — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49); searched `rg -n -S -i -e 'approv|confirm|interrupt|human|input\('` in `pocketflow` → 0 hits (No approval, interrupt, or human-in-the-loop primitive exists in the library.) (verified)
  - *To reach the next level:* No approval step at all; L1 needs at least a blanket human approval.
- **C L0:** No node path is gated, including nodes that run code, SQL, or network calls in official examples. — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49); [cookbook/pocketflow-text2sql/nodes.py:78](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/cookbook/pocketflow-text2sql/nodes.py#L78) (verified)
  - *To reach the next level:* Every path is ungated; L1 needs at least flagged tools gated.
- **D L0:** There is no approval to turn on. — searched `rg -n -S -i -e 'approv|confirm|interrupt|human|input\('` in `pocketflow` → 0 hits (No approval, interrupt, or human-in-the-loop primitive exists in the library.) (verified)
  - *To reach the next level:* Approval does not exist, let alone on by default.
- **B L0:** Nodes can take irreversible actions (official examples write files and commit model-written SQL) with no checkpoint or undo in the framework. — [cookbook/pocketflow-text2sql/nodes.py:84-86](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/cookbook/pocketflow-text2sql/nodes.py#L84-L86); [cookbook/pocketflow-coding-agent/nodes.py:184](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/cookbook/pocketflow-coding-agent/nodes.py#L184) (verified)
  - *To reach the next level:* No reversibility or checkpointing; L1 needs at least some actions reversible.
- **Cap:** none

### C3 Tool & action scoping — 0.10 (high)

The library has no tool abstraction and no argument validation: data passes from the shared store to node code unchecked. The only structural limit is routing: a model-returned action can select only a successor the developer wired, and an unknown action ends the flow with a warning. Official docs encourage programmable actions such as model-written SQL, and the examples follow that advice without bounds.

- **S L0:** Raw passthrough: prep() output goes straight to exec() with no schema or validation layer. — [pocketflow/__init__.py:13](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L13); searched `rg -n -S -i -e 'valid|schema|allow|deny|assert'` in `pocketflow` → 0 hits (No argument validation, schema, or allow/deny layer exists in the library.) (verified)
  - *To reach the next level:* No typed schema or validation; L1 needs at least denylist filtering.
- **C L0:** No node input is validated by the framework. — searched `rg -n -S -i -e 'valid|schema|allow|deny|assert'` in `pocketflow` → 0 hits (No argument validation, schema, or allow/deny layer exists in the library.) (verified)
  - *To reach the next level:* No node validates inputs by default; L1 needs a few validated.
- **D L1:** No nodes are enabled unless the developer wires them, and routing only reaches wired successors (unknown actions end the flow), but every wired node, including write and network ones, is reachable from any decision node. — [pocketflow/__init__.py:42-45](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L42-L45) (verified)
  - *To reach the next level:* Tool groups cannot be selected per task; L2 needs selectable groups of nodes.
- **B L1:** A misused node has whatever reach the developer gave it; routing is limited to wired successors but nothing bounds paths, hosts, or quantities. — [pocketflow/__init__.py:42-45](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L42-L45); [docs/design_pattern/agent.md:65](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/design_pattern/agent.md#L65) (verified)
  - *To reach the next level:* No workspace or quantity scoping; L2 needs scoping to a project or workspace.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

The core library contains no code-execution feature: no shell, interpreter, eval of model output, or process spawning. Nodes are ordinary Python methods the developer writes, so isolating any code they run is the developer's job. Note that an official cookbook example (code generator) runs model-written Python in-process with exec() and full builtins, so developers copying examples get no isolation.

- **Structural absence:** searched `rg -n -S -e 'subprocess|os\.system|os\.popen|\beval\(|compile\(|__import__|importlib'` in `pocketflow` → 0 hits (No process spawning or dynamic evaluation in the library; the only exec( matches are the Node.exec method name (checked separately).); searched `rg -n -S -e '\bexec\('` in `pocketflow` → 4 hits (All four hits are the user-overridable Node.exec method (definition, call sites, and the .pyi stub), not Python's exec builtin.)
- **Notes:** Cookbook (not the product): cookbook/pocketflow-code-generator/utils/code_executor.py:13 runs model-written code with exec() in-process; cookbook/pocketflow-self-healing-mermaid/nodes.py:61 runs a subprocess on the host.

### C5 Untrusted input blast radius — 0.00 (high)

The library does nothing to separate untrusted content from instructions. Whatever a node writes into the shared store, such as web search results, becomes context for the next model call with the same standing as the user's request, and the model's reply directly chooses the next action. The official agent pattern combines web search results, model-chosen outbound queries, and the process's credentials in one loop with no approval. A successful prompt injection can therefore leak data and take irreversible actions unattended.

- **S L0:** No structural limit: node results enter the shared dict and are read back into prompts without provenance or restriction. — [docs/design_pattern/agent.md:117-125](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/design_pattern/agent.md#L117-L125); [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49) (verified)
  - *To reach the next level:* No detection, delimiting, or capability limit; L1 needs at least detection or spotlighting.
- **C L0:** No source is distinguished; every node's output has the same standing in shared. — [docs/design_pattern/agent.md:117-125](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/design_pattern/agent.md#L117-L125) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished; L1 needs at least one source handled.
- **D L0:** No control exists to be on by default. — searched `rg -n -S -i -e 'approv|confirm|interrupt|human|input\('` in `pocketflow` → 0 hits (No approval, interrupt, or human-in-the-loop primitive exists in the library.) (verified)
  - *To reach the next level:* Nothing is on by default.
- **B L0:** With no approval, a hijacked agent in the documented pattern can exfiltrate through model-chosen search queries and take any irreversible action the developer wired, unattended. — [docs/design_pattern/agent.md:117-118](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/design_pattern/agent.md#L117-L118); [cookbook/pocketflow-text2sql/nodes.py:78](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/cookbook/pocketflow-text2sql/nodes.py#L78) (verified)
  - *To reach the next level:* Nothing forces approval for egress or irreversible actions; L1 needs at least one of the two blocked unattended.
- **Cap:** C5-WORSTCASE — B is L0: leak plus irreversible action unattended in the documented default pattern.

### C6 Memory, context & configuration integrity — 1.00 (high)

The core library has no memory, retrieval store, checkpointing, or auto-loaded configuration. The shared store is an in-memory dict the caller passes in and owns. The repository's .cursorrules files are documentation for the developer's IDE and are never read by the library. An official example (coding agent) does persist a model-written memory file and auto-load AGENTS.md from the working directory, so that pattern is the developer's risk if copied.

- **Structural absence:** searched `rg -n -S -i -e 'open\(|pickle|json|sqlite|dotenv|environ|getenv|memory|\.md'` in `pocketflow` → 0 hits (No file, database, or environment reads or writes anywhere in the library.)
- **Notes:** Cookbook (not the product): cookbook/pocketflow-coding-agent/nodes.py:12-29 reads and writes .memory.md and loads AGENTS.md from the workdir without validation.

### C7 Third-party extensions — 1.00 (high)

PocketFlow loads no third-party code at runtime: the library imports only Python standard modules and has no plugin loader, MCP client, tool hub, or model-file loading. Anything external comes from the developer's own node code. Cookbook examples that use MCP are separate applications, not part of the library.

- **Structural absence:** searched `rg -n -S -i -e 'mcp|plugin|entry_points|pip install|trust_remote_code|torch\.load|requests|urllib|http'` in `pocketflow` → 0 hits (No extension loading or network code in the library; it imports only asyncio, warnings, copy, and time (pocketflow/__init__.py:1).)

### C8 Secrets & sensitive-data protection — 0.30 (high)

The library itself never reads, stores, logs, or transmits credentials, and it ships no telemetry or logging, so it adds no leak paths of its own. It also offers no protection: nothing masks secrets, keeps them out of model-bound prompts, or scrubs them from what nodes see. The docs advise keeping keys in environment variables, where they are long-lived and reachable by every node in the process.

- **S L1:** Docs advise environment-variable keys; the library has no secret handling, masking, or redaction of its own. — [docs/utility_function/llm.md:27](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/docs/utility_function/llm.md#L27); searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.) (verified)
  - *To reach the next level:* No type-level masking or redaction; L2 needs masking and log filters on main paths.
- **C L1:** The library creates no logs, transcripts, or telemetry to leak through, but model-bound prompts and node outputs are entirely unprotected. — searched `rg -n -S -i -e 'logging|logger|print\(|trace|telemetry|sentry'` in `pocketflow` → 0 hits (No logging, tracing, or telemetry in the library.); searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.) (verified)
  - *To reach the next level:* Model-bound messages and node outputs have no protection; L2 needs logs and transcripts covered as protected paths.
- **D L2:** No telemetry and no logging by default; only warnings that carry action names, never payloads. — searched `rg -n -S -e 'warnings\.warn'` in `pocketflow` → 4 hits (The only output: warnings for overwritten successors, a lone Node with successors (sync and async), and an unmatched action ending a flow; none records what a node did.); searched `rg -n -S -i -e 'logging|logger|print\(|trace|telemetry|sentry'` in `pocketflow` → 0 hits (No logging, tracing, or telemetry in the library.) (verified)
  - *To reach the next level:* No redaction exists to be always on; L3 needs redaction on by default.
- **B L1:** Long-lived developer keys in the process environment are reachable by every node. — searched `rg -n -S -i -e 'api_key|secret|token|redact|mask|credential|environ|getenv'` in `pocketflow` → 0 hits (The core library never reads, scopes, or forwards credentials; nodes use whatever the host process holds.) (verified)
  - *To reach the next level:* Keys are not scoped by the framework; L2 needs scoped keys.
- **Cap:** none

### C9 Audit & traceability — 0.00 (high)

The library keeps no record of what a flow did. Nodes run, return actions, and mutate the shared dict with no structured log of which node ran, with what inputs, or what it returned. The only output is a handful of Python warnings about wiring problems. Tracing appears only as an opt-in cookbook example that relies on a third-party service, so after an incident there is nothing from the framework to reconstruct.

- **S L0:** Node executions are not recorded; only wiring warnings are emitted. — searched `rg -n -S -e 'warnings\.warn'` in `pocketflow` → 4 hits (The only output: warnings for overwritten successors, a lone Node with successors (sync and async), and an unmatched action ending a flow; none records what a node did.); searched `rg -n -S -i -e 'logging|logger|print\(|trace|telemetry|sentry'` in `pocketflow` → 0 hits (No logging, tracing, or telemetry in the library.) (verified)
  - *To reach the next level:* No record of node runs; L1 needs at least unstructured logs of actions.
- **C L0:** Nothing is recorded on any path. — searched `rg -n -S -i -e 'logging|logger|print\(|trace|telemetry|sentry'` in `pocketflow` → 0 hits (No logging, tracing, or telemetry in the library.); [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49) (verified)
  - *To reach the next level:* No path records actions; L1 needs the main path recorded.
- **D L0:** There is no record to be on by default. — searched `rg -n -S -i -e 'logging|logger|print\(|trace|telemetry|sentry'` in `pocketflow` → 0 hits (No logging, tracing, or telemetry in the library.) (verified)
  - *To reach the next level:* No default record; L1 needs recording on by default.
- **B L0:** Actions proceed with no record; a crash leaves nothing behind. — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49) (verified)
  - *To reach the next level:* No per-action records at all; L1 needs best-effort recording.
- **Cap:** none
- **Notes:** cookbook/pocketflow-tracing is an opt-in example using an external tracing service; it is not part of the library and was not credited.

### C10 Limits & kill switch — 0.00 (high)

There are no step, time, or cost limits. A flow runs `while` there is a next node, so a decision node that keeps routing back to itself loops forever, and parallel batch nodes start every item at once with no concurrency limit. Retries default to one attempt, which bounds nothing about damage. There is no stop or cancel primitive; halting means killing the process.

- **S L0:** No step, wall-clock, or cost limit in the orchestration loop and no halt primitive. — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49); searched `rg -n -S -i -e 'max_steps|max_iter|timeout|budget|limit|cancel'` in `pocketflow` → 0 hits (No step, time, cost, or concurrency limit and no cancellation in the library.) (verified)
  - *To reach the next level:* No iteration cap; L1 needs at least an enforced step cap.
- **C L0:** Limits apply to nothing, including nested flows and parallel batches. — [pocketflow/__init__.py:80](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L80); [pocketflow/__init__.py:84-85](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L84-L85) (verified)
  - *To reach the next level:* No limit on any loop; L1 needs the top-level loop limited.
- **D L0:** Unlimited by default; the only numeric default is max_retries=1. — [pocketflow/__init__.py:27](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L27) (verified)
  - *To reach the next level:* No default limits; L1 needs defaults to exist.
- **B L0:** A runaway flow can loop and spend indefinitely; nothing stops in-flight work short of killing the process. — [pocketflow/__init__.py:46-49](https://github.com/The-Pocket/PocketFlow/blob/f74d023f93607b8c3268133339a5e532a949898c/pocketflow/__init__.py#L46-L49); searched `rg -n -S -i -e 'max_steps|max_iter|timeout|budget|limit|cancel'` in `pocketflow` → 0 hits (No step, time, cost, or concurrency limit and no cancellation in the library.) (verified)
  - *To reach the next level:* No ceiling of any kind; L1 needs some ceiling.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web search results fed back into the decision prompt (docs/design_pattern/agent.md:117-125) · [B] sensitive data/systems: provider and other keys in the host process environment (docs/utility_function/llm.md:27); nothing in the framework scopes them · [C] state change / egress: model-chosen search queries (docs/design_pattern/agent.md:118) and model-written SQL committed in the text2sql example (cookbook/pocketflow-text2sql/nodes.py:78-86) · Same default session? Yes

## Highest-impact improvements
1. Add a default max_steps to Flow._orch/_orch_async that stops the loop when exceeded. — C10 S L0→L1, +0.075 before caps (Playbook 3 step 3)
2. Make the step cap on by default with a sensible value that nested flows share. — C10 D L0→L2, +0.100 before caps (Playbook 3 step 3)
3. Emit a structured record (node, action, timestamps, result status) for every node run through a hook on _run. — C9 S L0→L2, +0.150 before caps (Playbook 1 step 3)
4. Add an approval hook that shows the exact prep result before exec runs on nodes marked consequential. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Let nodes declare an input schema that the framework validates before exec. — C3 S L0→L2, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Version: the only tag is v0.0.0 (2025-03-24); the pinned commit is 185 commits past it (git describe --tags). setup.py declares 0.0.3, which was not used.
- Scope is the core library (pocketflow/); the 59 cookbook examples were sampled for unsafe patterns (exec of model code, unbounded file writes, model SQL commits, workdir memory and AGENTS.md loading) but not scored.
- Structural-absence credit on C4, C6, and C7 reflects a library that has no such features, not strong controls; the applicable-only score shows the controls.
- No text aimed at AI reviewers was found; .cursorrules and .cursor/rules are copies of the docs for developers' IDE and are not loaded by the library.
