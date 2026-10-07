# Defense-in-Depth Score: JARVIS (HuggingGPT)

**Repo:** https://github.com/microsoft/JARVIS (`hugginggpt`) · **Commit:** `7624cf388b47334ff8a0868e7d862dde18cfda86` · **Reviewed:** 2026-10-04
**What it is:** Microsoft research system in which an LLM plans tasks and dispatches them to Hugging Face models (HuggingGPT), plus EasyTool and TaskBench research code.
**Category:** AI Assistants
**Scored configuration:** HuggingGPT server mode as the README leads: awesome_chat.py --mode server with configs/config.default.yaml (hybrid inference, local models_server.py running), plus the bundled web client.
**Agent surface (default):** code execution no · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions yes · sub agents no · external communication no

## Score: 2.9 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L1 | 0.05 | C1-PASSTHRU | **0.05** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C4 | Code-execution isolation | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-PUBLICTRIGGER | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L0 | L0 | L1 | L0 | 0.05 | — | **0.05** | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C9 | Audit & traceability | L1 | L1 | L2 | L1 | 0.30 | — | **0.30** | High |
| C10 | Limits & kill switch | L1 | L1 | L1 | L0 | 0.20 | — | **0.20** | High |

Controls where a risk surface exists: 0.90 / 8.0 (11%); 2 criteria scored SA (surface absent).

HuggingGPT as shipped runs every request on the operator's OpenAI and Hugging Face credentials, and its default network exposure, credential handling, file handling and web client are not locked down. The model's plan runs without approval and fetches any URL it chooses. Treat it as research demo code: run it only on localhost, with throwaway keys.

## Critical gaps
- The server forwards client-supplied API keys to the downstream LLM API (token passthrough). (ASI03; C1) — [hugginggpt/server/awesome_chat.py:1054-1056](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L1054-L1056)
- A hijacked run can exfiltrate unattended via server-side URL fetches and has further unattended egress paths. (ASI01, LLM01; C5) — [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515)
- Unpinned third-party model files are unpickled with torch.load inside the model server process that holds the config with API keys. (ASI04, LLM03; C7) — [hugginggpt/server/models_server.py:279](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L279); [hugginggpt/server/models_server.py:57](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L57)

## Criterion details

### C1 Identity & least privilege — 0.05 (high)

Every request to the HuggingGPT server runs on the operator's own OpenAI key and Hugging Face token, read from the config file or environment at startup. The HTTP API has no authentication, and its default network exposure is not locked down. A request may also supply its own API key, which the server forwards to the model call. There is no per-request authorization anywhere.

- **S L0:** The server uses the operator's long-lived OpenAI key and Hugging Face token for every request, with no narrowing. — [hugginggpt/server/awesome_chat.py:105-108](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L105-L108); [hugginggpt/server/awesome_chat.py:157-160](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L157-L160) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity; the operator's personal keys are used directly.
- **C L0:** No route checks who the caller is; /hugginggpt, /tasks and /results all accept any JSON body and default to the server's key. — [hugginggpt/server/awesome_chat.py:1054-1056](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L1054-L1056); searched `rg -n -S 'login_required|auth' -g *.py` in `hugginggpt/server` → 3 hits (all 3 hits are Authorization headers for outbound OpenAI/HF calls, not inbound authentication) (verified)
  - *To reach the next level:* No authorization layer on any HTTP route or tool path.
- **D L0:** The default network exposure and access configuration are not locked down. (verified)
  - *To reach the next level:* No narrower default; lock down the default network exposure and access configuration.
- **B L1:** A hijacked or abused server can spend unbounded OpenAI credit and use the operator's Hugging Face token, whose scope the code never restricts (a write token works). — [hugginggpt/server/awesome_chat.py:157-160](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L157-L160); [hugginggpt/server/awesome_chat.py:206](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L206) (verified)
  - *To reach the next level:* Credentials span two external accounts with no scoping to a single project or read-only use.
- **Cap:** C1-PASSTHRU — The server accepts a client-supplied api_key in the request body and forwards it to the downstream LLM API (awesome_chat.py:1054 and send_request at :206).

### C2 Approval gates — 0.05 (high)

HuggingGPT has no approval step at all. The model's task plan is executed immediately: it fetches whatever image or audio URLs the plan names, calls remote or local models, writes generated files into the publicly served folder, and spends the operator's API credit, with no human seeing the plan first. The consequential actions are mostly low-impact (new media files, inference calls), but outbound requests and spend cannot be undone.

- **S L0:** Parsed tasks are dispatched straight to worker threads with no human or rule-based approval. — [hugginggpt/server/awesome_chat.py:947-950](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L947-L950); searched `rg -n -S 'approv|confirm' -g *.py` in `hugginggpt/server` → 0 hits (verified)
  - *To reach the next level:* No per-call approval of any kind.
- **C L0:** No action path crosses a gate, including arbitrary URL fetches and model calls. — [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515); [hugginggpt/server/awesome_chat.py:947-950](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L947-L950) (verified)
  - *To reach the next level:* No gate exists for any tool path.
- **D L0:** No approval mechanism exists in any configuration. — searched `rg -n -S 'approv|confirm' -g *.py` in `hugginggpt/server` → 0 hits (verified)
  - *To reach the next level:* Approval is not available even opt-in.
- **B L1:** Unattended actions are outbound HTTP GETs to model-chosen URLs, paid inference calls, and new files in the public folder; spend and outbound requests are irreversible, file writes are low-impact. — [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515); [hugginggpt/server/awesome_chat.py:441-442](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L441-L442) (verified)
  - *To reach the next level:* No rate or spend bound on consequential actions and no preview of fetches.
- **Cap:** none

### C3 Tool & action scoping — 0.25 (high)

The tool set is a fixed list of AI task types, and the code rejects task names outside its model catalogue, which is the main real restriction. Arguments are not validated: image and audio inputs are any URL the model writes, fetched server-side with no host allowlist (internal addresses and cloud metadata included), and local-path handling is not a strict boundary. The model's chosen model id is also used without checking it against the candidate list.

- **S L1:** Only the task type is allowlisted; URL arguments pass through raw, and local-path handling is not a strict boundary. — [hugginggpt/server/awesome_chat.py:827-832](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L827-L832); [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515) (verified)
  - *To reach the next level:* No URL/host allowlist for resource arguments.
- **C L1:** The task-type check applies to dispatch, but no tool validates its arguments, and the LLM-selected model id is not checked against candidates. — [hugginggpt/server/awesome_chat.py:871-874](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L871-L874); [hugginggpt/server/awesome_chat.py:254-257](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L254-L257) (verified)
  - *To reach the next level:* Argument validation on all built-in task handlers.
- **D L1:** All task types, including server-side URL fetching, are enabled by default; only inference_mode/local_deployment change which run locally. — [hugginggpt/server/configs/config.default.yaml:14-15](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L14-L15) (verified)
  - *To reach the next level:* No read-only or minimal default tool set; fetch capability cannot be turned off.
- **B L1:** A misused tool performs blind SSRF from the server to any host; local-file reach is not confined. — [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515) (verified)
  - *To reach the next level:* Fetches are not scoped to a host allowlist.
- **Cap:** none

### C4 Code-execution isolation — 1.00 (high)

HuggingGPT never interprets model-generated text as code. The model's output is parsed as JSON and only selects task types, model ids and media URLs; the single shell call in the model server runs ffmpeg on a temporary file path the code itself generates. This is a design absence, not a sandbox. Note that the separate EasyTool research scripts in the same repository do pass model output to Python eval(), which is outside the scored component.

- **Structural absence:** searched `rg -n -S '\beval\(|\bexec\(|subprocess|os\.system|os\.popen' -g *.py` in `hugginggpt/server` → 1 hits (the only hit is models_server.py:393 os.system(ffmpeg -i {video_path}) where video_path comes from diffusers export_to_video (a temp file), not from model or request text); [hugginggpt/server/models_server.py:391-393](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L391-L393)
- **Notes:** Out-of-scope easytool/ scripts call eval() on LLM output (easytool/easytool/funcQA.py:75, toolbench.py:74).

### C5 Untrusted input blast radius — 0.00 (high)

Nothing limits what a manipulated model can do. Model results from remote Hugging Face endpoints and content derived from user-supplied media are pasted into the final prompt with the same standing as the user's request. The bundled web client's handling of model output is not locked down, which gives a hijacked response further unattended egress paths. Because the server is shared, others can drive it with the operator's credentials.

- **S L0:** No provenance tracking, approval or capability restriction once untrusted content is read; the only defence is prompt wording. — [hugginggpt/server/awesome_chat.py:382-385](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L382-L385); [hugginggpt/server/configs/config.default.yaml:46](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L46) (verified)
  - *To reach the next level:* No approval or capability restriction after untrusted content enters the session.
- **C L0:** Model inference results, remote model outputs and request messages enter prompts undistinguished from the principal's input. — [hugginggpt/server/awesome_chat.py:382-385](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L382-L385); [hugginggpt/server/awesome_chat.py:1054-1056](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L1054-L1056) (verified)
  - *To reach the next level:* Untrusted sources are not distinguished from user instructions.
- **D L0:** No control exists to enable. — [hugginggpt/server/awesome_chat.py:382-385](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L382-L385) (verified)
  - *To reach the next level:* No control available in any configuration.
- **B L0:** A hijacked run can fetch arbitrary URLs server-side (exfiltration); on the shared multi-user server this reaches other users' media in the shared public folder. Local-file and web-client handling are not locked down either, leaving additional unattended egress paths. — [hugginggpt/server/awesome_chat.py:515](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L515); [hugginggpt/server/awesome_chat.py:1019](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L1019) (verified)
  - *To reach the next level:* Exfiltration channels (server-side fetch and other unattended egress paths) are not removed or gated.
- **Cap:** C5-PUBLICTRIGGER — The default server accepts instructions from parties other than the operator while holding the operator's OpenAI and Hugging Face credentials.

### C6 Memory, context & configuration integrity — 1.00 (high)

The HuggingGPT server keeps no memory between requests: the conversation history is sent by the client with each call, prompts and demos come only from the operator's --config file, and the logs it writes are never read back into the model. There is no .env loading or workspace instruction file. One caveat outside the scored server mode: the Gradio demo keeps a single global conversation list and API key shared by every visitor.

- **Structural absence:** searched `rg -n -S 'dotenv|chromadb|faiss|sqlite|save_memory' -g *.py` in `hugginggpt/server` → 0 hits; [hugginggpt/server/awesome_chat.py:38](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L38)
- **Notes:** Gradio mode (run_gradio_demo.py:8-9) uses module-global all_messages and OPENAI_KEY shared across all sessions; not the scored mode.

### C7 Third-party extensions — 0.05 (high)

In the default hybrid mode, the local model server loads about thirty third-party model repositories that the setup script clones from Hugging Face at whatever their latest revision is, with no pinning or hash check. At least one checkpoint is loaded with torch.load, which unpickles and can run arbitrary code, and other weights load in the same process. That process runs as the operator and reads the same config file that holds the API keys, so a tampered model file gets everything.

- **S L0:** Model weights are cloned unpinned (git clone/pull of latest) and at least one is deserialised with torch.load (pickle) without weights_only. — [hugginggpt/server/models_server.py:279](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L279); [hugginggpt/server/models/download.sh:40-45](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models/download.sh#L40-L45) (verified)
  - *To reach the next level:* No version pinning of downloaded model repos.
- **C L0:** No downloaded model type is verified; hub ids are also loaded directly by from_pretrained without a revision. — [hugginggpt/server/models_server.py:118](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L118); searched `rg -n -S 'sha256|hashlib|revision=' -g *.py` in `hugginggpt/server` → 0 hits (verified)
  - *To reach the next level:* No verification for any model source.
- **D L1:** The model list is fixed in the repo and the operator runs download.sh explicitly, but re-running it git-pulls whatever is new without showing changes; hybrid mode requires the local server. — [hugginggpt/server/models/download.sh:40-45](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models/download.sh#L40-L45); [hugginggpt/server/configs/config.default.yaml:15](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L15) (verified)
  - *To reach the next level:* Install does not show what will run or detect changed model files.
- **B L0:** Weights deserialise inside the model server process, same user, which loads the config holding the API keys. — [hugginggpt/server/models_server.py:57](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/models_server.py#L57) (verified)
  - *To reach the next level:* No separate, scrubbed or sandboxed process for model loading.
- **Cap:** none

### C8 Secrets & sensitive-data protection — 0.00 (high)

The README tells users to paste their OpenAI key and Hugging Face token into the tracked YAML config, with environment variables as an alternative; nothing is masked anywhere. A debug log file at full verbosity is on by default and records every prompt and model response. Outbound credential handling is not locked down. The web client stores the user's key in browser localStorage.

- **S L0:** Keys live in plaintext in the repo-tracked config (or env), their outbound handling is not locked down, and no masking exists. — [hugginggpt/server/configs/config.default.yaml:2](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L2); [hugginggpt/server/awesome_chat.py:105-108](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L105-L108) (verified)
  - *To reach the next level:* Secrets are not kept out of committed config and no redaction exists.
- **C L0:** No path (logs, outbound requests, web storage) is protected. — [hugginggpt/server/awesome_chat.py:389](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L389); [hugginggpt/web/src/views/home.vue:254](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/web/src/views/home.vue#L254) (verified)
  - *To reach the next level:* No path has redaction or protection.
- **D L0:** Full prompt/response logging to logs/debug.log at DEBUG level is on by default. — [hugginggpt/server/configs/config.default.yaml:12](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L12); [hugginggpt/server/awesome_chat.py:56-61](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L56-L61); [hugginggpt/server/awesome_chat.py:209](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L209) (verified)
  - *To reach the next level:* Verbose payload logging is on by default.
- **B L0:** Long-lived, unscoped OpenAI and Hugging Face keys are exposed because their handling is not locked down. (verified)
  - *To reach the next level:* Keys are long-lived and unscoped.
- **Cap:** none

### C9 Audit & traceability — 0.30 (high)

The server appends a JSON line per request to log files with the input, task plan, model choices, results and response, plus a full-verbosity debug log. The JSON record has no timestamp, no caller identity, and is only written at the end of a successful run, and the /tasks and /results endpoints, which also execute work, return before any success record is written. Logs live in the server's working directory where the agent has no tool to edit them.

- **S L1:** record_case writes per-request JSON (input, tasks, results) but no timestamps or actor; per-task details are only in unstructured debug lines. — [hugginggpt/server/awesome_chat.py:245-252](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L245-L252); [hugginggpt/server/awesome_chat.py:730](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L730) (verified)
  - *To reach the next level:* No structured per-call record with timestamps.
- **C L1:** /hugginggpt records results, but /results executes every task and returns before record_case, and /tasks returns early too. — [hugginggpt/server/awesome_chat.py:929-930](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L929-L930); [hugginggpt/server/awesome_chat.py:965-966](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L965-L966) (verified)
  - *To reach the next level:* Not every execution path writes a record.
- **D L2:** Logging is on by default to logs/ under the server's cwd; no model-reachable tool writes there, but the server process can. — [hugginggpt/server/configs/config.default.yaml:12](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/configs/config.default.yaml#L12); [hugginggpt/server/awesome_chat.py:56-61](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L56-L61) (verified)
  - *To reach the next level:* Not written by a component separate from the agent process.
- **B L1:** The success record is written once at the end of a run, so a crash or hang loses it; logging failures are not handled. — [hugginggpt/server/awesome_chat.py:974](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L974) (verified)
  - *To reach the next level:* Records are not flushed per action.
- **Cap:** none

### C10 Limits & kill switch — 0.20 (high)

The pipeline has a fixed shape (plan, choose, execute, respond), and a dependency-wait loop gives up after about 80 seconds, but nothing bounds how many tasks the model plans, each of which gets its own thread and paid model calls. Most outbound requests have no timeout, there is no cost cap, no rate limit, and no way to cancel a running request. Nothing limits how many runs callers start on the operator's key.

- **S L1:** Only a wait-loop cap (160 x 0.5 s) and per-call max_tokens exist; no task-count, wall-clock, or cost cap, and no halt. — [hugginggpt/server/awesome_chat.py:951-956](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L951-L956); [hugginggpt/server/awesome_chat.py:186](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L186) (verified)
  - *To reach the next level:* No enforced per-execution timeout or cost cap alongside the loop cap.
- **C L1:** The wait cap covers only the scheduling loop; spawned task threads are still joined and most HTTP calls have no timeout. — [hugginggpt/server/awesome_chat.py:539](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L539); [hugginggpt/server/awesome_chat.py:959-960](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L959-L960) (verified)
  - *To reach the next level:* Tool calls lack timeouts.
- **D L1:** The limits that exist are hard-coded; the model controls how many tasks it emits. — [hugginggpt/server/awesome_chat.py:947-950](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L947-L950) (verified)
  - *To reach the next level:* Model-controlled task count is not bounded by a default.
- **B L0:** Callers can start unlimited runs, each with an unbounded number of tasks, on the operator's account. — [hugginggpt/server/awesome_chat.py:947-950](https://github.com/microsoft/JARVIS/blob/7624cf388b47334ff8a0868e7d862dde18cfda86/hugginggpt/server/awesome_chat.py#L947-L950) (verified)
  - *To reach the next level:* No ceiling on runs or spend.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: remote model outputs and user-named media URLs enter prompts (hugginggpt/server/awesome_chat.py:382-385, :515) · [B] sensitive data/systems: operator OpenAI/HF keys (awesome_chat.py:105-108) · [C] state change / egress: server-side GET to any URL (awesome_chat.py:515) and further unattended egress paths · Same default session? Yes

## Highest-impact improvements
1. Harden default network exposure and outbound credential handling. — C8 B L0→L2, +0.100 before caps (Playbook 4)
2. Require inbound authentication on all routes and drop client-supplied api_key passthrough. — C1 C L0→L2, +0.150 before caps (Playbook 4)
3. Restrict resource arguments to http(s) on an allowlist with internal-address blocking, and harden local-path handling. — C3 S L1→L3, +0.150 before caps (Playbook 3)
4. Harden the web client's handling of model output. — C5 B L0→L1, +0.050 before caps (Playbook 1)
5. Pin model repos to commit hashes and load with weights_only=True / safetensors. — C7 S L0→L2, +0.150 before caps (Playbook 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored component is hugginggpt/ (server mode). easytool/ (which eval()s LLM output) and taskbench/ benchmark scripts were not scored; CLI and Gradio modes are footnoted only.
- Behaviour of third-party libraries (from_pretrained loading .bin pickles) is inferred from their documented behaviour, not read in this repo.
- No text aimed at AI reviewers was found in the repository.
