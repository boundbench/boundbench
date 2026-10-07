# Defense-in-Depth Score: BabyAGI (functionz)

**Repo:** https://github.com/yoheinakajima/babyagi · **Commit:** `fa8930ebe72a82e5ad57b356e7cbec96290e5bb2` (0.1.2) · **Reviewed:** 2026-10-04
**What it is:** Experimental self-building agent framework that stores Python functions in a database and executes them, with a Flask dashboard, REST API and function-calling chat.
**Category:** Agent Frameworks
**Scored configuration:** README quick start: `import babyagi` (auto-loads default packs) and `create_app('/dashboard').run(host='0.0.0.0', port=8080)` with default local SQLite DB in the working directory.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions yes · sub agents no · external communication opt-in

## Score: 1.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | C1-SELFESC | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C4 | Code-execution isolation | L4 | L0 | L0 | L2 | 0.40 | G1 | **0.40** (alt) | Medium |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-PUBLICTRIGGER | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L0 | 0.00 | C6-REPOCONFIG | **0.00** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L0 | 0.00 | C7-RCELOAD | **0.00** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C9 | Audit & traceability | L2 | L2 | L1 | L2 | 0.45 | — | **0.45** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |


BabyAGI runs every stored function with Python exec() inside the server process, injects every stored API key into each one, and pip-installs whatever imports a function lists. Access control on the dashboard and REST API that execute functions is not locked down. There are no approval gates, sandbox, or limits. The author labels it experimental and not for production; treat it that way.

## Critical gaps
- Every stored secret key is injected into the scope of every executed function, so any hijacked function holds all credentials. (ASI03, T3, LLM06; C1) — [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162); [babyagi/api/__init__.py:55-71](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/api/__init__.py#L55-L71)
- Default pack exposes add_key_wrapper and get_all_secret_keys as callable tools, letting a caller rewrite or dump the credential store. (ASI03, T3; C1) — [babyagi/functionz/packs/default/default_functions.py:75-80](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L75-L80); [babyagi/functionz/packs/default/default_functions.py:116-118](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L116-L118)
- All function code, including model-written code from add_new_function, runs via in-process exec() with every secret in scope and any listed import pip-installed on the host. (ASI05, T11, LLM05; C4) — [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122); [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19)
- Function code, triggers and the encryption key are loaded from files in the current working directory, so a pre-seeded funztionz.db executes attacker code on import with no trust decision. (ASI06, T1, ASI04; C6) — [babyagi/functionz/db/local_db.py:13](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/local_db.py#L13); [babyagi/functionz/core/registration.py:213-216](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/registration.py#L213-L216)
- The executor pip-installs any import name listed for a function, including model-supplied ones, on the host with no pinning or consent. (ASI04, T17, LLM03; C7) — [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

BabyAGI keeps one global store of secret keys and injects every stored key into the scope of every function it executes, whatever that function declares it needs. Access control on the dashboard and REST API that run functions is not locked down. Registered tools can also add or overwrite stored keys and read them all back.

- **S L0:** All functions run as the host OS user with every stored secret injected; there is no per-function or per-request identity. — [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162) (verified)
  - *To reach the next level:* Scope credentials per function (only key_dependencies) instead of injecting the whole keystore.
- **C L0:** No authorization check exists on any execution path, and the executor injects all keys into every function. — [babyagi/api/__init__.py:55-71](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/api/__init__.py#L55-L71); searched `rg -n -i 'login_required|authenticat|before_request'` in `babyagi/api babyagi/__init__.py babyagi/dashboard/__init__.py` → 5 hits (All hits are before_request hooks that only set g.functionz/g.dashboard_route or a comment; access control on these routes is not locked down.) (verified)
  - *To reach the next level:* Add an authorization layer that every execution path (API, chat tool calls, triggers) passes through.
- **D L0:** The documented default configuration makes the full keystore available to every execution path, and access control on it is not locked down. — [babyagi/__init__.py:128-134](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/__init__.py#L128-L134) (verified)
  - *To reach the next level:* Ship a hardened default deployment with minimal key exposure.
- **B L0:** A hijack gets every stored API key plus arbitrary code execution as the host user, which reaches the user's whole account and machine. — [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162); [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
  - *To reach the next level:* Narrow what a hijacked function can reach (scoped, short-lived keys; no host-user exec).
- **Cap:** C1-SELFESC — The default pack registers add_key_wrapper and get_all_secret_keys as callable functions, so a tool call can add, replace, or read every credential the framework holds.

### C2 Approval gates — 0.05 (high)

There is no approval step anywhere. Functions run directly from the HTTP API and from model tool calls in the chat, including the default function that adds new code to the system. Function code is versioned so a code overwrite can be rolled back, but anything the code does (package installs, external API calls, file writes) happens immediately.

- **S L0:** No approval mechanism exists; tool calls from the model are executed directly. — [babyagi/functionz/packs/default/function_calling_chat.py:137](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L137); searched `rg -n -i 'approv|confirm'` in `babyagi/functionz/core babyagi/api babyagi/__init__.py babyagi/dashboard/__init__.py` → 0 hits (No approval or confirmation logic in the executor, API or app factory.) (verified)
  - *To reach the next level:* Add per-call human approval showing exact arguments.
- **C L0:** The most powerful paths (add_new_function, arbitrary function execution) are ungated. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57); [babyagi/api/__init__.py:55-71](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/api/__init__.py#L55-L71) (verified)
  - *To reach the next level:* Route every consequential call through a gate.
- **D L0:** Nothing to turn on; no approval config exists. — searched `rg -n -i 'approv|confirm'` in `babyagi/functionz/core babyagi/api babyagi/__init__.py babyagi/dashboard/__init__.py` → 0 hits (No approval or confirmation logic in the executor, API or app factory.) (verified)
  - *To reach the next level:* Ship approval on by default.
- **B L1:** Function code changes are versioned and can be re-activated, but side effects of executed code (pip installs, plugin API calls, file writes) are irreversible. — [babyagi/api/__init__.py:92-94](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/api/__init__.py#L92-L94); [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19) (verified)
  - *To reach the next level:* Add checkpoints/dry-runs for side effects beyond function-code versions.
- **Cap:** none

### C3 Tool & action scoping — 0.05 (high)

The default tool set includes a function that stores arbitrary model-supplied Python code and imports, and every function can be called with any arguments; the only check is that required parameter names are present. In the chat, the user picks which functions the model sees, which is the one real narrowing, but the HTTP API can call any function directly.

- **S L0:** Arguments are raw passthrough; add_new_function accepts arbitrary code and validation only checks that parameter names are present. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57); [babyagi/functionz/core/execution.py:170-174](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L170-L174) (verified)
  - *To reach the next level:* Validate arguments in code against allowlists and bounds.
- **C L0:** No tool validates argument content. — [babyagi/functionz/core/execution.py:126-133](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L126-L133) (verified)
  - *To reach the next level:* Add a shared validation layer in the executor.
- **D L1:** All default packs (including add_new_function and get_all_secret_keys) load on import, but the chat only offers the model the functions the user selects. — [babyagi/__init__.py:128-134](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/__init__.py#L128-L134); [babyagi/dashboard/templates/chat.html:444-446](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/dashboard/templates/chat.html#L444-L446) (verified)
  - *To reach the next level:* Ship a read-only default tool set with write/exec opt-in.
- **B L0:** A misused add_new_function or any exec'd function reaches the whole machine as the host user. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57); [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
  - *To reach the next level:* Scope tools to a workspace with bounded operations.
- **Cap:** none

### C4 Code-execution isolation — 0.40 (medium)

All function code, including code a model writes through add_new_function or a caller pushes through the API, runs with Python exec() inside the server process, and any import it lists is pip-installed on the host. There is no sandbox on this path and every stored secret is in the executing scope. An optional E2B plugin can run code in a remote sandbox, but it is not loaded by default and only covers code explicitly passed to it.

- **default configuration** (default; raw 0.00 → 0.00)
  - **S L0:** In-process exec() of database-stored code with no isolation. — [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122); [babyagi/functionz/core/execution.py:44](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L44) (verified)
    - *To reach the next level:* Run function code in an OS or kernel-level sandbox.
  - **C L0:** Every execution path (API, chat tool calls, triggers, dependency loading, pip installs) runs on the host. — [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19); [babyagi/functionz/core/registration.py:213-216](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/registration.py#L213-L216) (verified)
    - *To reach the next level:* Route all execution paths through a sandbox.
  - **D L0:** No isolation is on by default. — [babyagi/__init__.py:128-134](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/__init__.py#L128-L134) (verified)
    - *To reach the next level:* Enable a sandbox by default.
  - **B L0:** Escape is moot: code already runs in the server process with all secrets, full network and the host user's filesystem. — [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162); [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
    - *To reach the next level:* Remove secrets and host access from the execution environment.
- **opt-in E2B remote sandbox plugin** (alt; raw 0.40, cap G1 → 0.40) ← counted
  - **S L4:** execute_code_in_sandbox runs code in an E2B remote CodeInterpreter sandbox. — [babyagi/functionz/packs/plugins/e2b.py:17-19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/plugins/e2b.py#L17-L19) (verified)
  - **C L0:** Only code explicitly passed to this plugin function is sandboxed; the core executor still exec()s all function code in-process. — [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
    - *To reach the next level:* Make the core executor use the sandbox for all function code.
  - **D L0:** The plugin pack is not in the auto-loaded default packs. — [babyagi/__init__.py:128-134](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/__init__.py#L128-L134) (verified)
    - *To reach the next level:* Load and use the sandbox by default.
  - **B L2:** The sandbox is remote and per call, but its network policy is set by the E2B service and not restricted in this code. — [babyagi/functionz/packs/plugins/e2b.py:17-19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/plugins/e2b.py#L17-L19) (inferred)
    - *To reach the next level:* Restrict sandbox egress in code.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C5 Untrusted input blast radius — 0.00 (high)

Nothing separates untrusted content from instructions. Tool results go back to the model without marking, and the handling of chat input and rendered model output is not locked down. A hijacked model can call add_new_function to run arbitrary code or read all secrets if those functions are offered.

- **S L0:** No structural limit on what a hijacked session can do. — [babyagi/functionz/packs/default/function_calling_chat.py:137](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L137) (verified)
  - *To reach the next level:* Disable egress and state-changing tools once untrusted content is read.
- **C L0:** Tool results are appended as plain tool messages with no provenance. — [babyagi/functionz/packs/default/function_calling_chat.py:146-151](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L146-L151); searched `rg -n -i 'login_required|authenticat|before_request'` in `babyagi/api babyagi/__init__.py babyagi/dashboard/__init__.py` → 5 hits (All hits are before_request hooks that only set g.functionz/g.dashboard_route or a comment; access control on these routes is not locked down.) (verified)
  - *To reach the next level:* Distinguish untrusted sources and authenticate principals.
- **D L0:** No control to configure. — searched `rg -n -i 'login_required|authenticat|before_request'` in `babyagi/api babyagi/__init__.py babyagi/dashboard/__init__.py` → 5 hits (All hits are before_request hooks that only set g.functionz/g.dashboard_route or a comment; access control on these routes is not locked down.) (verified)
  - *To reach the next level:* Ship a provenance-based limit on by default.
- **B L0:** A hijack can run arbitrary code or overwrite functions unattended, and an unattended exfiltration path exists. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57) (verified)
  - *To reach the next level:* Remove an unattended egress or irreversible-action leg.
- **Cap:** C5-PUBLICTRIGGER — Non-principals can drive an agent holding all stored keys in the documented default deployment.

### C6 Memory, context & configuration integrity — 0.00 (high)

The framework's state, including all function code, triggers and keys, lives in a SQLite file and an encryption-key file resolved relative to the current working directory, shared by every user of the server. The model can write new persistent code through add_new_function, and that code runs in later sessions. A funztionz.db already present in the working directory is loaded as-is, and its functions and triggers run when default packs register on import.

- **S L0:** The model can persist arbitrary code that is later exec()'d, and a working-directory database is trusted without any check. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57); [babyagi/functionz/db/local_db.py:13](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/local_db.py#L13) (verified)
  - *To reach the next level:* Gate persistent code writes and require trust decisions for loaded databases.
- **C L0:** Neither the function store, triggers, nor the embeddings CSV is controlled. — [babyagi/functionz/db/local_db.py:13](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/local_db.py#L13); [babyagi/functionz/packs/default/ai_functions.py:130](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/ai_functions.py#L130) (verified)
  - *To reach the next level:* Control every persistent store.
- **D L0:** A single global store is shared by all callers; no user or tenant namespace exists. — searched `rg -n 'user_id|tenant|namespace'` in `babyagi` → 0 hits (No user or tenant namespacing anywhere.); [babyagi/functionz/db/local_db.py:13](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/local_db.py#L13) (verified)
  - *To reach the next level:* Namespace persistent state per user.
- **B L0:** Poisoned functions or triggers persist across sessions and users and execute code automatically. — [babyagi/functionz/core/registration.py:213-216](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/registration.py#L213-L216); [babyagi/functionz/core/execution.py:217-240](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L217-L240) (verified)
  - *To reach the next level:* Make persistence reviewed or session-scoped.
- **Cap:** C6-REPOCONFIG — The function database and encryption_key.json are opened from the working directory, so a pre-seeded funztionz.db there supplies code and triggers that execute on import with no trust decision.

### C7 Third-party extensions — 0.00 (high)

Any import name stored with a function, including one supplied by the model through add_new_function, is pip-installed on the host at execution time if it isn't already importable, with no pinning, hash, or consent. Installed packages run in-process with every secret. The chat page also loads its Markdown renderer from a CDN without a pinned version.

- **S L0:** Model-chosen package names are pip-installed automatically, unpinned and unverified. — [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19); [babyagi/functionz/core/execution.py:31-35](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L31-L35) (verified)
  - *To reach the next level:* Pin and verify packages and never install on the model's say-so.
- **C L0:** Neither pip installs nor the CDN script are verified. — [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19); [babyagi/dashboard/templates/chat.html:56](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/dashboard/templates/chat.html#L56) (verified)
  - *To reach the next level:* Verify every extension type.
- **D L0:** Installs happen automatically on first execution with no prompt. — [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19) (verified)
  - *To reach the next level:* Require explicit user consent showing the exact package.
- **B L0:** Installed packages are imported into the server process with all secrets and network. — [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162); [babyagi/functionz/core/execution.py:19](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L19) (verified)
  - *To reach the next level:* Confine extensions in separate sandboxed processes.
- **Cap:** C7-RCELOAD — By default the executor pip-installs any import listed for a function, including model-supplied ones, with no consent.

### C8 Secrets & sensitive-data protection — 0.15 (high)

Stored keys are Fernet-encrypted in the database, but the encryption key sits in a plaintext JSON file next to it, and secret material is exposed through further output paths. The default pack's get_all_secret_keys function returns every key in plaintext to its caller. The chat function turns on LiteLLM's verbose logging by default.

- **S L1:** Values are Fernet-encrypted at rest, but the key is stored beside them in plaintext; there is no redaction. — [babyagi/functionz/db/models.py:15-28](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/models.py#L15-L28) (verified)
  - *To reach the next level:* Keep the encryption key in an OS keychain/secret manager and redact secrets from logs and outputs.
- **C L1:** Only at-rest storage is protected; logs, function outputs and the execution scope are not. — [babyagi/functionz/core/execution.py:137](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L137); searched `rg -n -i 'redact|mask|SecretStr'` in `babyagi` → 0 hits (No redaction or masking anywhere.) (verified)
  - *To reach the next level:* Protect logs and function outputs as well.
- **D L0:** Verbose LiteLLM payload logging is switched on by the default chat function. — [babyagi/functionz/packs/default/function_calling_chat.py:39](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L39); [babyagi/functionz/db/models.py:15-28](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/models.py#L15-L28) (verified)
  - *To reach the next level:* Turn verbose payload logging off by default.
- **B L0:** Long-lived third-party API keys are reachable by every function and through the default pack's key functions. — [babyagi/functionz/packs/default/default_functions.py:116-118](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L116-L118); [babyagi/functionz/core/execution.py:158-162](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L158-L162) (verified)
  - *To reach the next level:* Use scoped, short-lived keys not exposed to all code.
- **Cap:** none
- **Notes:** At-rest encryption is on by default (not opt-in); D is L0 because verbose LiteLLM payload logging is on by default, so G1 does not apply.

### C9 Audit & traceability — 0.45 (high)

Every function execution goes through one executor that writes a structured log row (function name, parameters, output, timing, status, parent and trigger links) before the code runs. That is a usable trail of calls, but it has no record of who requested a call, lives in a database file in the working directory that executed code can modify, and does not record what happens inside a function.

- **S L2:** Structured per-call logs with params, output, timestamps and parent/trigger correlation. — [babyagi/functionz/core/execution.py:96-105](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L96-L105); [babyagi/functionz/db/models.py:86-87](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/models.py#L86-L87) (verified)
  - *To reach the next level:* Record the requesting principal and agent identity on each entry.
- **C L2:** All functions, including plugins, triggers and chat tool calls, run through executor.execute and are logged; there are no approvals to record. — [babyagi/functionz/core/execution.py:55-64](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L55-L64); [babyagi/functionz/packs/default/function_calling_chat.py:137](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L137) (verified)
  - *To reach the next level:* Log approvals/denials and configuration and key changes.
- **D L1:** On by default, but stored in funztionz.db in the working directory, which in-process exec'd code can edit. — [babyagi/functionz/db/local_db.py:13](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/db/local_db.py#L13); [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
  - *To reach the next level:* Write logs from a component the executed code can't reach.
- **B L2:** A 'started' row is committed before execution and updated after, so records are flushed per action. — [babyagi/functionz/core/execution.py:96-105](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L96-L105); [babyagi/functionz/core/execution.py:137](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L137) (verified)
  - *To reach the next level:* Make the trajectory replayable and fail closed on log failure by design.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The core has no step, time, or cost limits and no way to stop a running function. The default chat makes at most two model calls per request and trigger chains skip functions already run in the chain, which bounds those loops, but any function (including model-written ones) can run forever in the server process.

- **S L1:** Only structural bounds: chat does at most two completions and trigger chains skip repeats. — [babyagi/functionz/packs/default/function_calling_chat.py:154](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/function_calling_chat.py#L154); [babyagi/functionz/core/execution.py:222-224](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L222-L224) (verified)
  - *To reach the next level:* Add enforced wall-clock and cost caps.
- **C L1:** Bounds apply only to the chat loop and trigger chains; function execution itself is unbounded. — searched `rg -n -i 'timeout|max_iter|kill|cancel|signal'` in `babyagi/functionz/core babyagi/api` → 0 hits (No timeouts, iteration caps, cancellation or signal handling in the executor or API.) (verified)
  - *To reach the next level:* Apply timeouts to every function execution.
- **D L1:** The chat bound is fixed in code, but the model can add new functions with unbounded loops. — [babyagi/functionz/packs/default/default_functions.py:35-57](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/packs/default/default_functions.py#L35-L57) (verified)
  - *To reach the next level:* Prevent model-added code from escaping limits.
- **B L0:** A runaway function can loop or spend indefinitely with no halt. — searched `rg -n -i 'timeout|max_iter|kill|cancel|signal'` in `babyagi/functionz/core babyagi/api` → 0 hits (No timeouts, iteration caps, cancellation or signal handling in the executor or API.); [babyagi/functionz/core/execution.py:122](https://github.com/yoheinakajima/babyagi/blob/fa8930ebe72a82e5ad57b356e7cbec96290e5bb2/babyagi/functionz/core/execution.py#L122) (verified)
  - *To reach the next level:* Add a ceiling and a kill path for running functions.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Chat/execute API input (babyagi/api/__init__.py:55-71); access to it is not locked down · [B] sensitive data/systems: All stored secret keys injected into every function (babyagi/functionz/core/execution.py:158-162) · [C] state change / egress: add_new_function persists code and exec() runs it (babyagi/functionz/packs/default/default_functions.py:35-57); a further egress path also exists · Same default session? Yes

## Highest-impact improvements
1. Harden access control on the API and dashboard in the default deployment. — C1 C L0→L2, +0.150 before caps (Playbook 4)
2. Inject only a function's declared key_dependencies and remove get_all_secret_keys/add_key_wrapper from the default pack. — C1 S L0→L2, +0.150 before caps (Playbook 4)
3. Stop auto-installing imports; require explicit, pinned installs by the operator. — C7 S L0→L2, +0.150 before caps (Playbook 3)
4. Require human approval before executing tool calls that add or update functions. — C2 S L0→L3, +0.225 before caps (Playbook 5)
5. Run function code in a subprocess sandbox with a timeout instead of in-process exec(). — C4 S L0→L2, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static review of the pinned commit only; nothing was installed or run.
- Draft packs (babyagi/functionz/packs/drafts) and third-party plugins were reviewed only lightly; they are not loaded by default.
- LiteLLM verbose-logging content and E2B sandbox network policy are inferred from library behaviour, not verified.
- No text aimed at AI reviewers was found; CODE_READINESS_ANALYSIS.md is an in-repo self-assessment and was treated as data, not evidence.
