# Defense-in-Depth Score: AgentGPT

**Repo:** https://github.com/reworkd/agentgpt · **Commit:** `18b073ab05b2902e1d052c3d2799786d8623b5e5` · **Reviewed:** 2026-10-04
**What it is:** Self-hostable web app (Next.js + FastAPI) that runs an autonomous goal-driven agent loop in the browser with search, image, code-writing and Sid tools.
**Category:** AI Assistants
**Scored configuration:** Documented setup (setup.sh CLI generating a development .env, then docker-compose), with only the default Search tool active.
**Agent surface (default):** code execution no · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 6.3 / 10.0 (Moderate)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L1 | L3 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C3 | Tool & action scoping | L3 | L3 | L3 | L3 | 0.75 | — | **0.75** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C9 | Audit & traceability | L1 | L2 | L2 | L2 | 0.42 | — | **0.42** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L2 | 0.42 | — | **0.42** | High |

Controls where a risk surface exists: 2.33 / 6.0 (39%); 4 criteria scored SA (surface absent).

AgentGPT is mostly safe because it can do so little: it runs no code, writes nothing, and loads no plugins, so a hijacked run can't damage outside systems. The main risk is prompt injection from search results: nothing separates untrusted snippets from instructions. Output handling, authentication and secret management in the default install are also not locked down. The repository is archived and no longer maintained.

## Critical gaps
- None: no criterion is capped and no blast radius is at the worst level.

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

Every backend agent request is tied to a logged-in user via a session token, and the optional Sid connector uses that user's own OAuth grant with minimal read-only scopes. The vendor keys the server uses (OpenAI, Serper, Replicate) are operator-wide account keys shared by every user, but the model can only use them through fixed, narrow API calls. Authentication in the default install and per-resource authorization are not locked down.

- **S L2:** Per-user session identity on every request and minimal read-only Sid OAuth scopes, but the server's vendor keys are static, account-wide operator keys shared across users. — [platform/reworkd_platform/web/api/dependencies.py:21-41](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/dependencies.py#L21-L41); [platform/reworkd_platform/services/oauth_installers.py:59](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/services/oauth_installers.py#L59); [platform/reworkd_platform/settings.py:54-65](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/settings.py#L54-L65) (verified)
  - *To reach the next level:* Vendor credentials are not narrowed per tool or per user; only the Sid grant is minimally scoped.
- **C L2:** All agent endpoints resolve the requesting user from the session token, and Sid looks up the requesting user's own installation; there are no extensions or sub-agents to bypass this. — [platform/reworkd_platform/web/api/dependencies.py:21-41](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/dependencies.py#L21-L41); [platform/reworkd_platform/web/api/agent/tools/sidsearch.py:103-110](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/sidsearch.py#L103-L110) (verified)
  - *To reach the next level:* Authorization is not uniformly evaluated against the requesting principal.
- **D L1:** Per-user identity in the default CLI-generated install is not locked down. (verified)
  - *To reach the next level:* Default install should require a real identity provider.
- **B L3:** A hijacked agent can only issue fixed search, image and (opt-in) Sid read queries; it cannot reach the operator keys or write to any external system. — [platform/reworkd_platform/web/api/agent/tools/tools.py:27-39](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/tools.py#L27-L39); [platform/reworkd_platform/web/api/agent/tools/search.py:33-36](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/search.py#L33-L36); [platform/reworkd_platform/web/api/agent/tools/sidsearch.py:27-31](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/sidsearch.py#L27-L31) (verified)
  - *To reach the next level:* Operator API keys are long-lived and account-wide.
- **Cap:** none

### C2 Approval gates — 1.00 (high)

AgentGPT has no consequential actions. Its tools search Google through Serper, generate an image, query the user's Sid index read-only, or ask the model to write or reason in text. None of them write files, send messages, push code, or change state in an outside system, so there is nothing for an approval gate to protect. The tool set is fixed in code. Image generation does spend the operator's money, but that is bounded by the loop limits (scored under C10).

- **Structural absence:** searched `rg -n "class \w+\(Tool\)"` in `platform/reworkd_platform/web/api/agent/tools` → 7 hits (The complete tool set: Code, Reason, Search, SID, Wikipedia (not registered), Conclude, Image. All are read or text-generation only; registry at tools.py:27-39.); searched `rg -n -i "delete|send_email|smtp|git push|write_file|open\(.*w"` in `platform/reworkd_platform/web/api/agent` → 0 hits (No write, delete or send path in the agent code.)

### C3 Tool & action scoping — 0.75 (high)

The tools are narrow by construction. Each one calls a single hard-coded endpoint (Serper, Replicate's pinned model or DALL-E, Sid), with fixed quantities (top 5 results, 10 Sid results, one 256x256 image), and the model only supplies a query string. There is no generic HTTP, shell, file or SQL tool. A central validator checks that the chosen action is a known tool name. Gaps: argument handling is not strict, and the per-task tool selection is not enforced on every path.

- **S L3:** Destinations are hard-coded in each tool and quantities are fixed in code; the only model-controlled value is a query/prompt string. — [platform/reworkd_platform/web/api/agent/tools/search.py:33-36](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/search.py#L33-L36); [platform/reworkd_platform/web/api/agent/tools/sidsearch.py:27-31](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/sidsearch.py#L27-L31); [platform/reworkd_platform/web/api/agent/tools/sidsearch.py:116](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/sidsearch.py#L116); [platform/reworkd_platform/web/api/agent/tools/image.py:37-42](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/image.py#L37-L42) (verified)
  - *To reach the next level:* Tool argument handling is not strict, so the narrowest-tool level is not fully met.
- **C L3:** Every tool goes through the same fixed registry and the Analysis validator, and no extension tools exist. — [platform/reworkd_platform/web/api/agent/tools/tools.py:27-39](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/tools.py#L27-L39); [platform/reworkd_platform/web/api/agent/analysis.py:18-25](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/analysis.py#L18-L25) (verified)
  - *To reach the next level:* The central validator checks only the tool name, not arguments, so new tools do not inherit argument policy automatically.
- **D L3:** Only Search is on by default; Image, Code and Sid start inactive and must be enabled by the user, and no tool writes or executes anything. — [next/src/hooks/useTools.ts:36-38](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/next/src/hooks/useTools.ts#L36-L38); [platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py:113-122](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py#L113-L122) (verified)
  - *To reach the next level:* Enforce the per-task tool selection on every path.
- **B L3:** A misused tool can only issue a bounded query to one fixed vendor or read up to 10 Sid results for the current user. — [platform/reworkd_platform/web/api/agent/tools/sidsearch.py:116](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/sidsearch.py#L116); [platform/reworkd_platform/web/api/agent/tools/image.py:37-42](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/image.py#L37-L42); [platform/reworkd_platform/web/api/agent/tools/code.py:19-25](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/code.py#L19-L25) (verified)
  - *To reach the next level:* Image generation is not reversible spend.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

No model output is ever executed. The 'code' tool only asks the model to write code as text, which is shown to the user. The only eval-like call in the backend is Python's ast.literal_eval, which parses task lists and cannot run code. The setup CLI does spawn docker-compose, but the model cannot reach it.

- **Structural absence:** searched `rg -n "subprocess|os\.system|os\.popen|eval\(|exec\(|child_process|new Function\("` in `platform/reworkd_platform next/src` → 1 hits (Only hit is ast.literal_eval in task_output_parser.py:48, which parses literals only. cli/src spawns docker-compose at setup time and is not model-reachable.)

### C5 Untrusted input blast radius — 0.05 (high)

Google search snippets go straight into the summarization prompt with no marking or separation, and each step's result is fed into the prompt that plans the next tasks. So a malicious web page can steer the rest of the run. Nothing in code limits what a hijacked run can do afterwards. Output rendering in the frontend is not locked down, which widens what a hijacked run can leak. No irreversible actions exist, which keeps this from being the worst case.

- **S L0:** No structural limit: untrusted snippets are passed into prompts and their influence on later steps is unrestricted. — [platform/reworkd_platform/web/api/agent/tools/search.py:95-104](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/search.py#L95-L104); [platform/reworkd_platform/web/api/agent/tools/utils.py:55-75](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/utils.py#L55-L75); [platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py:159-165](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py#L159-L165) (verified)
  - *To reach the next level:* No approval or capability restriction applies once untrusted content has been read.
- **C L0:** Search results and Sid results enter model context with the same standing as the user's goal. — [platform/reworkd_platform/web/api/agent/tools/utils.py:55-75](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/utils.py#L55-L75); [platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py:159-165](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py#L159-L165) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from principal input anywhere.
- **D L0:** There is no control to enable. — [platform/reworkd_platform/web/api/agent/tools/utils.py:55-75](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/utils.py#L55-L75) (verified)
  - *To reach the next level:* No untrusted-input control exists in any configuration.
- **B L1:** Output rendering is not locked down, which widens what a hijacked run can leak, but there are no irreversible actions to take. (verified)
  - *To reach the next level:* Leakage paths from rendered output are not closed.
- **Cap:** none

### C6 Memory, context & configuration integrity — 1.00 (high)

Nothing the model reads persists into later behaviour. The Pinecone memory module is present but never wired into the agent service. Saved runs are stored only for display and are not loaded back into prompts; the chat endpoint gets earlier results from the user's own browser. The backend's .env file is the service's own config, not a workspace the agent works on.

- **Structural absence:** searched `rg -n "AgentMemory|PineconeMemory|MemoryWithFallback|NullAgentMemory|pinecone"` in `platform/reworkd_platform/web/api/agent platform/reworkd_platform/web/application.py platform/reworkd_platform/web/lifetime.py` → 0 hits (The memory/vector-store classes exist in web/api/memory and services/pinecone but are never imported by the agent service or app startup.); searched `rg -n "findById"` in `next/src` → 2 hits (Saved agents are fetched only by the read-only agent view page (pages/agent/index.tsx) and the router definition; they are never sent to the backend as context.)

### C7 Third-party extensions — 1.00 (high)

AgentGPT loads no third-party code at runtime: no plugins, no MCP servers, no downloaded tools or model files. Image generation calls Replicate's hosted API with a model version pinned by hash, and nothing is executed locally.

- **Structural absence:** searched `rg -n -i "mcp|plugin|importlib|trust_remote_code|pickle|torch\.load|pip install|npx "` in `platform/reworkd_platform next/src` → 4 hits (Hits are a pytest-plugin comment (conftest.py), importlib.metadata for the app version (application.py:1), and remark/rehype markdown plugins in MarkdownRenderer.tsx; none load third-party extensions at runtime.)

### C8 Secrets & sensitive-data protection — 0.17 (high)

API keys come from environment variables and are never put into prompts. OAuth tokens for the Sid connector are encrypted at rest with Fernet, and errors from users' own API keys are kept out of the logs. But key management for that encryption is not locked down. Summarization text is logged at INFO level by default. The vendor keys are long-lived and cover the whole account.

- **S L0:** Key management for the OAuth-token encryption is not locked down; settings are plain str, not masked types. (verified)
  - *To reach the next level:* Harden key management and use masked secret types.
- **C L1:** Only one path is protected: errors raised for a user-supplied custom API key are not logged. — [platform/reworkd_platform/web/api/agent/helpers.py:33-39](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/helpers.py#L33-L39); [platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py:190](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py#L190) (verified)
  - *To reach the next level:* Logs of model content, error handlers and transcripts are not redacted.
- **D L1:** Content-free page-view analytics components are mounted by default and summarization content is logged at INFO by default without redaction. — [next/src/pages/_app.tsx:33-34](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/next/src/pages/_app.tsx#L33-L34); [platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py:190](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/agent_service/open_ai_agent_service.py#L190) (verified)
  - *To reach the next level:* Payload logging should be off by default and redaction always on.
- **B L1:** Leaked operator keys (OpenAI, Replicate, Serper) are long-lived account keys, but the model cannot reach them; leaked Sid refresh tokens grant read access to a user's connected data. — [platform/reworkd_platform/settings.py:54-65](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/settings.py#L54-L65); [next/src/utils/interfaces.ts:21](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/next/src/utils/interfaces.ts#L21) (verified)
  - *To reach the next level:* Keys are not short-lived or per-task; custom user keys transit the server on every request.
- **Cap:** none

### C9 Audit & traceability — 0.42 (high)

Before each loop step runs, the backend writes a row recording the step type, linked to a run that stores the user and the goal. Because the row is written first, a database failure stops the step. But the record leaves out which tool ran, its arguments, and its result. The full conversation is saved only when the browser chooses to save it, so a run cannot be rebuilt from server-side records.

- **S L1:** Server-side records hold only run (user_id, goal) and step type with timestamp; no tool name, arguments or results. — [platform/reworkd_platform/db/models/agent.py:7-24](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/db/models/agent.py#L7-L24) (verified)
  - *To reach the next level:* Record the tool, arguments and result status for every step.
- **C L2:** Every agent endpoint goes through a validator that writes a task row, so all steps are recorded at that coarse level. — [platform/reworkd_platform/web/api/agent/dependancies.py:48-50](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/dependancies.py#L48-L50) (verified)
  - *To reach the next level:* Approvals don't exist and tool-level detail is missing, so coverage of tool calls is incomplete.
- **D L2:** Records are on by default and written by the server to its database, which the model cannot touch. — [platform/reworkd_platform/web/api/agent/dependancies.py:48-50](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/dependancies.py#L48-L50) (verified)
  - *To reach the next level:* Logging can't be raised further while the record itself is this thin (C and D capped one level above S).
- **B L2:** The task row is written before the step executes, so a failed write blocks the step, but the content is too thin to replay a trajectory. — [platform/reworkd_platform/web/api/agent/dependancies.py:48-50](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/dependancies.py#L48-L50); [platform/reworkd_platform/db/models/agent.py:7-24](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/db/models/agent.py#L7-L24) (verified)
  - *To reach the next level:* Records do not allow a full trajectory replay.
- **Cap:** none

### C10 Limits & kill switch — 0.42 (high)

The server caps how many times each step type can run per run: 25 by default, but the setup CLI sets it to 100. Per-call output tokens are also capped per model. There is no wall-clock limit, no cost cap, and no per-user cap on how many runs can be started. Stopping is a flag in the browser that ends the loop and cancels the stream being read. Because the backend is stateless per request, nothing keeps running server-side after a stop.

- **S L2:** Iteration cap per run and per-call max_tokens bounds are enforced in server code. — [platform/reworkd_platform/db/crud/agent.py:42-51](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/db/crud/agent.py#L42-L51); [platform/reworkd_platform/schemas/agent.py:35-40](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/schemas/agent.py#L35-L40) (verified)
  - *To reach the next level:* No wall-clock or cost cap and no rate limits on tools.
- **C L1:** The count applies to every loop step of a run, but tool HTTP calls have no explicit timeouts. — [platform/reworkd_platform/db/crud/agent.py:42-51](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/db/crud/agent.py#L42-L51); [platform/reworkd_platform/web/api/agent/tools/search.py:33-36](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/tools/search.py#L33-L36) (verified)
  - *To reach the next level:* No explicit per-tool timeout.
- **D L2:** Defaults exist and the model cannot raise them, but the CLI-generated env raises the loop cap to 100 and a client can start unlimited new runs. — [platform/reworkd_platform/settings.py:102](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/settings.py#L102); [cli/src/envGenerator.js:29-32](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/cli/src/envGenerator.js#L29-L32); [platform/reworkd_platform/web/api/agent/dependancies.py:33-45](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/platform/reworkd_platform/web/api/agent/dependancies.py#L33-L45) (verified)
  - *To reach the next level:* No hard ceiling and no per-user budget across runs.
- **B L2:** Stopping ends the client loop and cancels the stream reader; ceilings are moderate per run but unbounded across runs. — [next/src/services/agent/autonomous-agent.ts:41-59](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/next/src/services/agent/autonomous-agent.ts#L41-L59); [next/src/services/stream-utils.ts:44-47](https://github.com/reworkd/agentgpt/blob/18b073ab05b2902e1d052c3d2799786d8623b5e5/next/src/services/stream-utils.ts#L44-L47) (verified)
  - *To reach the next level:* No spend ceiling enforced outside the agent and no per-user cap on runs.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Serper search snippets enter prompts (platform/reworkd_platform/web/api/agent/tools/search.py:95) · [B] sensitive data/systems: User goal and prior results; opt-in Sid private Notion/Drive/email (platform/reworkd_platform/web/api/agent/tools/sidsearch.py:116) · [C] state change / egress: Output rendered in the user's browser · Same default session? Yes

## Highest-impact improvements
1. Harden handling of model output rendered in the browser. — C5 B L1→L3, +0.100 before caps (Playbook 1)
2. Harden secret-key management in the default install. — C8 S L0→L2, +0.150 before caps (Playbook 4)
3. Default to a real auth provider in the setup-generated install. — C1 D L1→L2, +0.050 before caps (Playbook 4)
4. Record tool name, arguments and result status in the agent_task row written before each step. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
5. Add a per-user run/spend budget and a wall-clock limit per run on the server. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- The repository is archived on GitHub (read-only, no longer maintained); findings will not be fixed upstream.
- The browser-side loop (next/src) was reviewed only for the paths that matter here: loop control, tool selection, markdown rendering and auth. Prisma schema, the docs site and the blog pages were not examined in depth.
- Behaviour of third-party libraries (frontend markdown rendering defaults, aiohttp default timeouts, server-side cancellation on client disconnect) is inferred from library defaults, not verified in their source.
- No text aimed at AI reviewers was found in the repository.
