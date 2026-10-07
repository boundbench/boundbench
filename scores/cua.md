# Defense-in-Depth Score: Cua

**Repo:** https://github.com/trycua/cua · **Commit:** `2bce4442c107fc34c45b374b29a35d09f630fd83` · **Reviewed:** 2026-10-03
**What it is:** Computer-use agent infra: sandboxes/drivers, agent SDK, benchmarks
**Category:** AI Assistants
**Scored configuration:** cua-agent ComputerAgent with default constructor arguments and a hosted computer-use model, driving Sandbox.ephemeral(Image.linux()) with default placement (local, kind=auto, runtime=auto).
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions opt-in · sub agents no · external communication yes

## Score: 3.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L1 | L2 | L2 | 0.35 | — | **0.35** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | High |
| C3 | Tool & action scoping | L0 | L0 | L0 | L2 | 0.10 | — | **0.10** | High |
| C4 | Code-execution isolation | L3 | L2 | L2 | L2 | 0.57 | — | **0.57** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C7 | Third-party extensions | L1 | L1 | L1 | L0 | 0.20 | C7-RCELOAD | **0.20** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L0 | L2 | 0.40 | G1 | **0.40** | High |
| C10 | Limits & kill switch | L1 | L1 | L0 | L0 | 0.15 | G1 | **0.15** | High |

Controls where a risk surface exists: 2.07 / 9.0 (23%); 1 criterion scored SA (surface absent).

Cua puts its computer-use agent inside a well-built, disposable sandbox (gVisor when available, no host mounts, resource limits), so a hijacked agent cannot touch your own machine. But the agent loop itself has almost no safety controls: no human approval, no step or time limit by default, unvalidated actions, and it auto-acknowledges the model provider's safety checks. A malicious web page seen in the sandbox can steer the agent to leak data or act on the web unattended.

## Critical gaps
- A hijacked session can browse to any URL and act on the web unattended; provider safety checks are auto-acknowledged. (ASI01, ASI09; C5) — [libs/python/agent/cua_agent/agent.py:795-800](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L795-L800); [libs/python/agent/cua_agent/tools/browser_tool.py:367-373](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/tools/browser_tool.py#L367-L373)
- The Moondream3 loop and the shipped CLI load unpinned Hugging Face repos with trust_remote_code=True, running remote code in the agent process. (ASI04; C7) — [libs/python/agent/cua_agent/loops/moondream3.py:47-49](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/moondream3.py#L47-L49); [libs/python/agent/cua_agent/cli.py:398](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/cli.py#L398)

## Criterion details

### C1 Identity & least privilege — 0.35 (high)

The agent's GUI actions land inside a sandbox guest that, by default, holds no credentials of the user's, which keeps the model away from the operator's own accounts. But the framework has no authorization layer of its own: developer-registered function tools run in the host Python process with whatever the process can reach, and the guest itself runs as root. Access control on the optional playground server is not locked down.

- **S L1:** Model actions run as root inside an isolated guest with no injected credentials; function tools inherit the host process's full authority. — [libs/python/agent/cua_agent/agent.py:758-763](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L758-L763); [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858); searched `rg -n '^USER'` in `libs/images/linux/Dockerfile` → 0 hits (The canonical Linux image never switches away from root.) (verified)
  - *To reach the next level:* No per-tool or per-capability credential scoping and no authorization check before a tool runs.
- **C L1:** Computer actions go to the sandbox; developer function tools run in the host process via asyncio.to_thread with ambient authority. — [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858) (verified)
  - *To reach the next level:* Function tools, the playground server and the CLI are not routed through any shared authorization layer.
- **D L2:** By default the sandbox gets only caller-supplied env (none in the documented flow) and no host mounts; widening is an operator choice. — [libs/cua/crates/cua-vmm/src/container/mod.rs:1592-1593](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L1592-L1593); searched `rg -n 'binds:|mounts:|privileged:'` in `libs/cua/crates/cua-vmm/src/container/mod.rs` → 0 hits (The container create body sets no bind mounts, mounts or privileged flag.) (verified)
  - *To reach the next level:* No read-only default; the guest is a full root desktop.
- **B L2:** A hijacked agent controls one sandbox desktop with open egress; the host-held model and Cua keys are not exposed to it. — [libs/python/cua-sandbox/cua_sandbox/sandbox.py:1403-1406](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/cua-sandbox/cua_sandbox/sandbox.py#L1403-L1406); [libs/python/agent/cua_agent/computers/sandbox.py:162](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/computers/sandbox.py#L162) (verified)
  - *To reach the next level:* Writes inside the guest are unrestricted and not tenant- or time-scoped.
- **Cap:** none
- **Notes:** Access control on the opt-in playground server (agent.open()) is not locked down.

### C2 Approval gates — 0.05 (high)

There is no human approval step anywhere in the agent loop: every click, keystroke, URL visit and registered function call the model emits is executed immediately. When OpenAI's computer-use model raises its own pending safety checks (for example, a suspected malicious instruction on screen), the framework acknowledges them automatically; the confirmation hook is left as a TODO. The only thing limiting consequences is that the default sandbox is thrown away at the end, which does not undo anything the agent did on the web.

- **S L0:** No approval primitive; provider safety checks are auto-acknowledged in code. — searched `rg -n -i 'approv|confirm'` in `libs/python/agent/cua_agent/agent.py libs/python/agent/cua_agent/callbacks libs/python/agent/cua_agent/computers libs/python/agent/cua_agent/tools` → 0 hits (No approval or confirmation primitive anywhere in the agent loop, callbacks, computer handlers or tools.); [libs/python/agent/cua_agent/agent.py:795-800](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L795-L800); [libs/python/agent/cua_agent/agent.py:800](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L800) (verified)
  - *To reach the next level:* Per-call human approval of the exact action is needed for L2-L3.
- **C L0:** The most powerful paths (typing into a desktop, visit_url, function tools) execute without any gate. — [libs/python/agent/cua_agent/agent.py:758-763](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L758-L763); [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858); [libs/python/agent/cua_agent/tools/browser_tool.py:367-373](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/tools/browser_tool.py#L367-L373) (verified)
  - *To reach the next level:* A gate on the computer and function dispatch paths is needed for any coverage.
- **D L0:** Nothing to turn on; no approval mode exists. — searched `rg -n -i 'approv|confirm'` in `libs/python/agent/cua_agent/agent.py libs/python/agent/cua_agent/callbacks libs/python/agent/cua_agent/computers libs/python/agent/cua_agent/tools` → 0 hits (No approval or confirmation primitive anywhere in the agent loop, callbacks, computer handlers or tools.) (verified)
  - *To reach the next level:* An on-by-default approval mode is needed.
- **B L1:** In-guest changes vanish with the ephemeral sandbox, but web submissions, messages and purchases made from the guest are irreversible. — [libs/python/agent/cua_agent/computers/sandbox.py:162](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/computers/sandbox.py#L162); [libs/python/cua-sandbox/cua_sandbox/sandbox.py:1403-1406](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/cua-sandbox/cua_sandbox/sandbox.py#L1403-L1406) (verified)
  - *To reach the next level:* No previews, dry-runs or checkpoints for external actions taken through the desktop.
- **Cap:** none

### C3 Tool & action scoping — 0.10 (high)

The tools are general by design: arbitrary keystrokes and text into a full desktop, clicks anywhere, and a browser tool that visits any URL. The only argument check is that the call's parameter names match the Python signature; values (text, keys, URLs, coordinates) are passed through unvalidated, and every action is enabled by default. Damage is scoped to the sandbox guest, but inside it the agent can do anything a root user at the keyboard can.

- **S L0:** Raw passthrough: typed text, key chords and URLs are forwarded without validation; only signature binding is checked. — [libs/python/agent/cua_agent/agent.py:761](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L761); [libs/python/agent/cua_agent/tools/browser_tool.py:367-373](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/tools/browser_tool.py#L367-L373) (verified)
  - *To reach the next level:* Typed schemas with value validation (URL/host allowlists, bounded quantities) are needed for L2.
- **C L0:** No tool validates argument values. — [libs/python/agent/cua_agent/agent.py:761](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L761); [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858) (verified)
  - *To reach the next level:* At least some tools must validate values in code.
- **D L0:** Full desktop control and URL navigation are on whenever a computer is passed; no narrower default set. — [libs/python/agent/cua_agent/agent.py:758-763](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L758-L763) (verified)
  - *To reach the next level:* A read-only or narrower default action set is needed.
- **B L2:** Reach is one sandbox guest with full write inside it and open egress. — [libs/python/cua-sandbox/cua_sandbox/sandbox.py:1403-1406](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/cua-sandbox/cua_sandbox/sandbox.py#L1403-L1406); searched `rg -n 'binds:|mounts:|privileged:'` in `libs/cua/crates/cua-vmm/src/container/mod.rs` → 0 hits (The container create body sets no bind mounts, mounts or privileged flag.) (verified)
  - *To reach the next level:* No quantity bounds (actions, recipients, spend) inside the guest.
- **Cap:** none

### C4 Code-execution isolation — 0.57 (high)

This is the project's strongest area. The documented computer is a sandbox that, by default, runs under gVisor when the container engine has it, and Cua's own Mac runtime ships with gVisor built in; it mounts no host directories, publishes ports only on 127.0.0.1, sets CPU and memory limits and is destroyed when the run ends. Weak points: if gVisor is missing the SDK falls back to a plain runc container with only a log warning, the guest runs as root with unrestricted outbound network, and developer-registered function tools run on the host.

- **S L3:** gVisor (kernel-separated) is preferred, but the auto default falls back to a stock runc container (root, default capabilities). — [libs/cua/crates/cua-vmm/src/container/mod.rs:121-130](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L121-L130); [libs/cua/crates/cua-vmm/src/container/mod.rs:254-258](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L254-L258); [libs/cua/crates/cua-sandbox-core/src/placement.rs:862-870](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-sandbox-core/src/placement.rs#L862-L870); [libs/cua/crates/cua-vmm/src/container/mod.rs:1604-1615](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L1604-L1615) (verified)
  - *To reach the next level:* The fallback is an unhardened container; L4 needs kernel-separated isolation on every default path.
- **C L2:** All model actions on the computer go to the sandbox and the localhost provider was removed, but registered function tools run in the host process. — [libs/python/agent/cua_agent/computers/sandbox.py:145-148](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/computers/sandbox.py#L145-L148); [libs/python/agent/cua_agent/agent.py:758-763](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L758-L763); [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858) (verified)
  - *To reach the next level:* Registered tools run outside the sandbox; L3 needs every model-reachable path sandboxed.
- **D L2:** Sandboxing is on by default, but a missing runsc silently degrades to runc with only a log warning (require_gvisor defaults to false). — [libs/cua/crates/cua-vmm/src/container/mod.rs:121-130](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L121-L130); [libs/cua/crates/cua-vmm/src/container/mod.rs:254-258](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L254-L258) (verified)
  - *To reach the next level:* Degrading to the weaker runtime should fail closed or need an explicit operator flag.
- **B L2:** No host mounts or injected secrets, CPU/memory limits and an ephemeral lifetime, but root inside and unrestricted egress (network='none' only for local QEMU VMs). — searched `rg -n 'binds:|mounts:|privileged:'` in `libs/cua/crates/cua-vmm/src/container/mod.rs` → 0 hits (The container create body sets no bind mounts, mounts or privileged flag.); [libs/cua/crates/cua-vmm/src/container/mod.rs:1612](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L1612); [libs/python/cua-sandbox/cua_sandbox/sandbox.py:1403-1406](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/cua-sandbox/cua_sandbox/sandbox.py#L1403-L1406); searched `rg -n '^USER'` in `libs/images/linux/Dockerfile` → 0 hits (The canonical Linux image never switches away from root.) (verified)
  - *To reach the next level:* Egress must be off or allowlisted for L3.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

A computer-use agent reads whatever is on screen, including web pages an attacker controls, and the framework gives that content the same standing as the user's request. Nothing marks it as untrusted, and nothing limits what the agent may do after seeing it: it can browse to any URL (an exfiltration channel) and submit forms or messages without a person in the loop. The model provider's own safety checks are acknowledged automatically.

- **S L0:** No structural limit; provider-raised safety checks are auto-acknowledged rather than surfaced. — [libs/python/agent/cua_agent/agent.py:795-800](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L795-L800); [libs/python/agent/cua_agent/agent.py:800](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L800) (verified)
  - *To reach the next level:* Approval for egress or state change after untrusted content is read is needed for L2.
- **C L0:** Screenshots and tool results enter context without provenance or separation. — [libs/python/agent/cua_agent/agent.py:816-823](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L816-L823); [libs/python/agent/cua_agent/agent.py:852-858](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L852-L858) (verified)
  - *To reach the next level:* Untrusted sources must at least be distinguished from principal input.
- **D L0:** No control exists to be on by default. — searched `rg -n -i 'approv|confirm'` in `libs/python/agent/cua_agent/agent.py libs/python/agent/cua_agent/callbacks libs/python/agent/cua_agent/computers libs/python/agent/cua_agent/tools` → 0 hits (No approval or confirmation primitive anywhere in the agent loop, callbacks, computer handlers or tools.) (verified)
  - *To reach the next level:* Any on-by-default limit is needed.
- **B L0:** Rule of Two fully combined: untrusted pages, user task data in the guest, and arbitrary URL egress plus irreversible web actions, all unattended. — [libs/python/agent/cua_agent/tools/browser_tool.py:367-373](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/tools/browser_tool.py#L367-L373); [libs/python/cua-sandbox/cua_sandbox/sandbox.py:1403-1406](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/cua-sandbox/cua_sandbox/sandbox.py#L1403-L1406); [libs/python/agent/cua_agent/agent.py:758-763](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L758-L763) (verified)
  - *To reach the next level:* Either exfiltration or irreversible actions must require a human.
- **Cap:** C5-WORSTCASE — B is L0: a hijacked session can leak data and take irreversible web actions with no human involved.

### C6 Memory, context & configuration integrity — 1.00 (high)

The agent SDK keeps no memory between runs: no memory store, vector index or saved summaries are read back into the model's context, and it auto-loads no instruction or settings files from the environment it operates on. The agent acts inside a sandbox and cannot write the host files that configure it. The shipped CLI and Gradio UI call load_dotenv() on the host, which matters only if the user's own project directory is untrusted.

- **Structural absence:** searched `rg -n -i 'chromadb|faiss|vector_store|sqlite3|shelve|save_memory|pickle'` in `libs/python/agent/cua_agent` → 0 hits (No persistent memory or retrieval store in the agent package.); searched `rg -n -i 'AGENTS\.md|CLAUDE\.md|cursorrules'` in `libs/python/agent/cua_agent` → 0 hits (No instruction files auto-loaded into context.); searched `rg -n -i 'load_dotenv'` in `libs/python/agent/cua_agent` → 3 hits (All three are host-side entrypoints (cli.py:41, ui/gradio/app.py:46, proxy/examples.py:7), not the ComputerAgent library path; the agent cannot write them from the sandbox.)
- **Notes:** Persistent (non-ephemeral) sandboxes keep guest desktop state across runs; that is environment state, not agent memory, and is not scored here.

### C7 Third-party extensions — 0.20 (high)

By default the agent calls a hosted model and loads no third-party code. Local Hugging Face models are downloaded unpinned on first use, and trust_remote_code defaults to off in the API. However, the Moondream3 loop hard-codes trust_remote_code=True for an unpinned repository, and the shipped CLI turns it on for every model, so selecting those paths runs remote Python inside the agent's own process with all its keys.

- **S L1:** User-chosen model repositories, no revision pinning or hash checks. — [libs/python/agent/cua_agent/agent.py:271](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L271); [libs/python/agent/cua_agent/loops/moondream3.py:47-49](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/moondream3.py#L47-L49) (verified)
  - *To reach the next level:* Model revisions should be pinned.
- **C L1:** The trust_remote_code flag gates most local adapters but not the Moondream3 loop. — [libs/python/agent/cua_agent/loops/moondream3.py:47-49](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/moondream3.py#L47-L49); [libs/python/agent/cua_agent/adapters/huggingfacelocal_adapter.py:26](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/adapters/huggingfacelocal_adapter.py#L26) (verified)
  - *To reach the next level:* Every local-model path should honour the same consent flag and pinning.
- **D L1:** API default is off, but the official CLI forces it on for all models and Moondream3 ignores it (D lowered one level). — [libs/python/agent/cua_agent/agent.py:271](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L271); [libs/python/agent/cua_agent/cli.py:398](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/cli.py#L398); [libs/python/agent/cua_agent/loops/moondream3.py:47-49](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/moondream3.py#L47-L49) (verified)
  - *To reach the next level:* Remote code should never be enabled without an explicit, specific consent.
- **B L0:** Remote model code runs in-process with the model API key, Cua API key and host filesystem. — [libs/python/agent/cua_agent/loops/moondream3.py:47-49](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/moondream3.py#L47-L49); [libs/python/agent/cua_agent/agent.py:919](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L919) (verified)
  - *To reach the next level:* Run local models in a separate process with a scrubbed environment.
- **Cap:** C7-RCELOAD — The Moondream3 loop and the shipped CLI execute unpinned remote model code (trust_remote_code=True) without a specific consent step.

### C8 Secrets & sensitive-data protection — 0.25 (high)

API keys come from environment variables or constructor arguments and are never placed in the model's context or the sandbox. Product telemetry (PostHog and OpenTelemetry to Cua) is on by default but sends only counts, model names and action types, not prompts or screenshots. There is no secret redaction (trajectory saving also does not keep credentials out), and the PII anonymization callback is an unimplemented stub.

- **S L1:** Secrets from env/args; the only sanitizer strips image data, not keys. — [libs/python/agent/cua_agent/agent.py:919](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L919); [libs/python/agent/cua_agent/agent.py:927-932](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L927-L932); [libs/python/agent/cua_agent/callbacks/pii_anonymization.py:92-95](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/pii_anonymization.py#L92-L95) (verified)
  - *To reach the next level:* No type-level masking or log/trajectory filters for credentials.
- **C L1:** Telemetry is content-free by construction; trajectories, debug logs and error paths are unredacted. — [libs/python/agent/cua_agent/callbacks/telemetry.py:65-66](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/telemetry.py#L65-L66) (verified)
  - *To reach the next level:* Logs and transcripts must be redacted too.
- **D L1:** Telemetry is on by default (content-free); verbose logging and trajectories are easy to enable and unredacted. — [libs/python/agent/cua_agent/agent.py:270](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L270); [libs/python/core/cua_core/telemetry/_config.py:14-16](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/core/cua_core/telemetry/_config.py#L14-L16); [libs/python/agent/cua_agent/agent.py:386-396](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L386-L396) (verified)
  - *To reach the next level:* Telemetry should be opt-in.
- **B L1:** Long-lived provider and Cua API keys held in the host process; not reachable from the sandbox. — [libs/python/agent/cua_agent/agent.py:919](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L919); [libs/cua/crates/cua-vmm/src/container/mod.rs:1592-1593](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/cua/crates/cua-vmm/src/container/mod.rs#L1592-L1593) (verified)
  - *To reach the next level:* Keys are not scoped or short-lived.
- **Cap:** none
- **Notes:** PIIAnonymizationCallback returns messages unchanged (TODO) despite its docstring.

### C9 Audit & traceability — 0.40 (high)

An optional trajectory saver writes a structured, per-turn record of every model call, computer action, function call and screenshot to disk, flushed as each action completes. It is off unless the developer passes trajectory_dir, has no actor attribution or approval records (there are no approvals), and is written to a local directory the host process can alter. The telemetry that is on by default is product analytics, not an audit trail.

- **S L2:** Structured JSON per computer/function call with timestamps and screenshots. — [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:468](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L468); [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:522](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L522); [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:254-273](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L254-L273) (verified)
  - *To reach the next level:* No actor attribution (principal, approver) or correlation IDs.
- **C L2:** Covers every computer and function call routed through the agent loop. — [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:468](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L468); [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:522](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L522) (verified)
  - *To reach the next level:* No approvals or denials to record; config changes not logged.
- **D L0:** Opt-in: trajectory_dir defaults to None and logging only starts when verbosity is set. — [libs/python/agent/cua_agent/agent.py:265](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L265); [libs/python/agent/cua_agent/agent.py:337-342](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L337-L342) (verified)
  - *To reach the next level:* Recording should be on by default.
- **B L2:** Each artifact is written synchronously as the event happens. — [libs/python/agent/cua_agent/callbacks/trajectory_saver.py:254-273](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/trajectory_saver.py#L254-L273) (verified)
  - *To reach the next level:* No fail-closed behaviour; write errors are not surfaced as action blockers.
- **Cap:** G1 — The trajectory record exists only when trajectory_dir is passed.

### C10 Limits & kill switch — 0.15 (high)

The agent loop runs until the model stops producing actions; there is no step cap, no wall-clock limit and, by default, no cost cap. An optional budget callback stops the loop once accumulated provider cost exceeds a set amount, and each model request times out after 120 seconds. There is no cancel primitive beyond abandoning the async generator, so a runaway agent loops and spends indefinitely in the default configuration.

- **S L1:** Only an opt-in cost cap checked between steps, plus a per-request model timeout. — [libs/python/agent/cua_agent/callbacks/budget_manager.py:48](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/budget_manager.py#L48); [libs/python/agent/cua_agent/loops/anthropic.py:1805](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/loops/anthropic.py#L1805); searched `rg -n -i 'max_steps|max_iterations|max_turns|step_limit'` in `libs/python/agent/cua_agent/agent.py libs/python/agent/cua_agent/callbacks` → 0 hits (No step or iteration cap in the loop or any callback.) (verified)
  - *To reach the next level:* No iteration cap or session wall-clock limit.
- **C L1:** The budget check applies to the top-level loop only. — [libs/python/agent/cua_agent/agent.py:938](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L938); [libs/python/agent/cua_agent/callbacks/budget_manager.py:48](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/callbacks/budget_manager.py#L48) (verified)
  - *To reach the next level:* No tool/action timeouts counted against the budget.
- **D L0:** Unlimited by default: max_trajectory_budget is None and there is no step cap. — [libs/python/agent/cua_agent/agent.py:269](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L269); [libs/python/agent/cua_agent/agent.py:344-349](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L344-L349); [libs/python/agent/cua_agent/agent.py:936](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L936) (verified)
  - *To reach the next level:* Sensible default step and cost limits are needed.
- **B L0:** No ceiling by default: the loop continues while the model keeps acting. — [libs/python/agent/cua_agent/agent.py:936](https://github.com/trycua/cua/blob/2bce4442c107fc34c45b374b29a35d09f630fd83/libs/python/agent/cua_agent/agent.py#L936); searched `rg -n -i 'max_steps|max_iterations|max_turns|step_limit'` in `libs/python/agent/cua_agent/agent.py libs/python/agent/cua_agent/callbacks` → 0 hits (No step or iteration cap in the loop or any callback.) (verified)
  - *To reach the next level:* Default ceilings on steps, time and spend are needed.
- **Cap:** G1 — The only spend limit (BudgetManagerCallback) is added only when max_trajectory_budget is set.

## Rule-of-Two check
[A] untrusted input: screen content and web pages in the sandbox (agent.py:816-823, browser_tool.py:373) · [B] sensitive data/systems: the user's task data and anything they sign into in the guest; the framework does not constrain it · [C] state change / egress: arbitrary URL visits and typed form/message submissions (browser_tool.py:373, agent.py:758-763) · Same default session? Yes

## Highest-impact improvements
1. Add an approval callback consulted before every computer and function call, and surface pending_safety_checks instead of auto-acknowledging them. — C2 S L0→L2, +0.150 before caps (Playbook 5, step 1)
2. Ship default step and wall-clock limits and a default cost cap in ComputerAgent. — C10 D L0→L2, +0.100 before caps (Playbook 3, step 3)
3. Turn trajectory recording on by default to a directory outside the sandbox. — C9 D L0→L2, +0.100 before caps
4. Fail closed (or require an explicit flag) when gVisor is unavailable instead of falling back to runc. — C4 D L2→L3, +0.050 before caps (Playbook 3, step 1)
5. Redact credentials before writing trajectories and debug logs. — C8 S L1→L2, +0.075 before caps (Playbook 4)

## Re-audit log
- No changes.

## Limitations
- Static source review of commit 2bce4442c107fc34c45b374b29a35d09f630fd83 only; nothing was executed, installed, or probed.
- Framework scored by its defaults: absent primitives score L0 even where a developer could add them via callbacks.
- Scope is the Python agent SDK (libs/python/agent) plus the cua-sandbox / Rust container runtime it drives by default; Cua Spaces, cua-driver (host desktop MCP server), Lume, cua-bench, CUA-S1 and the cua-agents crate were not scored separately.
- Cloud (Fleet) sandboxes, QEMU/Lume VMs and cua-spacesd internals were only checked where they affect the default local container path.
- Model behaviour (refusals, deception) is out of scope; only code-level controls are scored.
- No reviewer-directed prompt injection found in the repo.
