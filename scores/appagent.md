# Defense-in-Depth Score: AppAgent

**Repo:** https://github.com/tencentqqgylab/appagent · **Commit:** `2c1900422caf6f9e94e96d5dd984b530e5a5fbf8` · **Reviewed:** 2026-10-04
**What it is:** Research multimodal LLM agent (Tencent QQGY Lab) that operates Android smartphone apps over adb by tapping, typing and swiping, with an exploration phase that writes per-element UI documentation.
**Category:** AI Assistants
**Scored configuration:** python run.py (deployment phase, scripts/task_executor.py) with the shipped config.yaml (OpenAI model, MAX_ROUNDS 20) against a connected Android device; learn.py exploration phase footnoted.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory yes · untrusted input yes · third party extensions no · sub agents no · external communication yes

## Score: 2.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C2 | Approval gates | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C3 | Tool & action scoping | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L0 | 0.00 | — | **0.00** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L0 | L0 | L1 | L1 | 0.10 | — | **0.10** | High |
| C9 | Audit & traceability | L1 | L2 | L1 | L2 | 0.38 | — | **0.38** | High |
| C10 | Limits & kill switch | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |

Controls where a risk surface exists: 1.07 / 9.0 (12%); 1 criterion scored SA (surface absent).

AppAgent gives a vision LLM unsupervised control of a real Android phone: every round it reads whatever is on screen and taps, types or swipes with no human approval, so on-screen content from any app can steer it into sending messages or other irreversible actions in the user's logged-in accounts. Host-side handling of model-generated action arguments is also not strict. The only real limit is a 20-round cap; there is no sandbox, no argument validation, no redaction, and model-written UI documentation persists and is re-injected as trusted guidance in later runs.

## Critical gaps
- The agent controls the user's entire phone and host account with no scoping or authorization check. (ASI03, T3; C1) — [scripts/config.py:6-9](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/config.py#L6-L9)
- Untrusted on-screen content from any app can drive unattended exfiltration and irreversible phone actions; no approval exists. (ASI01, T6, LLM01; C5) — [scripts/task_executor.py:207](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L207); [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236)

## Criterion details

### C1 Identity & least privilege — 0.00 (high)

The agent acts with the operator's full ambient authority on two fronts: adb gives it complete control of the connected phone and every account logged in there, and its host subprocesses run as the operator's OS user with the full environment inherited. There is no scoped identity, no per-action authorization check, and no narrowing of either authority. A hijacked agent therefore holds the user's whole phone, and host-side handling is not locked down either.

- **S L0:** Actions run through adb with full device control and host subprocesses as the OS user; no scoped identity exists. — [scripts/config.py:6-9](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/config.py#L6-L9) (verified)
  - *To reach the next level:* No dedicated or role-scoped identity; the agent uses adb's full device authority and the OS user's ambient authority.
- **C L0:** No authorization layer exists; every action goes straight to adb via execute_adb. — searched `rg -n -i 'confirm|approv'` in `scripts run.py learn.py` → 0 hits (No approval or confirmation step anywhere in the action path.) (verified)
  - *To reach the next level:* No authorization check on any action path.
- **D L0:** Default install runs with full device and OS-user authority; there is no narrower mode. — [scripts/config.py:6-9](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/config.py#L6-L9) (verified)
  - *To reach the next level:* No narrower default role or read-only mode exists.
- **B L0:** A hijack reaches the user's entire phone (messaging, email, payments in logged-in apps); host-side handling is not locked down either. (verified)
  - *To reach the next level:* Blast radius is not limited to one system or to read-only operations.
- **Cap:** none

### C2 Approval gates — 0.00 (high)

There is no approval step anywhere. Each round the model's chosen tap, text, long-press or swipe is parsed and executed immediately over adb, and the loop continues until the model says FINISH or 20 rounds pass. Consequential phone actions such as sending a message, confirming a purchase or deleting data happen without the user seeing them first.

- **S L0:** No approval mechanism; parsed actions execute directly. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236); searched `rg -n -i 'confirm|approv'` in `scripts run.py learn.py` → 0 hits (No approval or confirmation step anywhere in the action path.) (verified)
  - *To reach the next level:* No per-call human approval of actions.
- **C L0:** Every action path, including the text action, is ungated. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236); searched `rg -n -i 'confirm|approv'` in `scripts run.py learn.py` → 0 hits (No approval or confirmation step anywhere in the action path.) (verified)
  - *To reach the next level:* No action path traverses a gate.
- **D L0:** No approval exists to enable. — searched `rg -n -i 'confirm|approv'` in `scripts run.py learn.py` → 0 hits (No approval or confirmation step anywhere in the action path.) (verified)
  - *To reach the next level:* Approval is not on by default (it does not exist).
- **B L0:** Ungated actions include irreversible sends, purchases and deletions on a real phone, with no undo. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* No checkpoints, previews, or reversibility for phone or host actions.
- **Cap:** none

### C3 Tool & action scoping — 0.15 (high)

The action vocabulary is small (tap an element index, type text, long-press, swipe, grid taps), and some arguments are parsed as integers or checked against a fixed set of swipe directions. But the typed text argument receives little validation and its handling is not strict. Element indices are not bounds-checked, and all actions are enabled in every run.

- **S L1:** Area indices go through int() and swipe directions through an if/else set, but handling of the text argument is not strict. — [scripts/model.py:122](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/model.py#L122) (verified)
  - *To reach the next level:* No allowlist for the text argument; indices are not bounds-checked.
- **C L1:** Only tap/swipe indices and swipe direction get any parsing; text, the most dangerous argument, is effectively unvalidated. — [scripts/model.py:122](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/model.py#L122) (verified)
  - *To reach the next level:* Most actions, including text, are not validated against an allowlist.
- **D L0:** All actions, including text into any field, are enabled in every run. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* No reduced or per-task action set; everything is on by default.
- **B L0:** Misused actions reach any app on the phone. (verified)
  - *To reach the next level:* Actions are not scoped to one app or bounded in quantity.
- **Cap:** none
- **Notes:** D is L0 because the full action set is always on (least-agency posture), not because argument parsing is opt-in; G1 does not apply. Considered whether int() parsing and the swipe-direction check justify S L2 and kept L1 because the most dangerous argument (text) is effectively raw.

### C4 Code-execution isolation — 0.00 (high)

Actions are carried out by invoking adb as a host subprocess under the operator's user, with the API key and full environment available. Handling of model-generated arguments on that path is not strict. There is no sandbox of any kind.

- **S L0:** adb is invoked on the host as the same user via subprocess; no isolation primitive exists. (verified)
  - *To reach the next level:* No isolation boundary (container or OS sandbox) around adb invocation.
- **C L0:** The only exec path (execute_adb) is unsandboxed. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* No execution path is sandboxed.
- **D L0:** No sandbox exists to enable. (verified)
  - *To reach the next level:* Isolation is not on by default.
- **B L0:** Host subprocesses run as the operator's user with full host filesystem, network, the inherited environment and config.yaml holding the API key. — [scripts/config.py:6-9](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/config.py#L6-L9); [config.yaml:4](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/config.yaml#L4) (verified)
  - *To reach the next level:* Execution is host-equivalent with credentials reachable.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The agent's entire input is untrusted: screenshots and UI XML of whatever app is open, including incoming messages, web pages, notifications and ads, go to the model every round with the same standing as the user's task. Nothing separates that content from instructions or restricts what follows once it is read. A successful injection can make the agent type private data into a message or browser (exfiltration), take irreversible actions in logged-in apps, all without a human in the loop.

- **S L0:** No structural limit; screen content goes into the same single user message as the task. — [scripts/task_executor.py:207](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L207); [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* No Rule-of-Two enforcement or approval once untrusted screen content is read.
- **C L0:** Screen content from any app is not distinguished from the principal's task. — [scripts/task_executor.py:207](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L207) (verified)
  - *To reach the next level:* No untrusted source is distinguished.
- **D L0:** No control exists to enable. — [scripts/task_executor.py:207](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L207) (verified)
  - *To reach the next level:* No untrusted-input control on by default.
- **B L0:** A hijacked agent can leak on-device data by typing it into messaging/browser apps and take irreversible actions, unattended. — [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* Exfiltration and irreversible actions both happen without a human.
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.

### C6 Memory, context & configuration integrity — 0.10 (high)

The exploration phase saves model-written descriptions of UI elements to apps/<app>/auto_docs (or demo_docs) as plain files keyed by element ID, and every later deployment run injects them into the prompt with an instruction to always prioritize them. Nothing validates, reviews or tags these entries, so text injected during exploration (for example, by a malicious app's screen) persists and steers tool use in future sessions. The files are local, readable and deletable by the user, and they are parsed with ast.literal_eval, which does not execute code.

- **S L0:** Model output is written verbatim to doc files and re-injected as trusted guidance. — [scripts/self_explorer.py:246-248](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/self_explorer.py#L246-L248); [scripts/task_executor.py:178-203](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L178-L203) (verified)
  - *To reach the next level:* No validation, review, or provenance on doc writes.
- **C L0:** Neither auto_docs nor demo_docs writes are controlled. — [scripts/self_explorer.py:246-248](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/self_explorer.py#L246-L248); [scripts/document_generation.py:121-127](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/document_generation.py#L121-L127) (verified)
  - *To reach the next level:* No memory path is controlled.
- **D L1:** Docs are namespaced only by the app name in a local directory; there is no per-user or per-session isolation enforced in code. — [scripts/self_explorer.py:58](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/self_explorer.py#L58) (verified)
  - *To reach the next level:* No per-session or per-user namespace and no retention limit.
- **B L1:** Poisoned docs persist across the user's sessions and influence which elements the model taps; they are plain files the user can inspect. — [scripts/task_executor.py:178-203](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L178-L203) (verified)
  - *To reach the next level:* Persisted docs can still trigger ungated actions, not only text output.
- **Cap:** none

### C7 Third-party extensions — 1.00 (high)

The agent loads no third-party code at runtime: no plugins, MCP servers, dynamic imports, model-file loading or model-chosen package installs. Its Python dependencies are a build-time concern outside this criterion.

- **Structural absence:** searched `rg -n -i 'plugin|mcp|importlib|pickle|trust_remote|pip install|__import__'` in `scripts run.py learn.py` → 0 hits (No plugin, MCP, dynamic import, model-file deserialization, or package-install path.)

### C8 Secrets & sensitive-data protection — 0.10 (high)

The API key is meant to be pasted into the tracked config.yaml in plaintext, and that file overrides environment variables, so the file is effectively the only way to supply it. The whole process environment is merged into the config object and inherited by every adb subprocess. Logs store full prompts and model responses (not the key), and every phone screenshot, which may show private messages or financial data, is saved locally and sent to the model provider. There is no redaction on any path.

- **S L0:** The API key lives in plaintext in the tracked config.yaml, which overrides env vars; no masking exists. — [config.yaml:4](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/config.yaml#L4); [scripts/config.py:6-9](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/config.py#L6-L9); searched `rg -n -i 'redact|mask|secret'` in `scripts run.py learn.py` → 0 hits (No redaction or secret handling code.) (verified)
  - *To reach the next level:* Secrets are not taken from an env var or secret store, and nothing is masked.
- **C L0:** No path (logs, model-bound screenshots, subprocess env) is protected. — searched `rg -n -i 'redact|mask|secret'` in `scripts run.py learn.py` → 0 hits (No redaction or secret handling code.); [scripts/task_executor.py:210-213](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L210-L213) (verified)
  - *To reach the next level:* No path has redaction.
- **D L1:** No telemetry; full prompt/response logging and screenshot retention are on by default without redaction. — [scripts/task_executor.py:210-213](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L210-L213) (verified)
  - *To reach the next level:* Payload logging is on by default and unredacted.
- **B L1:** A long-lived model API key is stored in plaintext in config.yaml on the host. — [config.yaml:4](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/config.yaml#L4) (verified)
  - *To reach the next level:* The key is long-lived and not short-lived or scoped per task.
- **Cap:** none

### C9 Audit & traceability — 0.38 (high)

Each round's prompt, image filename and raw model response are appended as a JSON line to a log in the run's tasks directory, before the chosen action is executed, and screenshots are kept alongside. That lets someone reconstruct what the model decided, but the log has no per-step timestamp, no record of the actual adb command or its result, and no actor attribution. It sits in a plain local directory writable by the same user.

- **S L1:** Unstructured-ish JSONL of prompt and raw response per step; no executed command, result status, or per-entry timestamp. — [scripts/task_executor.py:210-213](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L210-L213) (verified)
  - *To reach the next level:* No structured record of executed actions with arguments, result status and timestamps.
- **C L2:** Every model decision in the deployment and exploration loops is logged, so all built-in actions are covered. — [scripts/task_executor.py:210-213](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L210-L213); [scripts/self_explorer.py:131-134](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/self_explorer.py#L131-L134) (verified)
  - *To reach the next level:* No record of approvals (none exist), configuration or memory writes.
- **D L1:** On by default in root_dir/tasks, writable by the same user the agent's host subprocesses run as. — [scripts/task_executor.py:46](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L46) (verified)
  - *To reach the next level:* Logs are not written by a component the agent's execution path cannot alter.
- **B L2:** The log is opened, written and closed per step before the action runs, so a write failure raises and stops the run. — [scripts/task_executor.py:210-213](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L210-L213); [scripts/task_executor.py:234-236](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L234-L236) (verified)
  - *To reach the next level:* No durable replayable record of executed actions and their results.
- **Cap:** none

### C10 Limits & kill switch — 0.35 (high)

The agent stops after MAX_ROUNDS (20 by default) and sleeps REQUEST_INTERVAL (10 seconds) between actions, which bounds how much it can do in one run. There is no wall-clock limit, no cost cap beyond a per-response token limit, and no timeout on the model HTTP request or adb subprocesses. Stopping is Ctrl-C on the foreground process; nothing is scheduled to keep running afterwards.

- **S L1:** An iteration cap enforced in code plus a fixed sleep between actions; no wall-clock, timeout, or cost budget. — [scripts/task_executor.py:142](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L142); [scripts/task_executor.py:281](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L281); searched `rg -n -i 'timeout'` in `scripts run.py learn.py` → 0 hits (No timeout on the model HTTP call or on adb subprocesses.) (verified)
  - *To reach the next level:* No wall-clock limit, per-execution timeout, or token/cost budget alongside the round cap.
- **C L1:** The round cap covers the top-level loop only; HTTP and subprocess calls have no timeouts. — [scripts/task_executor.py:142](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L142); [scripts/model.py:60](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/model.py#L60); searched `rg -n -i 'timeout'` in `scripts run.py learn.py` → 0 hits (No timeout on the model HTTP call or on adb subprocesses.) (verified)
  - *To reach the next level:* No tool or request timeouts.
- **D L2:** Sensible default of 20 rounds in config.yaml, operator-configurable, not model-modifiable. — [config.yaml:17](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/config.yaml#L17) (verified)
  - *To reach the next level:* Limits are not shown to be immune to tampering (config.yaml is writable by the same host user) and no hard ceiling exists.
- **B L2:** At most about 20 actions per run spaced 10 seconds apart; Ctrl-C ends the foreground process. — [config.yaml:17](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/config.yaml#L17); [scripts/task_executor.py:281](https://github.com/tencentqqgylab/appagent/blob/2c1900422caf6f9e94e96d5dd984b530e5a5fbf8/scripts/task_executor.py#L281) (verified)
  - *To reach the next level:* No cost ceiling and in-flight calls are not cancelled by design.
- **Cap:** none
- **Notes:** Considered crediting MAX_TOKENS (300 per call) as a token cap for S L2; kept L1 because it bounds a single response, not the run's spend.

## Rule-of-Two check
[A] untrusted input: Screenshots and UI XML of any app screen sent each round (scripts/task_executor.py:207) · [B] sensitive data/systems: Logged-in phone apps via adb plus host env/config.yaml API key (scripts/config.py:6, config.yaml:4) · [C] state change / egress: Ungated tap/text/swipe over adb (scripts/task_executor.py:236) · Same default session? Yes

## Highest-impact improvements
1. Run adb invocations inside an isolation boundary with a scrubbed environment so model-driven actions cannot affect the host. — C4 B L0→L2, +0.100 before caps (Playbook 3 step 1)
2. Validate the text action against an allowlist and bounds-check element indices before execution. — C3 S L1→L3, +0.150 before caps (Playbook 3)
3. Show each parsed action (exact text and target element) and require y/n before executing, at least for text input and any tap after typing. — C2 S L0→L3, +0.225 before caps (Playbook 5)
4. Read the API key from an environment variable that takes precedence over config.yaml, and pass a scrubbed env to subprocesses. — C8 S L0→L1, +0.075 before caps (Playbook 4)
5. Add timeouts to the model request and adb subprocesses and a wall-clock limit per run. — C10 S L1→L2, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 2c19004 only; nothing was executed, installed, or probed.
- Some implementation findings were reasoned from the code path and not demonstrated.
- Exploration phase (learn.py) shares the same controller and was reviewed but the deployment phase is what is scored.
- Model behaviour is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found in the repo.
