# Defense-in-Depth Score: PentAGI

**Repo:** https://github.com/vxcontrol/pentagi · **Commit:** `55a063ecb307e8786cb8b068655e1d80e96cad7b` · **Reviewed:** 2026-10-03
**What it is:** Fully autonomous multi-agent system for complex penetration testing tasks
**Category:** Cybersecurity
**Scored configuration:** Default docker-compose stack (docker-compose.yml with .env.example values): API bound to 127.0.0.1, DOCKER_INSIDE=false, ASK_USER=false, agents run shell in a per-flow Docker worker container.
**Agent surface (default):** code execution yes · filesystem write yes · network egress yes · external credentials no · persistent memory yes · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 3.0 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L2 | L2 | L2 | L2 | 0.50 | — | **0.50** | High |
| C2 | Approval gates | L0 | L0 | L0 | L1 | 0.05 | C2-POWERBYPASS | **0.05** | High |
| C3 | Tool & action scoping | L0 | L1 | L1 | L1 | 0.17 | — | **0.17** | High |
| C4 | Code-execution isolation | L2 | L3 | L3 | L2 | 0.62 | — | **0.62** | High |
| C5 | Untrusted input blast radius | L0 | L0 | L0 | L0 | 0.00 | C5-WORSTCASE | **0.00** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | L0 | L0 | 0.15 | — | **0.15** | High |
| C7 | Third-party extensions | L0 | L0 | L0 | L2 | 0.10 | C7-RCELOAD | **0.10** | High |
| C8 | Secrets & sensitive-data protection | L1 | L1 | L1 | L1 | 0.25 | — | **0.25** | High |
| C9 | Audit & traceability | L2 | L2 | L3 | L4 | 0.65 | — | **0.65** | High |
| C10 | Limits & kill switch | L2 | L2 | L2 | L1 | 0.45 | — | **0.45** | High |


PentAGI runs LLM-driven agents that execute arbitrary shell commands, with no human approval step, inside per-flow Docker containers that have unrestricted network access. The container boundary is the main protection: workers get no secrets and no Docker socket by default, every tool call is logged before it runs, and commands have timeouts and iteration caps. Nothing limits what injected web or target content can make the agent do, the model picks the container image, and answer/guide/code memory is shared across all users. Raising the score would start with an approval or target-scope gate, non-root hardened workers with egress limits, and per-user memory isolation.

## Critical gaps
- No approval gate exists for any tool call, including arbitrary shell execution; prompts state actions need no confirmation. (ASI09, ASI02, T10; C2) — [backend/pkg/tools/terminal.go:196-205](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L196-L205); [backend/pkg/templates/prompts/primary_agent.tmpl:9](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/templates/prompts/primary_agent.tmpl#L9)
- Untrusted web, scraper and target content flows into an agent that has an ungated shell with unrestricted egress and no injection controls. (ASI01, LLM01, T6; C5) — [backend/pkg/tools/terminal.go:196-205](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L196-L205); [backend/pkg/templates/prompts/primary_agent.tmpl:9](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/templates/prompts/primary_agent.tmpl#L9)
- The model chooses the sandbox image from any registry by default (empty allowlist), unpinned and unverified, and runs it without consent. (ASI04, T17, LLM03; C7) — [backend/pkg/config/config.go:73](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L73); [backend/pkg/providers/image_selection.go:64-66](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/image_selection.go#L64-L66)

## Criterion details

### C1 Identity & least privilege — 0.50 (high)

The model's tools run only inside a per-flow Docker container that is created with no environment variables, no cloud credentials and no Docker socket in the default configuration, so a hijacked agent does not hold API keys. Inside that container the agent is root with a broad capability set and unrestricted network egress. Operators are scoped by role privileges and every flow query is filtered by the owning user, but a seeded admin account ships with the documented password 'admin' (forced change on first login), and the shared vector store lets one user's agent read knowledge derived from other users' engagements.

- **S L2:** Dedicated sandbox identity with no injected credentials and role-scoped privileges for API users, but the agent is root in its container and read/write share one identity. — [backend/pkg/docker/worker_spec.go:20-27](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/worker_spec.go#L20-L27); searched `rg -n 'User:' --glob '!*_test.go'` in `backend/pkg/docker/worker_spec.go backend/pkg/docker/client.go` → 0 hits (no container User is ever set, so the image default (root in the shipped kali image) applies; 0 hits); [backend/pkg/server/auth/permissions.go:20-22](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/server/auth/permissions.go#L20-L22) (verified)
  - *To reach the next level:* L3 needs per-capability narrowing (non-root user, per-tool scoping); the worker runs as the image's default root user.
- **C L2:** All built-in tools of every agent execute through the same flow-scoped executor and container; API access is checked per user, but the knowledge stores are shared across users. — [backend/sqlc/models/flows.sql:27](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/sqlc/models/flows.sql#L27); [backend/pkg/tools/search.go:94-96](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/search.go#L94-L96); [backend/pkg/database/knowledge/vectorstore/vectorstore.go:12](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/database/knowledge/vectorstore/vectorstore.go#L12) (verified)
  - *To reach the next level:* L3/L4 need authorization evaluated against the requesting principal on every retrieval path; answer/guide/code retrieval has no user filter.
- **D L2:** Default install gives the agent no Docker access and no secrets; widening (DOCKER_INSIDE, host network, NET_ADMIN) is an operator env change, and the shipped admin account has a known password until changed. — [backend/pkg/config/config.go:36](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L36); [README.md:863](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/README.md#L863); [backend/migrations/sql/20241026_115120_initial_state.sql:87](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/migrations/sql/20241026_115120_initial_state.sql#L87) (verified)
  - *To reach the next level:* L3 needs a near-minimal default (non-root, no root-equivalent seeded login) with explicit elevation.
- **B L2:** If the sandbox identity or API authorization is hijacked the attacker gets shell and network reach from one container plus the ability to run flows, but no stored external credentials; the orchestrator's Docker socket and LLM keys are not exposed to the model. — [docker-compose.yml:230](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/docker-compose.yml#L230); [backend/pkg/docker/client.go:364](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L364) (verified)
  - *To reach the next level:* L3 needs writes scoped to one project with non-destructive limits; unrestricted egress means attacks against any reachable system.
- **Cap:** none

### C2 Approval gates — 0.05 (high)

There is no human approval step before any tool call. The terminal tool executes whatever shell string the model produces, and the system prompts tell the agents that every action is pre-authorized and needs no confirmation. The only interactive hook is an optional 'ask' tool (off by default) that the model itself decides to call. Attacks on target systems cannot be undone and nothing rate-limits them.

- **S L0:** No approval mechanism exists; the optional ask tool is model-initiated and prompts instruct agents that confirmation is not required. — searched `rg -n -i --glob '!*_test.go' --glob '!*.yml' 'requires_approval|approval|human_in_the_loop'` in `backend/pkg/tools backend/pkg/providers backend/pkg/controller` → 0 hits (searched Go/template sources (yml model catalogs excluded) for any approval concept; 0 hits); [backend/pkg/templates/prompts/primary_agent.tmpl:9](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/templates/prompts/primary_agent.tmpl#L9); [backend/pkg/tools/tools.go:1061-1064](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/tools.go#L1061-L1064) (verified)
  - *To reach the next level:* L1 needs at least one blanket human approval step before commands run.
- **C L0:** The most powerful tool (terminal, arbitrary shell in the sandbox) is invoked directly with no gate. — [backend/pkg/tools/terminal.go:196-205](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L196-L205); [backend/pkg/tools/executor.go:291](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L291) (verified)
  - *To reach the next level:* L1 needs a gate on flagged tools.
- **D L0:** There is no approval to be on by default; ASK_USER defaults to false and is not an approval gate. — [backend/pkg/config/config.go:25](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L25) (verified)
  - *To reach the next level:* L1 needs a default-on approval.
- **B L1:** Sandbox-local changes are contained and reversible by deleting the container, but actions against external targets (exploitation, scans, file changes on targets) are irreversible and unlimited; no spend or rate limits. — [backend/pkg/docker/client.go:376-379](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L376-L379); [backend/pkg/tools/terminal.go:32](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L32) (verified)
  - *To reach the next level:* L2 needs the common case reversible and approvals/limits bounding irreversible ones.
- **Cap:** C2-POWERBYPASS — The shell tool, the most powerful action path, runs with no approval gate in the default configuration.

### C3 Tool & action scoping — 0.17 (high)

The terminal tool takes an arbitrary shell string and the browser tool takes any URL; the only validation is that file paths must not contain shell-expansion tokens. There is no target scope list or command filtering in code (scope is only described in prompts). Each agent receives a role-specific tool list and the model cannot add tools, but every working agent gets the shell by default and there is no operator switch to remove it. Commands are confined to the container, not to a target set.

- **S L0:** Raw passthrough of an arbitrary shell string to the sandbox and arbitrary URLs to the scraper; file-path check only rejects shell-expansion text. — [backend/pkg/tools/terminal.go:204](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L204); [backend/pkg/tools/terminal.go:381-383](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L381-L383); searched `rg -n -i --glob '!*_test.go' --glob '!testdata' 'allowed.?targets|target.?scope|denylist|blocklist'` in `backend/pkg/tools backend/pkg/providers backend/pkg/docker backend/pkg/controller` → 1 hits (1 hit, a test-expectation string in providers/tester, not a control) (verified)
  - *To reach the next level:* L1 needs denylist/regex filtering on command content or targets.
- **C L1:** A few paths validate: JSON schema parsing, file-path expansion check, and an optional image allowlist; the shell and browser tools have none. — [backend/pkg/tools/terminal.go:383-390](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L383-L390); [backend/pkg/providers/image_selection.go:64-66](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/image_selection.go#L64-L66) (verified)
  - *To reach the next level:* L2 needs most built-in tools to validate arguments.
- **D L1:** Each agent gets a role-specific tool list (searcher and memorist have no shell), but terminal and file tools are on for all executing agents with no operator switch to disable them. — [backend/pkg/tools/tools.go:856](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/tools.go#L856); searched `rg -n -i --glob '!*_test.go' --glob '!testdata' 'terminal.*(enabled|disable)'` in `backend/pkg/config/config.go backend/pkg/tools` → 0 hits (no setting disables the terminal tool; 0 hits) (verified)
  - *To reach the next level:* L2 needs selectable tool groups with the default still including exec; L3 read-only default.
- **B L1:** A misused shell can run any command against any reachable host, limited only by the container boundary and a 2048-process cap. — [backend/pkg/docker/client.go:376-379](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L376-L379); [backend/pkg/docker/client.go:395](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L395) (verified)
  - *To reach the next level:* L2 needs reach scoped to a workspace/target set with full write only inside it.
- **Cap:** none

### C4 Code-execution isolation — 0.62 (high)

Every model-reachable command, file operation and delegated agent action runs by Docker exec inside a per-flow container; no agent path uses host process execution, and if a container cannot be created the flow fails or falls back to the default image, never to the host. The container drops all capabilities and re-adds a 14-item list (Docker's default set minus MKNOD, plus SYS_PTRACE), has a 2048-process limit and mounts only its own work directory with no secrets in its environment. It is not hardened beyond that: root user, writable root filesystem, no no-new-privileges (deliberately omitted so privilege-escalation testing works), no memory or CPU limits, unrestricted egress, and ports published on all interfaces. Operators can opt in to Docker access inside the sandbox; a raw host-socket mount is documented as host-root equivalent and a TLS dind option with a startup isolation probe exists but both are off by default.

- **S L2:** Stock-style container with CapDrop ALL plus a 14-capability allow-list, default seccomp, pids limit; root inside, writable rootfs, no no-new-privileges. — [backend/pkg/docker/worker_spec.go:22-26](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/worker_spec.go#L22-L26); [backend/pkg/docker/worker_spec.go:34-37](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/worker_spec.go#L34-L37); [backend/pkg/docker/client.go:368-370](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L368-L370); searched `rg -n --glob '!*_test.go' --glob '!testdata' --glob '!docs' --glob '!*.sh' --glob '!*.json' 'ReadonlyRootfs|SecurityOpt|Privileged|NanoCPUs|CPUShares|Ulimits'` in `backend/pkg` → 0 hits (no read-only rootfs, security-opt, CPU or ulimit settings are applied to workers; 0 hits) (verified)
  - *To reach the next level:* L3 needs non-root, no-new-privileges and a read-only root filesystem or a separate runtime.
- **C L3:** All terminal, file and delegated-agent execution goes through Docker exec in the flow container; no host exec path exists in agent code and failures fall back only to the default container image. — searched `rg -n --glob '!*_test.go' 'exec\.Command'` in `backend/pkg/tools backend/pkg/providers backend/pkg/docker backend/pkg/controller` → 0 hits (no host process execution in agent paths; 0 hits (os/exec appears only in installer and system packages)); [backend/pkg/docker/client.go:297-300](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L297-L300); [backend/pkg/tools/terminal.go:204-206](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L204-L206) (verified)
  - *To reach the next level:* L4 needs the C and D conditions for spawned tools/extensions and a documented fail-closed guarantee; limited by S+1.
- **D L3:** Containers are always used, the model cannot request unsandboxed execution or change container policy, and weakening needs explicit named operator settings (DOCKER_INSIDE, DOCKER_NETWORK=host, DOCKER_NET_ADMIN). — [backend/pkg/config/config.go:36-37](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L36-L37); [backend/pkg/docker/sandbox.go:242-243](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/sandbox.go#L242-L243); [.env.example:345](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/.env.example#L345) (verified)
  - *To reach the next level:* L4 needs the policy defined outside anything writable and elevated modes time-bounded; .env.example ships DOCKER_NET_ADMIN=true.
- **B L2:** Worker mounts only its flow work directory (read-write) and has an empty environment, but egress is unrestricted, it is not ephemeral per call, has no memory/CPU limits, runs as root, and ports are published on all interfaces by default. — [backend/pkg/docker/client.go:364](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L364); [backend/pkg/docker/client.go:419-422](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L419-L422); [backend/pkg/config/config.go:63](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L63) (verified)
  - *To reach the next level:* L3 needs network egress off or allowlisted and CPU/memory limits.
- **Cap:** none

### C5 Untrusted input blast radius — 0.00 (high)

The agents read web search results, scraped pages, target responses and uploaded files, and all of it enters the model context with the same standing as instructions; the prompts contain no untrusted-content rules and no code distinguishes sources. A hijacked agent has an unrestricted shell with unrestricted egress and can attack other systems or move engagement data out unattended, with no human step. No secrets are placed in the sandbox, but findings, uploaded scope documents and discovered credentials are.

- **S L0:** No structural limit: no taint tracking, quarantine, approval after untrusted reads, or detection; prompts do not even instruct the model to treat fetched content as data. — searched `rg -n -i 'untrusted|prompt injection|sanitiz'` in `backend/pkg/templates/prompts` → 0 hits (searched all prompt templates for untrusted-content handling; hit count recorded); [backend/pkg/tools/executor.go:602-609](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L602-L609) (verified)
  - *To reach the next level:* L1 needs detection or delimiting of untrusted content.
- **C L0:** Search, browser and terminal results (including target responses) are not distinguished from user instructions. — [backend/pkg/tools/browser.go:302-330](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/browser.go#L302-L330); [backend/pkg/tools/executor.go:341-345](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L341-L345) (verified)
  - *To reach the next level:* L1 needs one untrusted source handled.
- **D L0:** Nothing to be on by default. — [backend/pkg/config/config.go:25](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L25) (verified)
  - *To reach the next level:* L1 needs a control that is on but defeatable.
- **B L0:** Shell with unrestricted egress and irreversible actions against targets run unattended while engagement data is in reach. — [backend/pkg/tools/terminal.go:196-205](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L196-L205); [backend/pkg/docker/client.go:395](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L395) (verified)
  - *To reach the next level:* L1 needs only one of exfiltration or irreversible actions unattended.
- **Cap:** C5-WORSTCASE — B is L0: a hijacked agent can exfiltrate engagement data and take irreversible actions against targets with no human involved.

### C6 Memory, context & configuration integrity — 0.15 (high)

Tool results (terminal output, search results, file reads) and model-authored answers, guides and code snippets are written to a pgvector store automatically, with secret-pattern redaction but no validation, approval or expiry. Only the 'memory' type is filtered by flow; the answer, guide and code types live in one collection with no user or flow filter and are retrieved by later agents in other flows and for other users. Poisoned content from a web page or target can therefore persist and steer later engagements. Workspace files do not configure the agent.

- **S L1:** Writes are logged to the vector-store log and secrets are masked, but content is stored unvalidated and unapproved. — [backend/pkg/tools/executor.go:600-610](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L600-L610); [backend/pkg/tools/search.go:230](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/search.go#L230) (verified)
  - *To reach the next level:* L2 needs provenance on entries and presenting them as data.
- **C L1:** Only the flow-scoped memory retrieval is controlled; answer, guide and code stores are not. — [backend/pkg/tools/memory.go:74-78](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/memory.go#L74-L78); [backend/pkg/tools/guide.go:95-98](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/guide.go#L95-L98) (verified)
  - *To reach the next level:* L2 needs the main memory store controlled (done) and other stores/auto-loaded content controlled.
- **D L0:** Answer, guide and code knowledge is shared across all users and flows by default (single collection, no user/flow filter). — [backend/pkg/database/knowledge/vectorstore/vectorstore.go:12](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/database/knowledge/vectorstore/vectorstore.go#L12); [backend/pkg/tools/search.go:93-96](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/search.go#L93-L96); [backend/pkg/tools/code.go:98-101](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/code.go#L98-L101) (verified)
  - *To reach the next level:* L1 needs per-user isolation enforced even by a filter.
- **B L0:** Poisoned entries persist across sessions and users and are retrieved by agents that hold the shell. — [backend/pkg/tools/search.go:93-96](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/search.go#L93-L96); [backend/pkg/tools/tools.go:843-856](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/tools.go#L843-L856) (verified)
  - *To reach the next level:* L1 needs persistence limited to one user's sessions.
- **Cap:** none

### C7 Third-party extensions — 0.10 (high)

There is no plugin or MCP loader, but by default the model chooses which Docker image the sandbox runs (any image reference, because the allowlist is empty), and the installer agent installs packages on the model's request. Defaults are unpinned (:latest and untagged pentest images) with no digest or signature check. All of it runs inside the sandbox container with an empty environment, which is what limits the damage.

- **S L0:** Model-selected images are pulled and run automatically; default images are unpinned and unverified. — [backend/pkg/config/config.go:73](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L73); [backend/pkg/providers/image_selection.go:64-66](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/image_selection.go#L64-L66); searched `rg -n -i --glob '!*_test.go' 'cosign|digest|sha256:|signature'` in `backend/pkg/docker backend/pkg/providers/image_selection.go` → 0 hits (no integrity checking of images; 0 hits) (verified)
  - *To reach the next level:* L1 needs user-chosen but unpinned sources instead of model-chosen ones.
- **C L0:** No extension or image type is verified. — [backend/pkg/config/config.go:66-67](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L66-L67) (verified)
  - *To reach the next level:* L1 needs one type verified.
- **D L0:** Images the model names are installed automatically with no consent, and the empty allowlist means any image. — [.env.example:377-379](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/.env.example#L377-L379) (verified)
  - *To reach the next level:* L1 needs a generic consent flag or prompt.
- **B L2:** Third-party images and packages run in the sandbox container with an empty environment, not in the orchestrator. — [backend/pkg/docker/worker_spec.go:20-27](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/worker_spec.go#L20-L27); [backend/pkg/docker/client.go:364](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/client.go#L364) (verified)
  - *To reach the next level:* L3 needs per-extension sandboxing with scoped credentials and no shared data.
- **Cap:** C7-RCELOAD — By default the model picks and runs arbitrary remote container images without consent.

### C8 Secrets & sensitive-data protection — 0.25 (high)

Provider keys come from environment variables and are held by the orchestrator only; the sandbox environment is empty. A secret-pattern anonymizer redacts text before it is stored in the vector store or sent to external search engines, but terminal output goes to the model and the tool-call log unredacted and there is no log redaction. The default compose and env files are not locked down. A content-free update report is sent on a 3 hour timer by default; Langfuse and OpenTelemetry export are opt-in.

- **S L1:** Secrets are env vars with pattern-based redaction on memory and search paths only; shipped defaults are not locked down. — [backend/pkg/tools/tools.go:376-381](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/tools.go#L376-L381) (verified)
  - *To reach the next level:* L2 needs type-level masking and log filters on main paths.
- **C L1:** Redaction covers vector-store writes and external search queries only, not logs, tool-call records or model-bound terminal output. — [backend/pkg/tools/executor.go:609](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L609); [backend/pkg/tools/executor.go:330](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L330) (verified)
  - *To reach the next level:* L2 needs logs and transcripts covered.
- **D L1:** Content-free update/usage telemetry is on by default every 3 hours; verbose logging is off and Langfuse/OTEL are opt-in. — [backend/pkg/config/config.go:279](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L279); [backend/pkg/database/summary.go:31-33](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/database/summary.go#L31-L33) (verified)
  - *To reach the next level:* L2 needs telemetry opt-in.
- **B L1:** Long-lived provider keys and database credentials in a plaintext env file; not reachable from the model or sandbox environment. — [docker-compose.yml:282](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/docker-compose.yml#L282); [backend/pkg/docker/worker_spec.go:20-27](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/worker_spec.go#L20-L27) (verified)
  - *To reach the next level:* L2 needs scoped keys.
- **Cap:** none

### C9 Audit & traceability — 0.65 (high)

Every tool call from every agent is written to the database (name, arguments, result, status, duration, flow, task and subtask IDs) before the tool runs, and a failure to write stops the call. Terminal commands and output, messages and delegations between agents are also stored, and the log is written by the orchestrator, outside the sandbox. Records identify the flow and its owning user and the delegating agent role in separate tables, but the tool-call row itself carries no agent or user field and there are no approvals to record. Export to Langfuse or OpenTelemetry exists but is opt-in.

- **S L2:** Structured per-call record with arguments, result, status and timing in the toolcalls table. — [backend/migrations/sql/20241026_115120_initial_state.sql:195-207](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/migrations/sql/20241026_115120_initial_state.sql#L195-L207); [backend/pkg/tools/executor.go:330](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L330) (verified)
  - *To reach the next level:* L3 needs actor attribution (agent, requesting user) on each record and delegation chain correlation.
- **C L2:** All built-in tool calls for all agent executors are logged; delegations are logged in agentlogs; no approvals/denials or configuration changes exist to log. — [backend/migrations/sql/20241130_183411_new_type_logs.sql:15-24](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/migrations/sql/20241130_183411_new_type_logs.sql#L15-L24); [backend/pkg/tools/terminal.go:221](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L221) (verified)
  - *To reach the next level:* L3 needs approvals and denials recorded.
- **D L3:** On by default, written by the orchestrator into Postgres, which the sandbox does not reach on the default network; no switch to disable it. — [backend/pkg/controller/tclog.go:58](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/controller/tclog.go#L58); [docker-compose.yml:167](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/docker-compose.yml#L167) (verified)
  - *To reach the next level:* L4 needs tamper-evident or off-host storage.
- **B L4:** The tool-call record is created before execution and an error aborts the call; terminal stdin is logged before the exec is created. — [backend/pkg/tools/executor.go:330-333](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/executor.go#L330-L333); [backend/pkg/tools/terminal.go:219-224](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L219-L224) (verified)
- **Cap:** none

### C10 Limits & kill switch — 0.45 (high)

Each agent chain stops after 100 (main agents) or 20 (helper agents) model turns, repeated identical tool calls are cut off, and each shell command has a timeout (20 minutes by default, 3 hours maximum). Stopping a flow kills the commands it started inside the sandbox. There is no token, cost or wall-clock limit for a flow, and delegated agents start their own iteration budgets, so total work is bounded only by the tool graph and the operator's provider spend limits.

- **S L2:** Iteration cap plus enforced per-execution timeout; stop interrupts in-flight commands, but no token/cost/wall-clock cap. — [backend/pkg/providers/performer.go:37-38](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/performer.go#L37-L38); [backend/pkg/providers/performer.go:117-120](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/performer.go#L117-L120); [backend/pkg/tools/terminal.go:92-100](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/tools/terminal.go#L92-L100); searched `rg -n -i --glob '!*_test.go' --glob '!testdata' 'max.?cost|token.?budget|spend.?limit|cost.?limit|max.?tokens.?per'` in `backend/pkg/providers backend/pkg/controller backend/pkg/config` → 0 hits (no cost or token budget; hit count recorded) (verified)
  - *To reach the next level:* L3 needs step, wall-clock and cost caps all enforced.
- **C L2:** Top-level loops and tool timeouts are bounded; each sub-agent chain gets a fresh iteration budget. — [backend/pkg/providers/performers.go:76](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/performers.go#L76); [backend/pkg/providers/performer.go:99-114](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/providers/performer.go#L99-L114) (verified)
  - *To reach the next level:* L3 needs sub-agents to share one budget.
- **D L2:** Sensible defaults (100/20 calls, 1200 s) configurable by env; the model cannot raise them but delegation resets the budget. — [backend/pkg/config/config.go:316-317](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L316-L317); [backend/pkg/config/config.go:76](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/config/config.go#L76) (verified)
  - *To reach the next level:* L3 needs limits the model cannot reset by delegating.
- **B L1:** Ceilings are large (up to 100 calls of up to 3 hours each per chain, no spend ceiling); stop kills flow commands but assistant commands are launched as non-flow commands that stop leaves running. — [backend/pkg/docker/flow_commands.go:14-25](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/docker/flow_commands.go#L14-L25); [backend/pkg/controller/flow.go:912](https://github.com/vxcontrol/pentagi/blob/55a063ecb307e8786cb8b068655e1d80e96cad7b/backend/pkg/controller/flow.go#L912) (verified)
  - *To reach the next level:* L2 needs moderate ceilings.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: web search/scraper results and target responses enter context (backend/pkg/tools/browser.go:302, executor.go:602) · [B] sensitive data/systems: engagement data in /work, uploaded scope files, discovered credentials (backend/pkg/docker/client.go:364) · [C] state change / egress: ungated shell with unrestricted egress (backend/pkg/tools/terminal.go:196) · Same default session? Yes

## Highest-impact improvements
1. Add a per-command human approval or target-scope allowlist gate before terminal execution. — C2 S L0→L3, +0.225 before caps (Playbook 5)
2. Run workers as non-root with no-new-privileges, read-only rootfs, memory/CPU limits and egress allowlist. — C4 S L2→L3, +0.075 before caps (Playbook 3 step 1)
3. Scope answer/guide/code vector retrieval by user and flow and require review before cross-flow reuse. — C6 D L0→L2, +0.100 before caps (Playbook 2)
4. Default DOCKER_IMAGE_SELECTION_MODE to fixed and pin images by digest. — C7 S L0→L3, +0.225 before caps (Playbook 3)
5. Add a flow-wide token/cost and wall-clock budget shared by sub-agents. — C10 S L2→L3, +0.075 before caps (Playbook 3 step 3)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the default compose configuration; the opt-in DOCKER_INSIDE modes (raw host socket or TLS dind with startup probe), DOCKER_NETWORK=host, Graphiti, Langfuse and the optional installer were reviewed only at the configuration level, not scored as alternatives.
- The frontend rendering of model output, GraphQL/REST handlers beyond authentication and flow ownership, the LLM provider adapters and the installer wizard were not examined in depth.
- Whether the sandbox can reach the database depends on DOCKER_NETWORK, which the worker-node guide sets to pentagi-network; the default (empty) was scored.
- No reviewer-directed instructions were found in the repository text; repo CLAUDE.md is contributor guidance.
