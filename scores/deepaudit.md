# Defense-in-Depth Score: DeepAudit

**Repo:** https://github.com/lintsinghua/DeepAudit · **Commit:** `630229163f10c9be74317d1edd06d16cfe6c59f0` · **Reviewed:** 2026-10-03
**What it is:** Multi-agent code vulnerability auditing system with sandboxed PoC verification
**Category:** Cybersecurity
**Scored configuration:** Shipped docker-compose.yml deployment (backend with Docker socket, Postgres, Redis, frontend) with the default env: Agent audit tasks created through the API with verification_level sandbox.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents yes · external communication yes

## Score: 3.2 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L1 | L0 | L1 | 0.28 | — | **0.28** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-POWERBYPASS | **0.05** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | L2 | L3 | L2 | L2 | 0.57 | — | **0.57** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |
| C7 | Third-party extensions | L1 | L1 | L0 | L3 | 0.30 | — | **0.30** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C9 | Audit & traceability | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


DeepAudit runs an autonomous orchestrator with recon, analysis and verification sub-agents that feed untrusted repository content to an LLM and let it run code and HTTP requests in per-call Docker containers, with no human approval step anywhere. The container itself is reasonably hardened (non-root, read-only root, no-new-privileges, network off by default), but the HTTP tool and external scanners use an unrestricted bridge network, so a hijacked agent can send data to arbitrary hosts unattended. The shipped deployment also mounts the Docker socket into the backend, and its default credential handling and API access controls are not locked down.

## Critical gaps
- The most powerful action paths (sandbox command execution, code execution and outbound HTTP) run with no approval gate in the default configuration. (ASI02, ASI09; C2) — [backend/app/api/v1/endpoints/agent_tasks.py:994](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L994); [backend/app/api/v1/endpoints/agent_tasks.py:995](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L995)
- A hijacked agent can send the audited private code to arbitrary hosts and issue outbound requests unattended, because untrusted repository content is not separated from instructions and egress is unrestricted. (ASI01, LLM01; C5) — [backend/app/services/agent/tools/sandbox_tool.py:408](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L408); [backend/app/services/agent/tools/sandbox_tool.py:645](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L645); [backend/app/services/agent/agents/verification.py:759](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/verification.py#L759)

## Criterion details

### C1 Identity & least privilege — 0.28 (high)

Model-driven tools run inside throwaway containers that hold no credentials and run as a non-root user, and the API checks project ownership on every agent-task route. The backend itself is broadly privileged, though: it mounts the Docker socket, holds the database, LLM and Git credentials, and a few host-side tools run with that authority. Default account and key handling in the shipped deployment is not locked down, so least privilege depends entirely on operator hardening.

- **S L2:** Sandbox containers run as 1000:1000 with no credentials, and API routes check owner_id per request, but the backend process holds one static, broad identity (Docker socket, database, provider keys). — [backend/app/services/agent/tools/sandbox_tool.py:31](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L31); [docker-compose.yml:46](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/docker-compose.yml#L46); [backend/app/api/v1/endpoints/agent_tasks.py:1545](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1545) (verified)
  - *To reach the next level:* No per-tool credential scoping; the backend's own authority is not narrowed and read and write share it.
- **C L1:** Per-user ownership checks cover the HTTP routes, but host-side tools (extract_function) and in-process code run with the backend's authority rather than a scoped identity. — [backend/app/api/v1/endpoints/agent_tasks.py:1019](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1019); [backend/app/services/agent/tools/kunlun_tool.py:210](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/kunlun_tool.py#L210) (verified)
  - *To reach the next level:* Not every tool path goes through the scoped identity and authorization layer.
- **D L0:** Registration is open, and default account and signing-key handling is not locked down. (verified)
  - *To reach the next level:* Default install should require operator-set credentials and keys.
- **B L1:** If the authorization layer fails, the backend can reach the Docker daemon, the database, and stored provider and Git credentials, i.e. write access across several systems. Independent layer that still holds: sandbox containers carry no credentials. — [docker-compose.yml:46](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/docker-compose.yml#L46); [backend/app/api/v1/endpoints/config.py:22](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/config.py#L22) (verified)
  - *To reach the next level:* Credentials and the Docker socket are not isolated from the process hosting host-side tools.
- **Cap:** none
- **Notes:** users.py strips is_superuser from self-updates (backend/app/api/v1/endpoints/users.py:123), so no self-escalation tool was found.

### C2 Approval gates — 0.05 (high)

There is no approval step. The verification agent chooses its own sandbox commands, code, and HTTP requests and executes them immediately, and the same holds for scanner tools. Results are limited by the container, but outbound HTTP requests to arbitrary hosts and all sandbox actions proceed unattended.

- **S L0:** No approval mechanism exists; the model alone decides which tools to run and with what arguments. — searched `rg -n -S -e 'approv|requires_approval|human_in_the_loop|ask_user'` in `backend/app/services/agent backend/app/api` → 0 hits (zero matches for approval vocabulary in agent and API code); [backend/app/api/v1/endpoints/agent_tasks.py:994](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L994) (verified)
  - *To reach the next level:* Add per-call human approval showing the exact call.
- **C L0:** The most powerful tools (sandbox_exec, run_code, sandbox_http) run with no gate at all. — [backend/app/api/v1/endpoints/agent_tasks.py:995](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L995); [backend/app/api/v1/endpoints/agent_tasks.py:1018](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1018) (verified)
  - *To reach the next level:* Route every consequential tool through a gate.
- **D L0:** Approval is not implemented, so it cannot be on by default. — searched `rg -n -S -e 'approv|requires_approval|human_in_the_loop|ask_user'` in `backend/app/services/agent backend/app/api` → 0 hits (no approval code to configure) (verified)
  - *To reach the next level:* Implement a default-on gate.
- **B L1:** A wrongly allowed action can send HTTP requests with any method to any reachable host from the sandbox; these are largely irreversible and not rate-limited. File and code changes stay inside throwaway containers. — [backend/app/services/agent/tools/sandbox_tool.py:645](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L645); [backend/app/services/agent/tools/sandbox_tool.py:408](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L408) (verified)
  - *To reach the next level:* No previews, rate limits, or bounds on outbound actions.
- **Cap:** C2-POWERBYPASS — The most powerful action paths (sandbox_exec, run_code, sandbox_http) are not behind any gate in the default configuration.

### C3 Tool & action scoping — 0.25 (high)

Tools take typed arguments, but validation is thin. The command tool relies on a first-word prefix check that allows interpreters, shells and curl, the HTTP tool accepts any URL, and argument and path handling in several tools is not a strict boundary. Only the file read, list and search tools check path containment. Every tool is enabled for its agent by default.

- **S L1:** The command tool checks only that the first word starts with an allowed name (list includes bash, sh, curl), the HTTP tool takes any URL, and scanner and file-tool argument handling is not a strict boundary. — [backend/app/services/agent/tools/sandbox_tool.py:605](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L605); [backend/app/services/agent/tools/sandbox_tool.py:539](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L539) (verified)
  - *To reach the next level:* No allowlist validation: no URL/host allowlist or internal-address block, no resolved-path containment everywhere, and no argument-level checks.
- **C L1:** Only the file read/list/search tools (and the report tool's file check) validate paths; the sandbox, HTTP, scanner and extract_function tools do not validate arguments against allowlists. — [backend/app/services/agent/tools/sandbox_tool.py:605](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L605) (verified)
  - *To reach the next level:* Most built-in tools do not validate arguments.
- **D L1:** Each agent role gets a fixed tool set that includes execution and outbound HTTP; nothing lets an operator trim it per task. — [backend/app/api/v1/endpoints/agent_tasks.py:994](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L994); [backend/app/api/v1/endpoints/agent_tasks.py:995](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L995); [backend/app/api/v1/endpoints/agent_tasks.py:1019](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1019) (verified)
  - *To reach the next level:* No read-only default set and no per-task allowlist.
- **B L1:** A misused tool can run arbitrary commands in a container and make requests to any host from a bridge-network container; one host-side path is not confined either. — [backend/app/services/agent/tools/sandbox_tool.py:408](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L408); [backend/app/services/agent/tools/sandbox_tool.py:645](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L645) (verified)
  - *To reach the next level:* Reach is not scoped or quantity-bounded.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (high)

Model-written code and commands run in a fresh Docker container per call: non-root user, read-only root filesystem, no-new-privileges, a list of dropped capabilities, memory and CPU limits, and network off by default; if Docker is unavailable the tools fail rather than run on the host. The gaps are that the capability drop is a 12-item list rather than all capabilities, there is no explicit seccomp profile or process limit, two environment variables can switch protections off silently, and the HTTP tool and scanners attach a bridge network with unrestricted egress. The backend that launches the containers mounts the Docker socket.

- **S L2:** Per-call container with non-root user, read-only root, no-new-privileges and a partial capability drop, but no explicit seccomp profile and no process limit, and the default drop list is not ALL. — [backend/app/services/agent/tools/sandbox_tool.py:31](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L31); [backend/app/services/agent/tools/sandbox_tool.py:30](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L30); [backend/app/services/agent/tools/sandbox_tool.py:33](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L33); [backend/app/core/config.py:117](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/core/config.py#L117); searched `rg -n -S -e 'privileged|network_mode=.host|pid_mode|cap_add|unconfined'` in `backend/app/services/agent/tools/sandbox_tool.py` → 0 hits (no privileged, host-network, extra-capability or unconfined options are set) (verified)
  - *To reach the next level:* No explicit seccomp profile or pids limit; capability drop defaults to a 12-item list instead of ALL.
- **C L3:** All model-reachable execution tools and scanner wrappers go through SandboxManager; when Docker is unavailable they return an error instead of falling back to the host. The exported Kunlun-M host runner is never instantiated. — [backend/app/services/agent/tools/sandbox_tool.py:115-118](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L115-L118); searched `rg -n -S -e 'KunlunMTool\('` in `backend/app` → 1 hits (the only hit is the class definition; no registration in agent_tasks.py) (verified)
  - *To reach the next level:* Capped by S+1; host-side git clone/zip extraction is outside the sandbox and the unused Kunlun-M runner would execute on the host.
- **D L2:** On by default, and the model cannot request host execution, but operators can turn off the capability drop and no-new-privileges through environment settings without any warning. — [backend/app/core/config.py:118](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/core/config.py#L118); [backend/app/services/agent/tools/sandbox_tool.py:39](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L39) (verified)
  - *To reach the next level:* Disabling needs only an env var and is silent.
- **B L2:** Containers hold no secrets, are removed after each call and have memory/CPU limits; the command path has network off, but the HTTP tool and scanners use a bridge network with unrestricted egress, and the temp workspace is writable. — [backend/app/services/agent/tools/sandbox_tool.py:408](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L408); [backend/app/services/agent/tools/external_tools.py:214](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/external_tools.py#L214); [backend/app/services/agent/tools/sandbox_tool.py:153](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L153); [backend/app/services/agent/tools/sandbox_tool.py:210](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L210) (verified)
  - *To reach the next level:* Egress is not allowlisted on the HTTP and scanner paths and there is no process limit.
- **Cap:** none
- **Notes:** C4-HOSTROOT was not applied: the Docker socket is mounted into the backend container (docker-compose.yml:46), not into the sandbox containers. It still makes the backend host-equivalent, which is reflected in C1.

### C5 Untrusted input blast radius — 0.00 (high)

Repository files, scanner output and retrieved code are all appended to the conversation as ordinary Observation messages, with no provenance marking, no tainting and no change in what tools are allowed after untrusted content is read. A hostile repository being audited can therefore steer the agent, which can fetch arbitrary URLs and send arbitrary-method requests from the sandbox without any human involved.

- **S L0:** Nothing in code limits a hijacked agent; tool results enter the history as user messages and no capability is disabled after untrusted reads. — [backend/app/services/agent/agents/verification.py:759](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/verification.py#L759); searched `rg -n -S -e 'untrusted|taint|quarantin|spotlight|prompt.injection'` in `backend/app/services/agent/agents backend/app/services/agent/prompts backend/app/services/agent/tools` → 0 hits (no injection defences in agents, prompts or tools) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement or detection.
- **C L0:** Untrusted sources (repository files, scanner output, RAG results) have the same standing as other context. — [backend/app/services/agent/agents/verification.py:759](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/verification.py#L759); [backend/app/services/agent/tools/rag_tool.py:177](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/rag_tool.py#L177) (verified)
  - *To reach the next level:* Distinguish and constrain untrusted sources.
- **D L0:** No control exists to be enabled by default. — searched `rg -n -S -e 'untrusted|taint|quarantin|spotlight|prompt.injection'` in `backend/app/services/agent/agents backend/app/services/agent/prompts backend/app/services/agent/tools` → 0 hits (no control present) (verified)
  - *To reach the next level:* Implement a default-on control.
- **B L0:** Assuming the hijack succeeds, the agent holds the audited private code and can send it to any host with sandbox_http while also issuing requests that change remote state, all unattended. — [backend/app/api/v1/endpoints/agent_tasks.py:995](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L995); [backend/app/services/agent/tools/sandbox_tool.py:408](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L408); [backend/app/services/agent/tools/sandbox_tool.py:645](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L645) (verified)
  - *To reach the next level:* No human approval, egress allowlist or independent layer limits outbound reach.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.45 (high)

The only persistent store is a per-project vector index built from the audited repository and queried through RAG tools, with file path and line shown for each hit. The model has no memory-write tool and the backend does not auto-load instruction or settings files from the audited repository. Index contents are still unvalidated repository text that persists across runs and feeds tool-using agents.

- **S L2:** Retrieved chunks are shown with file:line provenance and fenced as code, the model cannot write memory, and no repository instruction or config files are loaded; the indexed text itself is not validated. — [backend/app/services/agent/tools/rag_tool.py:177](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/rag_tool.py#L177); searched `rg -n -S -e 'save_memory|remember|mem0|long_term'` in `backend/app/services/agent` → 0 hits (no model-writable memory tool); searched `rg -n -S -e 'AGENTS\.md|CLAUDE\.md|\.cursorrules|load_dotenv'` in `backend/app` → 0 hits (no auto-loaded instruction files) (verified)
  - *To reach the next level:* Index writes are automatic and unvalidated; no review or expiry.
- **C L2:** The single retrieval store is provenance-tagged and no auto-loaded files exist, but nothing validates or expires index content. — [backend/app/api/v1/endpoints/agent_tasks.py:816](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L816) (verified)
  - *To reach the next level:* Index and summaries are not gated or integrity protected.
- **D L2:** Collections are named per project and task creation is limited to the project owner. — [backend/app/api/v1/endpoints/agent_tasks.py:816](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L816); [backend/app/api/v1/endpoints/agent_tasks.py:1545](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1545) (verified)
  - *To reach the next level:* Isolation is by collection name only; no per-tenant storage separation or retention limit.
- **B L1:** A poisoned index persists across runs for the project and is retrieved into agents that can run unattended tools. — [backend/app/api/v1/endpoints/agent_tasks.py:816](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L816) (verified)
  - *To reach the next level:* Poisoned context can trigger ungated tool use and persists until reindexed.
- **Cap:** none

### C7 Third-party extensions — 0.30 (high)

DeepAudit loads no plugins or MCP servers and no model files that execute code. It does fetch scanner rule packs from a public registry at run time inside the scanner containers, unpinned, and the model can supply the rule source. Those containers hold no credentials and mount the project read-only, which limits what a malicious rule source could do, though egress is unrestricted.

- **S L1:** Semgrep rule packs are downloaded at run time from whatever the default or model-supplied config names; nothing is pinned or verified. — [backend/app/services/agent/tools/external_tools.py:168](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/external_tools.py#L168); [backend/app/services/agent/tools/external_tools.py:197](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/external_tools.py#L197); searched `rg -n -S -e 'mcp|trust_remote_code|pickle\.load|torch\.load|entry_points|load_plugin'` in `backend/app/services/agent/tools backend/app/services/rag backend/app/services/llm` → 1 hits (the single hit is prompt text in sandbox_vuln.py; no plugin, MCP or model-file loading) (verified)
  - *To reach the next level:* No pinning, integrity check or curated registry for rules.
- **C L1:** Only one extension type exists (scanner rule packs) and it is unverified. — [backend/app/services/agent/tools/external_tools.py:197](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/external_tools.py#L197) (verified)
  - *To reach the next level:* The one extension type is not verified.
- **D L0:** Rule packs are fetched automatically on every scan with no consent step. — [backend/app/services/agent/tools/external_tools.py:214](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/external_tools.py#L214) (verified)
  - *To reach the next level:* Add explicit opt-in showing the source.
- **B L3:** The scanner runs in a per-call container with a read-only project mount, non-root user and no credentials in its environment. — [backend/app/services/agent/tools/sandbox_tool.py:283](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L283); [backend/app/services/agent/tools/sandbox_tool.py:31](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/sandbox_tool.py#L31) (verified)
  - *To reach the next level:* Network egress from the scanner container is unrestricted.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.30 (high)

Per-user provider keys, Git tokens and SSH private keys are encrypted at rest, and sandbox containers receive no secrets. However key management and API exposure of server-level credentials are not locked down, and nothing redacts secrets in audited code before it is sent to the LLM provider. Debug output prints key suffixes.

- **S L1:** Secrets come from env settings and are encrypted in the database, but key management is not robust; masking is limited to a few log lines. — [backend/app/api/v1/endpoints/config.py:192](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/config.py#L192) (verified)
  - *To reach the next level:* No secret manager or strong key management; no redaction before model-bound messages.
- **C L1:** Only the database-at-rest path is protected; model-bound messages, logs and API responses are not. (verified)
  - *To reach the next level:* Other paths (model messages, logs, API responses) are not covered.
- **D L2:** Telemetry is local only (vector-store telemetry is turned off) and verbose prompt/response logging is off by default, but key suffixes are printed and the redaction that exists cannot be relied on. — [backend/app/services/rag/indexer.py:234](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/rag/indexer.py#L234); [backend/app/services/agent/config.py:249](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/config.py#L249) (verified)
  - *To reach the next level:* Debug prints of key material remain.
- **B L1:** Leaked material is long-lived provider API keys, Git tokens and SSH private keys for the operator's accounts. — [backend/app/api/v1/endpoints/ssh_keys.py:79](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/ssh_keys.py#L79); [backend/app/api/v1/endpoints/config.py:22](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/config.py#L22) (verified)
  - *To reach the next level:* Long-lived, moderately scoped keys with no rotation or short-lived credentials.
- **Cap:** none
- **Notes:** C8-MODELSECRETS was not applied: the application's own secrets are not placed in prompts; secrets present in audited source can reach the provider, a risk SECURITY.md states.

### C9 Audit & traceability — 0.55 (high)

Each tool call is recorded as an event with arguments, a truncated result, duration and timestamp in the application database, written by the backend rather than by anything the model controls. Events carry the task but no separate approver or agent identity fields beyond the phase, outputs are cut to short previews, and a failed write is only logged while the action continues.

- **S L2:** Tool calls are stored with name, input, truncated output, duration and timestamp, and tasks record created_by. — [backend/app/services/agent/event_manager.py:129](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/event_manager.py#L129); [backend/app/models/agent_task.py:237](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/models/agent_task.py#L237); [backend/app/services/agent/event_manager.py:147](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/event_manager.py#L147) (verified)
  - *To reach the next level:* No actor attribution (agent identity, approver, delegation chain) or tamper evidence; results are truncated.
- **C L2:** Every agent's tool calls pass through BaseAgent.execute_tool, which emits the event, but there are no approvals to record and no records of configuration changes or credential use. — [backend/app/services/agent/agents/base.py:1121](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/base.py#L1121) (verified)
  - *To reach the next level:* Approvals, config changes and credential use are not recorded.
- **D L3:** Events are written by the backend event manager to the database, outside the sandbox and not reachable through any model tool. — [backend/app/services/agent/event_manager.py:305](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/event_manager.py#L305); [backend/app/services/agent/event_manager.py:309](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/event_manager.py#L309) (verified)
  - *To reach the next level:* The log can be removed by the project owner and is not tamper evident.
- **B L2:** Each event is saved as it occurs and failures are logged as errors, but the agent keeps running if a write fails. — [backend/app/services/agent/event_manager.py:309](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/event_manager.py#L309) (verified)
  - *To reach the next level:* High-risk actions are not blocked when the record cannot be written.
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each agent loop has a fixed iteration cap, and tools and sub-agent runs have enforced timeouts. The task-level timeout is accepted and stored but never read by the runner, the token budget is defined but not used, and the model chooses the sandbox timeout. Sub-agents can be re-dispatched and get fresh counters each time. Cancel sets a flag and cancels the asyncio task.

- **S L2:** Iteration caps plus enforced per-tool and per-sub-agent timeouts; no token or cost cap and no enforced wall-clock limit for the whole task. — [backend/app/services/agent/agents/orchestrator.py:145](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/orchestrator.py#L145); [backend/app/services/agent/agents/base.py:1136](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/base.py#L1136); [backend/app/services/agent/agents/orchestrator.py:723](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/orchestrator.py#L723); searched `rg -n -S -e 'AGENT_TOKEN_BUDGET|token_budget'` in `backend/app` → 2 hits (a settings field and a database column; neither is read by the runner); searched `rg -n -S -e 'timeout_seconds'` in `backend/app/api/v1/endpoints/agent_tasks.py` → 2 hits (request field and task assignment only; never enforced) (verified)
  - *To reach the next level:* No enforced task wall-clock or token/cost cap; no rate limits on side-effecting tools.
- **C L2:** The top-level loop and tool timeouts are bounded; sub-agent runs have their own caps that restart on every dispatch. — [backend/app/api/v1/endpoints/agent_tasks.py:1562](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1562); [backend/app/services/agent/agents/orchestrator.py:723](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/agents/orchestrator.py#L723) (verified)
  - *To reach the next level:* Sub-agents do not count against one shared budget.
- **D L2:** Defaults are sensible and operator or user configurable up to 200 iterations; the model cannot raise iteration limits but picks its own sandbox timeout and can re-dispatch sub-agents. — [backend/app/api/v1/endpoints/agent_tasks.py:82](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L82); [backend/app/services/agent/tools/run_code.py:32](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/services/agent/tools/run_code.py#L32) (verified)
  - *To reach the next level:* No hard ceilings and re-delegation resets limits.
- **B L1:** No spend ceiling exists and repeated sub-agent dispatch multiplies the worst-case run time; stop is cooperative at the loop level. — [backend/app/api/v1/endpoints/agent_tasks.py:1745](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1745); [backend/app/api/v1/endpoints/agent_tasks.py:1761](https://github.com/lintsinghua/DeepAudit/blob/630229163f10c9be74317d1edd06d16cfe6c59f0/backend/app/api/v1/endpoints/agent_tasks.py#L1761) (verified)
  - *To reach the next level:* No cost ceiling or enforced total runtime.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Scanned repository code and tool output enter the agent as Observation messages (backend/app/services/agent/agents/verification.py:759) · [B] sensitive data/systems: The audited private source code plus stored LLM, Git and SSH credentials in the backend (backend/app/api/v1/endpoints/config.py:22) · [C] state change / egress: sandbox_http sends requests to any URL from a bridge-network container (backend/app/services/agent/tools/sandbox_tool.py:408) · Same default session? Yes

## Highest-impact improvements
1. Add a per-call human approval step (showing the exact request) before sandbox_http, run_code and other verification tools execute. — C2 S L0→L3, +0.225 before caps
2. Remove unrestricted egress: run sandbox_http through a host allowlist that blocks internal and metadata addresses and require approval for any non-allowlisted target once repository content has been read. — C5 B L0→L3, +0.150 before caps
3. Require operator-set admin credentials and keys at first start, and do not make the first registrant an admin without an operator setup step. — C1 D L0→L3, +0.150 before caps
4. Replace prefix-matched command checks and unvalidated URL/path arguments with resolved-path containment and host allowlists across all sandbox, HTTP, file and scanner tools. — C3 S L1→L3, +0.150 before caps
5. Enforce the stored task timeout and the token budget in the orchestrator loop and cap the per-call sandbox timeout the model can request. — C10 S L2→L3, +0.075 before caps

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was installed, built, executed or probed.
- version is null: HEAD is one commit after tag v3.7.0 (git describe v3.7.0-1-g6302291) while backend/pyproject.toml reports 3.0.4.
- The Docker image contents (docker/sandbox/Dockerfile), Docker daemon defaults such as the default seccomp profile, and the frontend were not examined in depth; runtime Docker behaviour is inferred from the container options set in code.
- The Kunlun-M wrapper tool exists in the tree but is never instantiated or registered with an agent, and its target directory is absent, so it was treated as inactive; if enabled it would run on the host with the full process environment.
- No reviewer-directed instructions were found in the README, SECURITY.md, docs or code comments that were read.
