# Defense-in-Depth Score: AutoChain

**Repo:** https://github.com/forethought-technologies/autochain · **Commit:** `5a1203bb01208b3e186c927222ae31702d309270` · **Reviewed:** 2026-10-04
**What it is:** Lightweight Python framework for building LLM agents with tools, memory and workflow evaluation.
**Category:** Agent Frameworks
**Scored configuration:** Library defaults as in the README: Chain(agent=ConversationalAgent.from_llm_and_tools(llm=ChatOpenAI()), memory=BufferMemory()) with developer-registered tools.
**Agent surface (default):** code execution no · filesystem write opt-in · network egress opt-in · external credentials yes · persistent memory opt-in · untrusted input opt-in · third party extensions opt-in · sub agents no · external communication no

## Score: 2.5 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L2 | L0 | L2 | L1 | 0.30 | G1 | **0.30** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C7 | Third-party extensions | L1 | L0 | L1 | L0 | 0.12 | — | **0.12** | High |
| C8 | Secrets & sensitive-data protection | L1 | L0 | L1 | L1 | 0.17 | — | **0.17** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |
| C10 | Limits & kill switch | L2 | L1 | L2 | L1 | 0.38 | — | **0.38** | High |

Controls where a risk surface exists: 1.47 / 9.0 (16%); 1 criterion scored SA (surface absent).

AutoChain is a thin agent loop with almost no safety controls: whatever tool the model picks runs immediately, with no approval, no argument validation by default, and no handling of untrusted tool output. Its surface is small (no code execution, no bundled write tools), so real risk depends on the tools a developer registers. The shipped Redis memory backend is not integrity-protected, and the HuggingFace wrapper's own example enables remote code. The project has been inactive since November 2023.

## Critical gaps
- Untrusted tool output feeds straight back into a loop that runs any registered tool, including an egress-capable search tool, with no human step. (ASI01, LLM01; C5) — [autochain/chain/base_chain.py:137-143](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L137-L143); [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93)
- Third-party HuggingFace model code runs in-process with no pinning, and the shipped example enables trust_remote_code. (ASI04, LLM03; C7) — [autochain/models/huggingface_text_generation_model.py:32](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L32); [autochain/models/huggingface_text_generation_model.py:85](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L85)

## Criterion details

### C1 Identity & least privilege — 0.05 (medium)

AutoChain has no identity or authorization layer. The OpenAI model wrapper always reads OPENAI_API_KEY from the process environment and sets it globally on the openai module, ignoring the openai_api_key field, and every registered tool is a plain Python callable that runs in-process with whatever authority the developer's process holds. Nothing checks who is asking or narrows what a tool can do with the credentials around it.

- **S L0:** Credentials are ambient process environment variables, set globally on the openai module; no scoped identity primitive exists. — [autochain/models/chat_openai.py:171](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L171); [autochain/models/chat_openai.py:182](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L182); searched `rg -n -i 'permission|authoriz|least.privilege'` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No dedicated or scoped identity for the agent or per-tool credentials.
- **C L0:** Tools run as bare in-process callables with no authorization check on the tool path. — [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93); searched `rg -n -i 'permission|authoriz|least.privilege'` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No authorization layer on the tool executor.
- **D L0:** Default install runs with the full authority of the launching process; least privilege is left entirely to the developer. — [autochain/models/chat_openai.py:171](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L171); [autochain/tools/base.py:88](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/base.py#L88) (verified)
  - *To reach the next level:* No narrower default identity.
- **B L1:** The framework itself holds only long-lived model-provider and search API keys, but tools it runs inherit the whole process authority and nothing narrows it. — [autochain/models/chat_openai.py:171](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L171); [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93) (inferred)
  - *To reach the next level:* Framework does nothing to limit what a hijacked tool path can reach beyond the developer's own choices.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no human approval step anywhere in the agent loop. When the model picks a registered tool, the chain looks the name up and calls it immediately; the only pre-action checks are additional LLM calls (should-answer, clarifying-question and confidence scoring), which are not approvals. There is no checkpoint, undo, or dry-run primitive.

- **S L0:** No approval mechanism; the only pre-action checks are LLM prompts (clarify_args_for_agent_action, is_generation_confident). — [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93); searched `rg -n -i 'approv|confirm|human_in|input\('` in `autochain` → 6 hits (Hits are _parse_input/fix_action_input names and the interactive eval harness prompt (base_test.py:130); none gates a tool call.) (verified)
  - *To reach the next level:* No human approval primitive for tool calls.
- **C L0:** Every tool, including the most powerful a developer registers, is executed directly. — [autochain/chain/chain.py:83](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L83); [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93) (verified)
  - *To reach the next level:* No gate on any tool path.
- **D L0:** No approval exists to be on by default. — searched `rg -n -i 'approv|confirm|human_in|input\('` in `autochain` → 6 hits (Hits are _parse_input/fix_action_input names and the interactive eval harness prompt (base_test.py:130); none gates a tool call.) (verified)
  - *To reach the next level:* Approval is not available at all, let alone default-on.
- **B L0:** No checkpoint, undo, or dry-run primitive bounds a wrongly taken action; registered tools run to completion. — [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93); searched `rg -n -i 'undo|rollback|checkpoint|dry.run'` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No rollback or preview primitive for consequential actions.
- **Cap:** none

### C3 Tool & action scoping — 0.30 (high)

Tools can optionally declare a pydantic args_schema, which the shared run path uses to type-check inputs, but it defaults to None and none of the bundled tools set it, so model arguments normally pass straight through to the developer's function. Unknown tool names are rejected. The retry path after a tool error asks the LLM to fix the input but then re-runs the tool with the original input, outside the error handler.

- **S L2:** When set, args_schema gives typed pydantic validation of tool inputs in the shared run path; there are no allowlists, path, URL, or bound checks. — [autochain/tools/base.py:63-73](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/base.py#L63-L73); [autochain/tools/base.py:33](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/base.py#L33) (verified)
  - *To reach the next level:* No allowlist validation (path containment, host allowlists, numeric bounds).
- **C L0:** Validation applies only to tools that set args_schema, and no bundled tool sets it (the Google search root_validator only checks its API key config), so in the default configuration no tool validates inputs. — searched `rg -n 'args_schema'` in `autochain` → 2 hits (Only the field definition and its use in Tool._parse_input; no tool sets it.); [autochain/tools/google_search/tool.py:17-22](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/google_search/tool.py#L17-L22) (verified)
  - *To reach the next level:* No built-in tool validates its inputs.
- **D L2:** The agent gets exactly the tools the developer passes (empty by default) and unknown tool names are refused, but there is no read-only default tier or per-task allowlist. — [autochain/chain/chain.py:109](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L109); [autochain/chain/chain.py:83](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L83); [autochain/tools/base.py:33](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/base.py#L33) (verified)
  - *To reach the next level:* No read-only default tool set with write/exec requiring explicit enabling.
- **B L1:** Tools are arbitrary in-process Python callables; the bundled Google search sends any model-chosen query to an external API. — [autochain/tools/base.py:88](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/base.py#L88); [autochain/tools/google_search/tool.py:22](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/google_search/tool.py#L22) (verified)
  - *To reach the next level:* Framework does not scope or bound what a misused tool can reach.
- **Cap:** G1 — Argument validation (args_schema) is opt-in per tool and defaults to None.

### C4 Code-execution isolation — 1.00 (high)

AutoChain has no code-execution feature: no shell tool, no Python executor, no eval/exec or subprocess anywhere in the library. Developer tools are ordinary Python functions, and model output is only parsed as JSON or OpenAI function calls. Memory-store and remote model loading paths are scored under memory and extensions.

- **Structural absence:** searched `rg -n 'subprocess|os\.system|eval\(|exec\(|Popen|compile\('` in `autochain` → 2 hits (Both hits are re.compile regex calls, not code execution.)

### C5 Untrusted input blast radius — 0.00 (high)

Tool results, including Google search results and vector-store documents, are fed back to the model as function messages or scratchpad text with nothing limiting what a hijacked agent can then do. No detection, taint tracking, or approval-after-untrusted-input exists. A search tool doubles as an outbound channel, and any developer-registered tool runs unattended.

- **S L0:** Nothing structurally limits a hijacked agent; tool output is appended to context and the next tool call runs immediately. — [autochain/chain/base_chain.py:137-143](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L137-L143); searched `rg -n -i 'untrusted|inject|sanitiz'` in `autochain` → 2 hits (Both hits are docstrings about prompt-template substitution, not input handling.) (verified)
  - *To reach the next level:* No approval or capability restriction after untrusted content is read.
- **C L0:** Untrusted tool results are not distinguished beyond the OpenAI function role; the conversational agent concatenates them into the prompt scratchpad. — [autochain/agent/conversational_agent/conversational_agent.py:116-118](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/agent/conversational_agent/conversational_agent.py#L116-L118) (verified)
  - *To reach the next level:* No source is treated as untrusted.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'untrusted|inject|sanitiz'` in `autochain` → 2 hits (Both hits are docstrings about prompt-template substitution, not input handling.) (verified)
  - *To reach the next level:* No untrusted-input control at all.
- **B L0:** Worst case: injected search or document content can make the agent leak conversation or internal-search data through the search query and run any registered tool, unattended; the framework prevents neither. — [autochain/tools/google_search/tool.py:22](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/tools/google_search/tool.py#L22); [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93) (verified)
  - *To reach the next level:* No human step before egress or state-changing tools.
- **Cap:** C5-WORSTCASE — B is L0: untrusted input, private data and unattended egress/tool execution combine in the default loop.

### C6 Memory, context & configuration integrity — 0.25 (high)

Default buffer memory lives in process, but the shipped Redis memory persists the whole conversation, tool inputs and intermediate steps (including untrusted tool output) for an hour and re-injects them on the next run with no validation. Its read-back path is not integrity-protected. The long-term vector memory has no per-user namespace. No workspace files are auto-loaded.

- **S L1:** Conversation and intermediate steps are written to memory automatically and re-injected; the only provenance is the function-message role, and nothing validates writes. — [autochain/chain/base_chain.py:43-50](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L43-L50) (verified)
  - *To reach the next level:* No provenance-tagged presentation of memory as data, and no write validation.
- **C L1:** No memory store has write controls; Redis and vector stores are equally uncontrolled. — [autochain/memory/long_term_memory.py:42](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/memory/long_term_memory.py#L42) (verified)
  - *To reach the next level:* Main store controlled is still missing.
- **D L1:** Redis keys are namespaced by a required developer-supplied prefix, but the long-term vector memory uses one global index with no namespace. — [autochain/memory/redis_memory.py:52](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/memory/redis_memory.py#L52); [autochain/memory/long_term_memory.py:42](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/memory/long_term_memory.py#L42) (verified)
  - *To reach the next level:* Per-user/session namespaces are not enforced across all stores.
- **B L1:** Poisoned Redis state persists across runs sharing a prefix (1 hour TTL) and can steer tool use. — [autochain/memory/redis_memory.py:22](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/memory/redis_memory.py#L22); searched `rg -n 'dotenv|AGENTS.md|CLAUDE.md'` in `autochain` → 0 hits (verified)
  - *To reach the next level:* Persisted content can trigger unattended tool use.
- **Cap:** none

### C7 Third-party extensions — 0.12 (high)

The optional HuggingFace model wrapper loads any model name through transformers with no pinned revision or integrity check, and its own docstring and readme example pass trust_remote_code=True. Loaded models run in-process with full access to the agent's environment. The transformers default keeps remote code off unless the developer passes that flag.

- **S L1:** Model source is developer-chosen but unpinned (no revision), fetched at load time. — [autochain/models/huggingface_text_generation_model.py:85-91](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L85-L91); searched `rg -n -i 'sha256|hashlib|revision='` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No version pinning of model revisions.
- **C L0:** No extension type (HF models, wrapped LangChain chains) is verified. — searched `rg -n -i 'sha256|hashlib|revision='` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No verification for any extension type.
- **D L1:** Using the HF wrapper is explicit, but model_kwargs forward straight to transformers and the shipped example sets trust_remote_code=True (lowered one level per the examples trap). — [autochain/models/huggingface_text_generation_model.py:32](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L32); [autochain/models/huggingface_text_generation_model.py:91](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L91) (verified)
  - *To reach the next level:* Explicit install that shows what will run, and examples that keep remote code off.
- **B L0:** Loaded model code and wrapped chains run in-process with the agent's environment and credentials. — [autochain/models/huggingface_text_generation_model.py:85](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/huggingface_text_generation_model.py#L85) (verified)
  - *To reach the next level:* No process separation for third-party code.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.17 (high)

API keys come from environment variables and are never redacted; there is no masking helper anywhere. The chain prints every tool input and output to stdout unconditionally, and verbose mode logs full prompts and conversation history. There is no telemetry. Keys are long-lived provider keys held in the process environment, readable by every in-process tool.

- **S L1:** Secrets are read from env vars; no masking or redaction exists. — [autochain/models/chat_openai.py:171](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L171); searched `rg -n -i 'redact|mask|SecretStr'` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No type-level masking or log filters.
- **C L0:** No path (logs, stdout, model-bound messages) is protected. — searched `rg -n -i 'redact|mask|SecretStr'` in `autochain` → 0 hits; [autochain/chain/chain.py:105](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L105) (verified)
  - *To reach the next level:* No path redacts sensitive data.
- **D L1:** No telemetry, but tool I/O is printed unconditionally and full-payload INFO logging is one flag away, unredacted. — searched `rg -n -i 'telemetry|sentry|posthog|analytics'` in `autochain` → 0 hits; [autochain/utils.py:58](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/utils.py#L58); [autochain/chain/base_chain.py:91](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L91) (verified)
  - *To reach the next level:* Logging defaults that don't emit tool payloads.
- **B L1:** Long-lived OpenAI/Google keys sit in the process environment and global openai module state. — [autochain/models/chat_openai.py:171](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L171); [autochain/models/chat_openai.py:182](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L182) (verified)
  - *To reach the next level:* Scoped keys.
- **Cap:** none

### C9 Audit & traceability — 0.20 (high)

The only record of tool calls is an unstructured print line per action to stdout, plus optional INFO logging of prompts. Nothing is written to durable storage, there are no timestamps, actor fields or correlation IDs, and the record disappears with the process unless the developer captures stdout.

- **S L1:** Unstructured print of tool name, input and output for each action. — [autochain/chain/chain.py:105](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L105) (verified)
  - *To reach the next level:* No structured per-call record with timestamps.
- **C L1:** Covers the Chain tool path; the LangChain wrapper chain bypasses it entirely. — [autochain/chain/langchain_wrapper_chain.py:30](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/langchain_wrapper_chain.py#L30) (verified)
  - *To reach the next level:* Not all tool paths recorded.
- **D L1:** The print is on by default but goes only to stdout; no stored record exists. — [autochain/chain/chain.py:105](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L105); searched `rg -n -i 'audit|FileHandler|open\('` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No default record stored outside the agent's reach.
- **B L0:** Records are lost at process exit; nothing is flushed durably. — searched `rg -n -i 'audit|FileHandler|open\('` in `autochain` → 0 hits (verified)
  - *To reach the next level:* No durable per-action record.
- **Cap:** none

### C10 Limits & kill switch — 0.38 (high)

The chain stops after 15 iterations by default and supports an optional wall-clock limit, both checked cooperatively between steps. There is no token or cost cap, no tool timeout, and LLM calls default to no request timeout with six retries each; each step can make several LLM calls (planning, confidence, clarification). Stopping is a raised KeyboardInterrupt with in-flight work left to finish.

- **S L2:** Iteration cap plus an optional wall-clock limit, both enforced in code between steps. — [autochain/chain/base_chain.py:170-179](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L170-L179); [autochain/chain/base_chain.py:30-31](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L30-L31) (verified)
  - *To reach the next level:* No token/cost cap and no rate limits on side-effecting tools.
- **C L1:** Limits apply only to the top-level loop; tool calls and LLM retries are unbounded. — searched `rg -n -i 'timeout'` in `autochain` → 8 hits (All hits are the LLM request_timeout field (default None) and tenacity retry on openai Timeout; no tool or session timeout.); [autochain/chain/chain.py:93](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/chain.py#L93) (verified)
  - *To reach the next level:* No tool timeouts.
- **D L2:** Default 15-iteration cap; wall-clock limit off by default; configurable by the developer, not the model. — [autochain/chain/base_chain.py:30-31](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/chain/base_chain.py#L30-L31) (verified)
  - *To reach the next level:* Wall-clock limit is off by default and nothing stops the model-independent ceiling from being raised arbitrarily.
- **B L1:** With no default wall-clock or cost ceiling, each step may make several LLM calls with 6 retries and no request timeout, and a hung tool blocks indefinitely. — [autochain/models/base.py:76](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/base.py#L76); [autochain/models/chat_openai.py:151](https://github.com/forethought-technologies/autochain/blob/5a1203bb01208b3e186c927222ae31702d309270/autochain/models/chat_openai.py#L151) (verified)
  - *To reach the next level:* Moderate time ceilings by default.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Google search and vector-store results return to the model (autochain/tools/google_search/tool.py:22, autochain/chain/base_chain.py:137) · [B] sensitive data/systems: Conversation history and internal search documents (autochain/memory/long_term_memory.py:42) · [C] state change / egress: Model-chosen search queries leave the process; any registered tool runs unattended (autochain/chain/chain.py:93) · Same default session? Yes

## Highest-impact improvements
1. Add an approval hook to Chain.take_next_step that tools can require (e.g. requires_approval flag) and that shows the exact tool input before tool.run. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Harden RedisMemory's read-back path. — C6 B L1→L2, +0.050 before caps (Playbook 2)
3. Write a structured JSONL record (timestamp, tool, input, status) per tool call instead of print. — C9 S L1→L2, +0.075 before caps (Playbook 1 step 3)
4. Set a default max_execution_time and tool timeout. — C10 D L2→L3, +0.050 before caps (Playbook 3 step 3)
5. Pin HF model revisions and drop trust_remote_code=True from the docstring and readme examples. — C7 S L1→L2, +0.075 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Behaviour of third-party libraries (transformers trust_remote_code default, openai SDK env handling, pydantic v1 field copying) is inferred, not verified in their source.
- Workflow-evaluation harness and tests were read only for tool usage, not scored as a deployment mode.
- No text aimed at AI reviewers was found in the repository.
