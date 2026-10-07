# Defense-in-depth score: n8n-MCP

**Repo:** https://github.com/czlonkowski/n8n-mcp · **Commit:** `3a2ec761e006089638d281a27d15ba030ec3ec88` (v2.91.0) · **Reviewed:** 2026-10-05
**What it is:** MCP server that gives AI assistants n8n node documentation, templates and validation, and, with an n8n API key, tools to create, edit, run and delete workflows and other resources on an n8n instance.
**Category:** Agent Frameworks
**Scored configuration:** Local stdio server launched with `npx n8n-mcp` and the README's full configuration (N8N_API_URL and N8N_API_KEY set), no DISABLED_TOOLS, default telemetry and default (strict) URL security mode.
**Agent surface (default):** code execution yes · filesystem write no · network egress yes · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents yes · external communication yes

## Score: 3.3 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L2 | L0 | L0 | 0.23 | none | **0.23** | High |
| C2 | Approval gates | L1 | L1 | L2 | L2 | 0.35 | none | **0.35** | High |
| C3 | Tool & action scoping | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | none | **0.05** | Medium |
| C5 | Untrusted input blast radius | L2 | L1 | L2 | L0 | 0.33 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L0 | L0 | L0 | L2 | 0.10 | C6-REPOCONFIG | **0.10** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | none | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L0 | L1 | 0.35 | none | **0.35** | High |
| C9 | Audit & traceability | L1 | L1 | L1 | L0 | 0.20 | none | **0.20** | High |
| C10 | Limits & kill switch | L2 | L2 | L1 | L1 | 0.40 | none | **0.40** | High |

Controls where a risk surface exists: 2.33 / 9.0 (26%); 1 criterion scored SA (surface absent).

With an n8n API key configured, n8n-MCP hands the model every workflow, credential, execution and agent operation that key allows, including building and running workflows that execute code and call any service the instance has credentials for. The server protects its own outbound connections well and backs up workflows before edits, but it has no read-only default, no confirmation step of its own, and returns instance content to the model without marking it as untrusted, so a hijacked session can both read sensitive data and make irreversible changes. Usage telemetry, including sanitized workflow content, is on by default.

## Critical gaps
- A hijacked session can both read instance data and make irreversible changes (create, run or delete workflows) with no human step in the server. (ASI01, LLM01; C5). Evidence: [src/mcp/tools-n8n-manager.ts:476-483](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L476-L483); [src/mcp/tools-n8n-manager.ts:385-386](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L385-L386); [src/mcp/handlers-n8n-manager.ts:1347-1352](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L1347-L1352)
- The single n8n API key gives the model the whole instance and, through workflows, every connected service's stored credentials. (ASI03, T3; C1). Evidence: [src/services/n8n-api-client.ts:219-221](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L219-L221); [docs/SECURITY_HARDENING.md:5](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L5)
- Configuration is auto-loaded from the launch directory, so a repository can change which tools are offered and other security-relevant settings without a trust decision. (ASI06, T1; C6). Evidence: [src/config/n8n-api.ts:21-26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L21-L26)

## Criterion details

### C1 Identity & least privilege: 0.23 (high confidence)

Every management tool uses a single long-lived n8n API key supplied in the environment, so the model acts with whatever the key's owner can do on the instance. The server does not ask for a scoped key, split read and write credentials, or check individual requests against a policy; its only narrowing is an operator list of tools or operations to switch off, which is empty by default. Because workflows can use any credential stored in n8n, a misused session reaches every service the instance is connected to.

- **S L1:** One dedicated n8n API key, chosen by the operator and static for the run, is attached to every request; the server does not request or require a narrower key. Evidence: [src/services/n8n-api-client.ts:219-221](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L219-L221); [docs/SECURITY_HARDENING.md:5](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L5) (verified)
  - *To reach the next level:* No per-tool or per-capability credentials, and no authorization check mapping requests to a least-privilege policy.
- **C L2:** All built-in management tools share the one API client and key; no tool constructs its own privileged client. Evidence: [src/services/n8n-api-client.ts:219-221](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L219-L221); [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795) (verified)
  - *To reach the next level:* No authorization layer in code that every tool path passes through.
- **D L0:** Configuring the API URL and key exposes the full management tool set, including deletes, credential management and workflow execution; narrowing requires the operator to list tools in DISABLED_TOOLS. Evidence: [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795); [src/mcp/tool-policy.ts:36-37](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tool-policy.ts#L36-L37) (verified)
  - *To reach the next level:* No read-only or minimal default; write tools should require explicit operator enabling.
- **B L0:** The key can create and run workflows that use every credential stored in the instance, so a hijacked session reaches the whole n8n instance and the services connected to it. Evidence: [docs/SECURITY_HARDENING.md:5](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L5); [SECURITY.md:22](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/SECURITY.md#L22) (verified)
  - *To reach the next level:* Authority would need to be limited to one system or mostly read operations.
- **Cap:** none
- **Notes:** HTTP multi-tenant mode (opt-in, not scored) takes per-request instance URLs and keys from client headers.

### C2 Approval gates: 0.35 (high confidence)

As a tool server, n8n-MCP leaves approval to the MCP host and gives it risk annotations on every tool. Several tools bundle reads and writes behind one name (for example list, get and delete of executions), and the annotations are not accurate on every mutating tool, so a host cannot always tell a read from a destructive call. There is no read-only mode or server-side confirmation by default; an operator can switch off individual operations, and the server then recomputes the annotations. Workflow edits are backed up locally before they are applied and can be rolled back, but deletes and workflow runs cannot be undone.

- **S L1:** Every tool carries readOnlyHint/destructiveHint annotations, but some tools mix read and write operations and the hints are not accurate on every mutating tool. Evidence: [src/mcp/tools-n8n-manager.ts:238-254](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L238-L254); [src/mcp/tools-n8n-manager.ts:476-483](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L476-L483) (verified)
  - *To reach the next level:* Separate read and write tools with accurate destructive annotations on every tool.
- **C L1:** Hosts that gate on annotations catch the tools flagged destructive, but not every mutating path is flagged. Evidence: [src/mcp/tools-n8n-manager.ts:238-254](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L238-L254); [src/mcp/tools-n8n-manager.ts:1016-1017](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L1016-L1017) (verified)
  - *To reach the next level:* Every mutating path must carry an accurate destructive signal.
- **D L2:** Annotations are static in code and the model cannot change them; operator environment settings that disable operations rewrite them silently. Evidence: [src/mcp/server.ts:708-713](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L708-L713); [src/mcp/tool-policy.ts:36-37](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tool-policy.ts#L36-L37) (verified)
  - *To reach the next level:* No server-enforced confirmation or read-only mode on by default.
- **B L2:** Full and partial workflow updates create a local backup by default and versions can be rolled back, but deletes of workflows, credentials, executions and data rows, and workflow or agent runs, are irreversible. Evidence: [src/mcp/handlers-workflow-diff.ts:245](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-workflow-diff.ts#L245); [src/mcp/handlers-n8n-manager.ts:1347-1352](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L1347-L1352); [src/services/workflow-versioning-service.ts:154-159](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/workflow-versioning-service.ts#L154-L159) (verified)
  - *To reach the next level:* No preview or dry run for most external actions and no undo for deletes.
- **Cap:** none

### C3 Tool & action scoping: 0.40 (high confidence)

Tool arguments are validated with typed schemas, identifiers are encoded before they go into API paths, and the server's own outbound requests pass a careful URL guard that blocks internal and metadata addresses, pins DNS and refuses redirects. But the central tools take whole workflow definitions and agent arguments and pass them to n8n as they are, with no restriction on node types, so the model can build workflows that run code or call any host. All tools are enabled by default; an operator can switch off individual tools or operations.

- **S L2:** Typed schemas, path-segment encoding and a strict SSRF guard with DNS pinning and no redirects protect the server's own requests, but workflow bodies and agent arguments are forwarded without restriction. Evidence: [src/utils/ssrf-protection.ts:308](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/utils/ssrf-protection.ts#L308); [src/services/n8n-api-client.ts:384-393](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L384-L393); [src/services/n8n-api-client.ts:229-231](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L229-L231); [src/mcp/tools-n8n-manager.ts:936](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L936) (verified)
  - *To reach the next level:* No allowlist of node types or bounds on what a created workflow may do.
- **C L2:** Built-in handlers validate inputs through schemas and the shared API client, but the agent tool forwards its arguments unchanged. Evidence: [src/services/n8n-api-client.ts:956-958](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L956-L958); [src/mcp/tools-n8n-manager.ts:936](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L936) (verified)
  - *To reach the next level:* Every tool, including agent arguments, should pass a shared validation layer.
- **D L1:** All documentation and management tools are on by default once the API is configured; individual tools and operations can be disabled. Evidence: [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795); [src/mcp/tool-policy.ts:36-37](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tool-policy.ts#L36-L37) (verified)
  - *To reach the next level:* Tool groups or a read-only default set.
- **B L1:** A misused tool reaches the whole n8n instance, and through workflow nodes any host and connected service, with only the n8n instance's own settings as limits. Evidence: [src/mcp/tools-n8n-manager.ts:13-14](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L13-L14); [src/triggers/handlers/webhook-handler.ts:91-96](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/triggers/handlers/webhook-handler.ts#L91-L96) (verified)
  - *To reach the next level:* Tool reach scoped to a project or bounded quantities.
- **Cap:** none

### C4 Code-execution isolation: 0.05 (medium confidence)

The server never runs model-written code itself, but it lets the model create workflows containing code and command nodes and run them on the n8n instance, and run n8n agents that use real tools. n8n-MCP adds no isolation of its own around this; its maintainers state that n8n is the security boundary and point to n8n's own Code-node sandbox and settings. Whatever code runs has network access and can use the instance's stored credentials.

- **S L0:** No isolation primitive in the server; model-authored workflow code runs wherever n8n runs it. Evidence: [SECURITY.md:22](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/SECURITY.md#L22); [src/mcp/tools-n8n-manager.ts:385-386](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L385-L386) (verified)
  - *To reach the next level:* An isolation boundary owned by the server (for example refusing code nodes unless enabled) does not exist.
- **C L0:** No execution path through the server is contained by any control of its own. Evidence: [src/mcp/tools-n8n-manager.ts:385-386](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L385-L386); [src/mcp/tools-n8n-manager.ts:13-14](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L13-L14) (verified)
  - *To reach the next level:* Every model-reachable execution path would need to pass an isolation control.
- **D L0:** Workflow creation and execution are enabled by default with no isolation setting to turn on. Evidence: [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795) (verified)
  - *To reach the next level:* No isolation control exists to be on by default.
- **B L1:** Workflow code runs in the n8n instance with full network egress and access to stored credentials through credential-bearing nodes; how far it reaches beyond that depends on the instance's own sandbox settings. Evidence: [docs/SECURITY_HARDENING.md:5](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L5); [docs/SECURITY_HARDENING.md:19](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L19) (inferred)
  - *To reach the next level:* Execution limited to a workspace with no credentials and restricted egress.
- **Cap:** none

### C5 Untrusted input blast radius: 0.25 (high confidence)

The model reads content that others can write: execution data from workflows triggered by outside events, workflow names, notes and parameters, community templates and third-party node documentation. Results come back in a consistent JSON envelope, but nothing marks this content as untrusted or records where it came from, and the default tool set holds no read-only or no-egress mode. The same session can read instance data and create, run or delete workflows, so a successful injection can both leak data and act irreversibly without the server involving a human.

- **S L2:** Responses use a structured envelope (success, data, message) that separates server metadata from returned content. Evidence: [src/mcp/server.ts:1041-1050](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L1041-L1050); [src/mcp/handlers-n8n-manager.ts:4612-4615](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L4612-L4615) (verified)
  - *To reach the next level:* No provenance or untrusted flag on returned content.
- **C L1:** The envelope applies to every tool, but no source of attacker-controlled content (execution data, workflow text, templates, node docs) is distinguished. Evidence: searched `rg -n -i -e 'untrusted|provenance'` in `src/mcp/handlers-n8n-manager.ts src/mcp/handlers-workflow-diff.ts src/services/execution-processor.ts` → 0 hits (no untrusted or provenance marking in the main handlers or execution formatter) (verified)
  - *To reach the next level:* Distinguish every untrusted source, including execution data and template text.
- **D L2:** The envelope is always on, but no mode dropping a Rule-of-Two leg is enabled by default. Evidence: [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795); [src/mcp/tool-policy.ts:36-37](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tool-policy.ts#L36-L37) (verified)
  - *To reach the next level:* Read-only or no-egress mode on by default.
- **B L0:** In one default session the model can read execution data and workflow content, then create, run or delete workflows that reach any host and stored credential, with no human step in the server. Evidence: [src/mcp/tools-n8n-manager.ts:476-483](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L476-L483); [src/mcp/tools-n8n-manager.ts:385-386](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L385-L386); [src/mcp/handlers-n8n-manager.ts:1347-1352](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L1347-L1352) (verified)
  - *To reach the next level:* Exfiltration or irreversible actions would need to require approval.
- **Cap:** C5-WORSTCASE: B is L0 in the default configuration: data leak and irreversible actions are both reachable unattended.

### C6 Memory, context & configuration integrity: 0.10 (high confidence)

The server keeps no memory that is fed back to the model as instructions; its local store holds workflow backups. At startup, however, it reads settings from the directory it is launched in, without any trust decision. When the host is opened inside a repository, that repository can therefore change the server's configuration, including which tools are offered and how outbound URLs are checked, for every launch from that directory.

- **S L0:** Configuration is auto-loaded from a .env file in the working directory with no trust prompt, and its values feed security-relevant settings. Evidence: [src/config/n8n-api.ts:21-26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L21-L26); [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795); [src/utils/ssrf-protection.ts:308](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/utils/ssrf-protection.ts#L308) (verified)
  - *To reach the next level:* Security-relevant configuration only from user or operator scope.
- **C L0:** The working-directory configuration path is not controlled. Evidence: [src/config/n8n-api.ts:21-26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L21-L26) (verified)
  - *To reach the next level:* Every auto-loaded configuration path should be controlled.
- **D L0:** The working-directory load is always on, with no option to turn it off. Evidence: [src/config/n8n-api.ts:21-26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L21-L26) (verified)
  - *To reach the next level:* No auto-load from the working directory by default.
- **B L2:** A planted configuration persists for every launch from that directory and can change which tools are offered and how URLs are checked, but cannot itself trigger tool use. Evidence: [src/config/n8n-api.ts:21-26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L21-L26); [src/mcp/server.ts:791-795](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L791-L795) (verified)
  - *To reach the next level:* Workspace configuration would need to be session-scoped or easily inspected and purged.
- **Cap:** C6-REPOCONFIG: A .env file in the working directory can, without a trust decision, enable the management tools and change security-relevant settings.

### C7 Third-party extensions: 1.00 (high confidence)

The server loads no plugins, MCP servers, downloaded tools or model files at runtime; its tools, templates and documentation are bundled with the package. The optional connection to the n8n instance's own MCP endpoint is a remote API on the operator's instance, and its output is treated in the untrusted-input criterion. How the server itself is installed (npx) is the project's own supply chain and out of scope here.

- **Structural absence:** searched `rg -n -S -e 'loadPlugin|registerPlugin|StdioClientTransport|trust_remote_code|npm install -g|npx -y'` in `src/mcp src/services src/triggers src/utils src/templates src/http-server-single-session.ts` → 4 hits (hits are an update-command string, a troubleshooting string, and the MCP client in src/utils/mcp-client.ts, which only the separate n8n community node (src/n8n/MCPNode.node.ts) uses; nothing in the server loads third-party code)

### C8 Secrets & sensitive-data protection: 0.35 (high confidence)

The API key comes from the environment and is never put into tool output, and credential secrets returned by n8n are stripped before the model sees them. Logs record only argument metadata, not values. Telemetry is on by default and sends tool usage, workflow-change intents and sanitized workflow structures, including surviving node parameters, to the maintainer's telemetry service; sanitization is pattern-based. Workflows returned to the model are not scrubbed of secrets hard-coded in node parameters.

- **S L2:** Credential data is stripped from responses, tool-call logs carry only metadata, and telemetry payloads pass pattern-based sanitizers; the key itself sits in plaintext host configuration. Evidence: [src/mcp/handlers-n8n-manager.ts:4599-4600](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L4599-L4600); [src/utils/redaction.ts:55-60](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/utils/redaction.ts#L55-L60); [PRIVACY.md:26](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/PRIVACY.md#L26) (verified)
  - *To reach the next level:* No OS keychain or secret manager, and no redaction of workflow content before it reaches the model.
- **C L2:** Logs and telemetry are protected and credential responses are stripped, but workflow and execution content reach the model unredacted. Evidence: [src/mcp/server.ts:868-874](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L868-L874); [src/mcp/handlers-n8n-manager.ts:4599-4600](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L4599-L4600) (verified)
  - *To reach the next level:* Model-bound workflow and execution content are not redacted.
- **D L0:** Telemetry is enabled on first run unless an environment variable disables it, and sends workflow content and intent text to a third-party service. Evidence: [src/telemetry/config-manager.ts:197-198](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/telemetry/config-manager.ts#L197-L198); [src/telemetry/telemetry-types.ts:120](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/telemetry/telemetry-types.ts#L120); [PRIVACY.md:9](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/PRIVACY.md#L9) (verified)
  - *To reach the next level:* Telemetry should be opt-in and content-free.
- **B L1:** The leaked key would be a long-lived n8n API key with the owner's authority on the instance. Evidence: [src/services/n8n-api-client.ts:219-221](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L219-L221); [docs/SECURITY_HARDENING.md:5](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/docs/SECURITY_HARDENING.md#L5) (verified)
  - *To reach the next level:* Scoped, short-lived or rotatable keys.
- **Cap:** none

### C9 Audit & traceability: 0.20 (high confidence)

In the default stdio mode the logger drops all output, so tool calls are not recorded anywhere the operator can read. The only durable trace is the local backup store, which saves the previous version of a workflow, with the trigger that caused it, before full or partial updates; deletes, runs and credential changes leave no record. The model can delete or prune those backups with its own tool, and a failed backup does not stop the update.

- **S L1:** Workflow backups record a snapshot and the kind of update before edits; no tool-call record exists in stdio mode. Evidence: [src/utils/logger.ts:63-69](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/utils/logger.ts#L63-L69); [src/mcp/stdio-wrapper.ts:21-23](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/stdio-wrapper.ts#L21-L23); [src/database/schema.sql:205-212](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/database/schema.sql#L205-L212) (verified)
  - *To reach the next level:* No structured record of every tool call with arguments, status and time.
- **C L1:** Only full and partial workflow updates (and autofix) are recorded. Evidence: [src/mcp/handlers-workflow-diff.ts:245](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-workflow-diff.ts#L245) (verified)
  - *To reach the next level:* Deletes, runs, credential and data-table changes are not recorded.
- **D L1:** Backups are on by default but the model can skip them per call and delete or prune them through its own tool. Evidence: [src/mcp/handlers-workflow-diff.ts:245](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-workflow-diff.ts#L245); [src/mcp/tools-n8n-manager.ts:650](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L650) (verified)
  - *To reach the next level:* The record should be outside anything the model's tools can delete.
- **B L0:** A failed backup is logged (silently in stdio) and the update proceeds. Evidence: [src/mcp/handlers-n8n-manager.ts:1206-1211](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-n8n-manager.ts#L1206-L1211) (verified)
  - *To reach the next level:* Errors writing the record should be surfaced and records flushed per action.
- **Cap:** none

### C10 Limits & kill switch: 0.40 (high confidence)

The server bounds its own work in several places: API calls time out after 30 seconds by default, webhook calls after two minutes, responses over 1 MB are truncated and execution data is trimmed to a couple of items unless asked for more. The model can raise agent and lookup timeouts to ten minutes and request all execution items, and there is no rate or concurrency limit. Workflow and agent runs it starts keep running in n8n after a timeout or after the session stops.

- **S L2:** Server-enforced timeouts on API and webhook calls, a 1 MB response cap and a small default execution item limit. Evidence: [src/config/n8n-api.ts:9](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L9); [src/services/n8n-api-client.ts:1293](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/n8n-api-client.ts#L1293); [src/mcp/server.ts:1034-1037](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L1034-L1037); [src/services/execution-processor.ts:355-357](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/execution-processor.ts#L355-L357) (verified)
  - *To reach the next level:* No caps on every operation and no rate or concurrency limits.
- **C L2:** Timeouts apply to every n8n API request through the shared client and the response cap to every tool. Evidence: [src/config/n8n-api.ts:9](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/config/n8n-api.ts#L9); [src/mcp/server.ts:1034-1037](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/server.ts#L1034-L1037) (verified)
  - *To reach the next level:* Work started on the instance (workflow and agent runs) does not count against any limit.
- **D L1:** Defaults are sensible, but the model can raise per-call timeouts to ten minutes and remove the execution item limit. Evidence: [src/mcp/handlers-agents.ts:37](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/handlers-agents.ts#L37); [src/services/execution-processor.ts:355-357](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/services/execution-processor.ts#L355-L357) (verified)
  - *To reach the next level:* The model should not be able to raise its own limits.
- **B L1:** Agent and workflow runs continue in n8n after the call times out, and call volume is unbounded. Evidence: [src/mcp/tools-n8n-manager.ts:937](https://github.com/czlonkowski/n8n-mcp/blob/3a2ec761e006089638d281a27d15ba030ec3ec88/src/mcp/tools-n8n-manager.ts#L937) (verified)
  - *To reach the next level:* Stopping should cancel in-flight work and per-session ceilings should exist.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Execution data, workflow content, community templates and node docs returned to the model (src/mcp/tools-n8n-manager.ts:476, src/mcp/tools.ts:224) · [B] sensitive data/systems: n8n instance via the owner's API key; workflows can use every stored credential (src/services/n8n-api-client.ts:220) · [C] state change / egress: Create, update, delete and run workflows and agents (src/mcp/tools-n8n-manager.ts:13, 238, 385, 930) · Same default session? Yes

## Highest-impact improvements
1. Ship a read-only default (or a single READ_ONLY switch) that hides every write, delete and run operation until the operator enables them. (C1 D L0→L3, +0.150 before caps; Playbook 4)
2. Make telemetry opt-in and stop sending workflow parameters and intent text. (C8 D L0→L2, +0.100 before caps)
3. Split mixed read/write tools and set accurate destructive annotations on every mutating tool. (C2 S L1→L2, +0.075 before caps; Playbook 5)
4. Stop loading configuration from the working directory; read settings only from the host-provided environment or an explicit config path. (C6 S L0→L2, +0.150 before caps; Playbook 2)
5. Write an append-only local log of every tool call (tool, operation, ids, outcome, time) outside the model's reach in stdio mode. (C9 S L1→L2, +0.075 before caps; Playbook 1 step 3)

## Re-audit log
- C4 B: L0 → L1. Workflow code reaches credentials through credential-bearing nodes and has egress, matching L1; host-equivalent reach depends on n8n's own sandbox settings outside this repo (marked inferred).
- C10 D: L2 → L1. Agent and lookup timeoutMs accepts model values up to 600000 ms and itemsLimit -1 removes the execution item cap, so the model can raise its own limits.
- C2 B: L1 → L2. Full and partial updates back up the prior workflow by default with rollback, so the common edit case is reversible.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the stdio server with n8n API credentials configured. Without them only read-only documentation tools are offered; HTTP mode (AUTH_TOKEN, optional multi-tenant header credentials) and the separate n8n community node in src/n8n were not scored.
- What code inside created workflows can reach depends on the n8n instance's own settings (Code-node sandbox, node exclusions, API key scopes), which are outside this repository; C4 B is inferred.
- The 5,000-line management handler file and the HTTP server were reviewed along the main paths only, not line by line.
