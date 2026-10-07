# Defense-in-Depth Score: DBHub

**Repo:** https://github.com/bytebase/dbhub · **Commit:** `4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb` · **Reviewed:** 2026-10-03
**What it is:** Token-conscious database MCP server for Postgres, MySQL, SQL Server, Oracle, MariaDB, SQLite
**Category:** Data & Analytics
**Scored configuration:** The README's lead invocation (package 1.4.0): npx @bytebase/dbhub --transport http --port 8080 --dsn <url>, with no TOML config, so execute_sql is write-enabled with no row limit or timeout; HTTP access control is not locked down.
**Agent surface (default):** code execution yes · filesystem write no · network egress no · external credentials yes · persistent memory no · untrusted input yes · third party extensions no · sub agents no · external communication no

## Score: 4.1 / 10.0 (Minimal)

| # | Criterion | S | C | D | B | Raw | Cap | Score | Confidence |
|---|---|---|---|---|---|---|---|---|---|
| C1 | Identity & least privilege | L1 | L0 | L0 | L2 | 0.17 | G1 | **0.17** | High |
| C2 | Approval gates | L3 | L3 | L0 | L3 | 0.60 | G1 | **0.50** (alt) | High |
| C3 | Tool & action scoping | L2 | L3 | L0 | L2 | 0.47 | G1 | **0.47** (alt) | High |
| C4 | Code-execution isolation | L0 | L0 | L0 | L1 | 0.05 | — | **0.05** | Medium |
| C5 | Untrusted input blast radius | L2 | L3 | L3 | L0 | 0.53 | C5-WORSTCASE | **0.25** | High |
| C6 | Memory, context & configuration integrity | L1 | L1 | SA | L1 | 0.40 | — | **0.40** | High |
| C7 | Third-party extensions | SA | SA | SA | SA | 1.00 | — | **1.00** (SA) | High |
| C8 | Secrets & sensitive-data protection | L2 | L2 | L3 | L2 | 0.55 | — | **0.55** | High |
| C9 | Audit & traceability | L2 | L2 | L2 | L0 | 0.40 | — | **0.40** | High |
| C10 | Limits & kill switch | L2 | L1 | L1 | L1 | 0.33 | — | **0.33** | High |

Controls where a risk surface exists: 3.18 / 9.0 (35%); 1 criterion scored SA (surface absent).

As shipped in the README's lead command, DBHub gives MCP clients full read-write SQL access to your database, with no row limit or timeout, over an HTTP transport whose access control is not locked down. Its read-only mode is well built (a statement classifier plus a read-only database transaction on every engine), but it is opt-in and can only be enabled through a TOML file. Use a least-privilege database account and set readonly and max_rows in TOML before exposing it; the MCPB bundle and Claude Code plugin already ship read-only.

## Critical gaps
- In the default configuration a hijacked host model can run destructive SQL (DROP, DELETE, UPDATE) unattended, so the worst case of a prompt injection from database content is irreversible damage to the database (C5-WORSTCASE). (ASI01, ASI02, LLM01, T6; C5) — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67)

## Criterion details

### C1 Identity & least privilege — 0.17 (high)

DBHub connects with one database credential supplied by the operator (the DSN) and uses it for every tool and every client. In the README's lead setup (HTTP transport), access control on the HTTP endpoint is not locked down. Read-only access is possible only through a TOML file; the old --readonly flag now exits with an error.

- **S L1:** Every tool runs on the single operator-supplied DSN credential; DBHub neither narrows it per tool nor issues per-request credentials (AWS IAM tokens are opt-in). — [src/config/env.ts:261-264](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L261-L264); [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67); [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189) (verified)
  - *To reach the next level:* No separate narrower credential for read tools versus write tools, and no deterministic per-request authorization before the credential is used.
- **C L0:** No per-principal authorization runs on the tool path, and HTTP access control is not locked down. (verified)
  - *To reach the next level:* No per-principal authorization policy on the main tool path.
- **D L0:** The README's lead command gives a write-enabled execute_sql over HTTP; least privilege requires a TOML file plus further hardening of HTTP access control. — [README.md:93](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/README.md#L93); [src/config/env.ts:30-31](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L30-L31); [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189) (verified)
  - *To reach the next level:* Default is not read-only; write access is not an explicit operator elevation.
- **B L2:** A hijacked or misused server can do whatever the DSN account can do in one database (read, write, DDL); TOML multi-source configs widen this to several databases. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [dbhub.toml.example:19-22](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/dbhub.toml.example#L19-L22) (The shipped example uses the postgres superuser, which also reaches server-side file and program functions.) (verified)
  - *To reach the next level:* Writes are not limited to non-destructive operations, and the credential is not scoped to one tenant or mostly read.
- **Cap:** G1 — The controls that would narrow who can use the credential and what it may do (TOML readonly mode and HTTP hardening options) exist but are off in the scored configuration.
- **Notes:** With the default stdio transport (src/config/env.ts:319-320) only the launching host process can call the server, which removes the network exposure but not the write-enabled default. The Claude Code plugin and MCPB bundle ship read-only TOML configs.

### C2 Approval gates — 0.50 (high)

As a tool server, DBHub's job is to tell the host which calls are dangerous. By default the single execute_sql tool both reads and writes, and it is honestly flagged as destructive, so a host can only gate all SQL or none. DBHub has a strong opt-in read-only mode: a per-tool TOML setting that rejects non-read statements and also runs them in a read-only database transaction, with matching read-only annotations. Because it is off by default and cannot be set from the command line, a wrongly approved call in the shipped setup can drop or delete production data with no undo.

- **default configuration** (default; raw 0.28 → 0.28)
  - **S L1:** Annotations are derived from the access policy and are accurate, but the default execute_sql mixes reads and writes in one tool, so the host cannot separate them. — [src/utils/tool-metadata.ts:166-170](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-metadata.ts#L166-L170); [src/tools/index.ts:93-98](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/index.ts#L93-L98); [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189) (verified)
    - *To reach the next level:* Separate read and write tools (or a read-only execute_sql by default) so readOnlyHint/destructiveHint split reads from writes.
  - **C L2:** Every registered tool, including TOML custom tools, gets annotations from the same policy or statement classifier; nothing mutating is flagged read-only by construction. — [src/tools/index.ts:156-166](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/index.ts#L156-L166); [src/utils/tool-metadata.ts:166-170](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-metadata.ts#L166-L170); [src/tools/index.ts:93-98](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/index.ts#L93-L98) (verified)
    - *To reach the next level:* Custom-tool annotations rest on a keyword classifier its authors call best-effort, so a statement calling a side-effecting function can be labelled read-only.
  - **D L1:** The shipped default is write-enabled; the read-only mode exists only in TOML and the --readonly flag was removed. — [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189); [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/config/env.ts:30-31](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L30-L31) (verified)
    - *To reach the next level:* Read-only by default, with writes enabled only by an explicit, loudly named operator setting.
  - **B L0:** In the default configuration execute_sql accepts DROP, TRUNCATE, DELETE and GRANT with no preview, checkpoint or undo. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/tools/execute-sql.ts:54-58](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L54-L58); [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67) (verified)
    - *To reach the next level:* No rollback, dry-run, or quantity bound on write statements in the default configuration.
- **opt-in read-only mode (TOML readonly = true)** (alt; raw 0.60, cap G1 → 0.50) ← counted
  - **S L3:** A server-enforced read-only mode: the classifier denies non-read classes (unknown statements deny), the engine runs the batch read-only, and annotations switch to readOnlyHint. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/utils/sql-access-policy.ts:79-86](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L79-L86); [src/connectors/postgres/index.ts:810-811](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L810-L811); [src/connectors/sqlite/index.ts:489-490](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/sqlite/index.ts#L489-L490); [src/connectors/oracle/index.ts:753-755](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/oracle/index.ts#L753-L755) (verified)
    - *To reach the next level:* No preview/dry-run tool by default (explain_sql is opt-in), and the classifier is self-described as best-effort.
  - **C L3:** Every tool path uses the same policy: execute_sql and custom tools are gated, explain_sql always runs read-only, search_objects is read-only by construction. — [src/tools/execute-sql.ts:54-58](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L54-L58); [src/tools/custom-tool-handler.ts:205](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L205); [src/tools/explain-sql.ts:107](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/explain-sql.ts#L107) (verified)
    - *To reach the next level:* The read/non-read boundary is a first-keyword allowlist plus name denylists, not a parsed statement, and its authors document bypasses.
  - **D L0:** Read-only must be enabled per tool in a TOML file; it is off in the README's lead invocation. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/config/env.ts:30-31](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L30-L31); [mcpb/dbhub.toml:20-24](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/mcpb/dbhub.toml#L20-L24) (verified)
    - *To reach the next level:* On by default.
  - **B L3:** If the classifier is bypassed, engine read-only transactions still reject DML on every connector; DDL on MySQL/Oracle and privilege-gated escape-hatch functions remain residual. — [src/connectors/postgres/index.ts:810-811](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L810-L811); [src/connectors/oracle/index.ts:753-755](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/oracle/index.ts#L753-L755); [src/utils/allowed-keywords.ts:139-141](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/allowed-keywords.ts#L139-L141) (verified)
    - *To reach the next level:* No rate limits on calls and the engine backstop does not cover DDL on every engine.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.
- **Notes:** The alternative scored is the per-tool readonly setting shipped on in the MCPB bundle and Claude Code plugin configs (mcpb/dbhub.toml, plugin/dbhub.toml).

### C3 Tool & action scoping — 0.47 (high)

The default execute_sql tool takes arbitrary SQL with no validation, row limit or timeout, against any table the credential can reach. Narrower options exist: TOML custom tools use parameterized statements with typed and enumerated arguments, search_objects validates and caps its inputs, and the opt-in read-only mode combines a keyword classifier with a database-level read-only transaction. None of these is the default, so a misused tool in the shipped setup can read or rewrite the whole database.

- **default configuration** (default; raw 0.12 → 0.12)
  - **S L0:** execute_sql passes the model's SQL string to the database unchanged when readonly is unset. — [src/tools/execute-sql.ts:19-21](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L19-L21); [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67) (verified)
    - *To reach the next level:* Some validation of the default SQL path (at least a read-only classifier or statement allowlist).
  - **C L1:** Only search_objects (enums, bounded limit, identifier quoting) and opt-in custom tools validate their arguments. — [src/tools/search-objects.ts:47-53](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/search-objects.ts#L47-L53); [src/tools/custom-tool-handler.ts:186-187](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L186-L187) (verified)
    - *To reach the next level:* The main tool, execute_sql, has no argument validation by default.
  - **D L1:** Two tools by default, but one is write-enabled raw SQL; a TOML [[tools]] list can drop it per source. — [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189); [src/tools/builtin-tools.ts:14-17](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/builtin-tools.ts#L14-L17) (verified)
    - *To reach the next level:* Read-only tool set by default with writes requiring explicit enabling.
  - **B L0:** A misused execute_sql can read, modify or drop any table the DSN account can reach, with no row or quantity bound by default. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/connectors/postgres/index.ts:800-804](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L800-L804) (verified)
    - *To reach the next level:* Scope tools to specific tables/operations and bound quantities (max_rows, statement allowlists) by default.
- **opt-in read-only mode with max_rows** (alt; raw 0.47, cap G1 → 0.47) ← counted
  - **S L2:** First-keyword allowlist with escape-hatch denylists, backed by an engine-level read-only transaction; custom tools use bound parameters. — [src/utils/allowed-keywords.ts:291-298](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/allowed-keywords.ts#L291-L298); [src/utils/allowed-keywords.ts:139-141](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/allowed-keywords.ts#L139-L141); [src/connectors/postgres/index.ts:810-811](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L810-L811) (verified)
    - *To reach the next level:* Name matching is bypassable per the authors; no parsed-statement validation or table allowlist.
  - **C L3:** The same policy gates execute_sql and custom tools; explain_sql is always read-only and rejects ANALYZE forms. — [src/tools/execute-sql.ts:54-58](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L54-L58); [src/tools/custom-tool-handler.ts:205](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L205); [src/tools/explain-sql.ts:53-55](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/explain-sql.ts#L53-L55) (verified)
    - *To reach the next level:* No single central policy layer that new tools inherit automatically.
  - **D L0:** Off by default; only TOML can enable it. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/config/env.ts:30-31](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L30-L31) (verified)
    - *To reach the next level:* Read-only tool set by default.
  - **B L2:** Reads of the whole database remain possible, bounded only by max_rows when the operator sets it. — [src/tools/execute-sql.ts:63-66](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L63-L66) (verified)
    - *To reach the next level:* No table or schema scoping; reads span every object the account can see.
- **Cap:** G1 — Opt-in mechanism: off in the scored default configuration.

### C4 Code-execution isolation — 0.05 (medium)

Raw SQL from the model is code, and DBHub adds no isolation around it: statements run in the database with the full privileges of the configured account, and SQLite databases are opened inside DBHub's own process. DBHub itself starts no subprocesses and evaluates no JavaScript. What a statement can reach depends entirely on the database account: with a superuser or highly privileged account, server-side features (file reads and writes, program execution) are within reach in the default write-enabled mode.

- **S L0:** No isolation primitive: SQL goes straight to the engine on the operator's connection, and SQLite runs in-process via node:sqlite. — [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67); [src/connectors/sqlite/index.ts:184-198](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/sqlite/index.ts#L184-L198); searched `rg -n 'child_process|spawn\(|execSync|execFile|eval\(|new Function' --glob '!**/__tests__/**'` in `src` → 0 hits (DBHub itself spawns no processes and evaluates no JS; the only interpreter of model text is the database engine via SQL.) (verified)
  - *To reach the next level:* Some boundary around model-written SQL (a restricted engine role, read-only session, or sandboxed database) applied by DBHub.
- **C L0:** With no isolation layer, no execution path is covered. — [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67); [src/tools/custom-tool-handler.ts:218-222](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L218-L222) (verified)
  - *To reach the next level:* Route every SQL execution path through an isolating layer.
- **D L0:** Nothing restricts execution by default; the read-only session mode is off unless configured in TOML. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/tools/registry.ts:186-189](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/registry.ts#L186-L189) (verified)
  - *To reach the next level:* An isolation default that ships on.
- **B L1:** Model-written SQL reaches everything the DSN account can; with superuser accounts this extends to server-side file and program functions on the database host, and in-process SQLite can create files as the OS user. — [dbhub.toml.example:19-22](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/dbhub.toml.example#L19-L22); [src/utils/allowed-keywords.ts:117-131](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/allowed-keywords.ts#L117-L131) (inferred)
  - *To reach the next level:* No credential, filesystem or network bound on what executed SQL can reach.
- **Cap:** none
- **Notes:** B is inferred: what SQL can reach depends on the operator's database account and on engine features (PostgreSQL COPY ... PROGRAM, MySQL INTO OUTFILE, SQL Server xp_cmdshell, SQLite ATTACH/VACUUM INTO), which DBHub's own classifier names but does not block in write mode.

### C5 Untrusted input blast radius — 0.25 (high)

Database contents are untrusted input: any row can carry instructions aimed at the model reading it. DBHub returns results as structured JSON that names the source and the SQL each result came from, which helps a host tell data from metadata, but nothing marks the rows as untrusted. Its read-only mode would drop the state-change leg, but it is off by default. In the README's HTTP setup, a hijacked session can delete or alter shared data that every other client of the server also uses, so the worst case is rated at the bottom.

- **S L2:** Tool results are JSON objects separating rows from metadata (source_id, per-statement sql, count, truncated). — [src/utils/response-formatter.ts:81-91](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/response-formatter.ts#L81-L91); [src/tools/execute-sql.ts:72-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L72-L76); [src/utils/tool-handler-helpers.ts:38-49](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-handler-helpers.ts#L38-L49) (verified)
  - *To reach the next level:* No untrusted-content flag on returned rows that a host could act on.
- **C L3:** Every tool (built-in and custom, success and error) returns through the same JSON formatter; tool descriptions are built from operator config only. — [src/utils/response-formatter.ts:81-91](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/response-formatter.ts#L81-L91); [src/utils/tool-metadata.ts:160-162](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-metadata.ts#L160-L162) (verified)
  - *To reach the next level:* Only principals should be able to instruct; DBHub does not bind HTTP clients to principals.
- **D L3:** Structured output is always on and nothing in a tool call or result can turn it off. — [src/utils/response-formatter.ts:81-91](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/response-formatter.ts#L81-L91); [src/tools/custom-tool-handler.ts:227-232](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L227-L232) (verified)
  - *To reach the next level:* An untrusted-data marker that is on by default and not configurable.
- **B L0:** If a host model is hijacked by database content, the default write-enabled execute_sql can delete or rewrite data unattended; in HTTP mode the same shared credential serves every client, so other users' data is in reach. — [src/utils/sql-access-policy.ts:73-76](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/sql-access-policy.ts#L73-L76); [src/tools/execute-sql.ts:67](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L67) (verified)
  - *To reach the next level:* Drop a Rule-of-Two leg by default (read-only mode on, or auth that binds each client to a principal).
- **Cap:** C5-WORSTCASE — Worst case (B L0): a hijacked agent can leak data and take irreversible actions unattended.
- **Notes:** Single-user stdio deployments would rate B at L1 (irreversible writes unattended, but DBHub itself provides no outbound channel). The multi-user rule lowers it one level for the HTTP mode the README leads with.

### C6 Memory, context & configuration integrity — 0.40 (high)

DBHub has no memory, retrieval store, or instruction files. The TOML config is loaded only from an explicit --config path and is hot-reloaded from that path. However, in TOML mode and in the env-var fallback mode, DBHub silently loads a .env file from the current working directory first, which can supply the database DSN, switch the transport to HTTP, or disable the DNS-rebinding check when those values are not already set. The README's --dsn invocation does not load it.

- **S L1:** A .env from the working directory is loaded silently in TOML and fallback modes and can set security-relevant settings that are read afterwards. — [src/config/env.ts:92-97](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L92-L97); [src/config/env.ts:709-711](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L709-L711); [src/config/env.ts:277-278](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L277-L278); [src/config/env.ts:456-458](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L456-L458) (verified)
  - *To reach the next level:* Workspace files must not be able to change security settings (allowed hosts, transport, DSN) without a trust decision.
- **C L1:** The TOML config is restricted to an explicit --config path (no cwd discovery), but the .env loader is not. — [src/config/toml-loader.ts:64-71](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/toml-loader.ts#L64-L71); [src/config/env.ts:92-97](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L92-L97) (verified)
  - *To reach the next level:* Apply the same explicit-path rule to .env loading.
- **D SA:** No memory store or namespaces exist to isolate. — searched `rg -n -i 'remember|vector|embedding|long-term' --glob '!**/__tests__/**'` in `src` → 0 hits (No memory store, vector index, or remember tool exists.) (verified)
- **B L1:** A planted .env persists across sessions and changes which database and HTTP protections every later run uses. — [src/config/env.ts:709-711](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L709-L711); [src/server.ts:162-165](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/server.ts#L162-L165) (verified)
  - *To reach the next level:* Loaded config should be session-scoped or require review before use.
- **Cap:** none
- **Notes:** C6-REPOCONFIG was considered and not applied: its trigger (cwd .env loaded with no trust decision) is verified in TOML mode (src/config/env.ts:709-711), which the Claude Code plugin and MCPB bundle use, but the README's lead --dsn invocation returns before any .env load (src/config/env.ts:261-264). dotenv not overriding variables already set in the real environment is library behaviour relied on here.

### C7 Third-party extensions — 1.00 (high)

DBHub loads no third-party code at runtime: there is no plugin system, no MCP client, and no model-driven package installs. Database drivers and optional cloud SDKs are fixed, string-literal imports that ship with the package, which is the project's own build supply chain and out of scope here.

- **Structural absence:** searched `rg -n 'import\(' --glob '!**/__tests__/**' --glob '!*.d.ts'` in `src` → 15 hits (All 15 hits are string-literal specifiers: DBHub's own connector modules (src/index.ts), node:sqlite, the demo loader, the tool registry, a type-only import, and the optional @aws-sdk/rds-signer, @aws-sdk/credential-providers and @azure/identity SDKs. None takes a model-, user- or config-supplied module path.); searched `rg -n -i 'plugin|extension' --glob '!**/__tests__/**'` in `src` → 7 hits (Hits are a comment about the policy 'confirm' extension point, fixture-helper comments about .toml file extensions, and MySQL/MariaDB authentication plugins; none is an extension loader.)

### C8 Secrets & sensitive-data protection — 0.55 (high)

Database passwords are redacted when DSNs appear in startup logs and connection errors, the web API omits passwords and SSH credentials, and there is no telemetry. Credentials come from command-line flags, environment variables, or config files in plain text, and the SSH tunnel and the HTTP-mode request log (which holds full SQL text) do not fully protect sensitive data.

- **S L2:** DSN passwords are masked in logs and errors, and the sources API strips passwords. — [src/connectors/manager.ts:147](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/manager.ts#L147); [src/config/env.ts:520-521](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L520-L521); [src/api/sources.ts:39-51](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/api/sources.ts#L39-L51) (verified)
  - *To reach the next level:* No keychain or secret-manager support of DBHub's own, and credentials are passed on the command line in the README's lead command.
- **C L2:** Logs and errors are covered, but the request log of SQL text is not protected and custom-tool errors echo statement and parameter values. — [src/utils/tool-handler-helpers.ts:100-110](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-handler-helpers.ts#L100-L110); [src/tools/custom-tool-handler.ts:248](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L248) (verified)
  - *To reach the next level:* Redact or protect the request log and error payloads, not just DSN-bearing log lines.
- **D L3:** No telemetry at all; DSN redaction is always on and not configurable. — searched `rg -n -i 'sentry|posthog|telemetry|opentelemetry'` in `src package.json` → 0 hits; [src/connectors/manager.ts:147](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/manager.ts#L147) (verified)
  - *To reach the next level:* Minimise or protect stored transcripts by default.
- **B L2:** A leak exposes a long-lived database password (and any SSH password/key) scoped to one account; none is reachable through the tools. (verified)
  - *To reach the next level:* Short-lived, rotatable credentials by default (AWS IAM tokens are opt-in).
- **Cap:** none
- **Notes:** The SSH observation relies on library behaviour.

### C9 Audit & traceability — 0.40 (high)

Every tool call is recorded with its tool name, SQL, timestamp, duration, success and error, and the client's User-Agent. The record lives only in memory, keeps the last 100 calls per source, and disappears on restart; nothing is written to disk or shipped elsewhere, and tool calls are not logged to the console. A model can push its own earlier calls out of the record simply by making more calls.

- **S L2:** A structured per-call record with SQL text, status and timing. — [src/utils/tool-handler-helpers.ts:100-110](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-handler-helpers.ts#L100-L110); [src/requests/store.ts:22-33](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/requests/store.ts#L22-L33) (verified)
  - *To reach the next level:* No actor attribution beyond the User-Agent, and custom-tool argument values are not recorded.
- **C L2:** execute_sql, search_objects, explain_sql, health_check and custom tools all call trackToolRequest. — [src/utils/tool-handler-helpers.ts:100-110](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/utils/tool-handler-helpers.ts#L100-L110); [src/tools/search-objects.ts:735](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/search-objects.ts#L735); [src/tools/custom-tool-handler.ts:254](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/custom-tool-handler.ts#L254) (verified)
  - *To reach the next level:* Read-only denials are recorded but config reloads and credential use are not.
- **D L2:** On by default and written by server code, but the 100-entry FIFO lets a caller evict its own history by volume. — [src/requests/store.ts:22-33](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/requests/store.ts#L22-L33) (verified)
  - *To reach the next level:* A record the caller cannot displace (no FIFO eviction driven by request volume).
- **B L0:** Records are in memory only and lost on restart or crash; eviction is silent. — [src/requests/store.ts:22-33](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/requests/store.ts#L22-L33); searched `rg -n 'appendFile|createWriteStream|writeFileSync' --glob '!**/__tests__/**'` in `src` → 0 hits (No code writes a durable log or audit file.) (verified)
  - *To reach the next level:* Durable, per-action flushed records.
- **Cap:** none

### C10 Limits & kill switch — 0.33 (high)

search_objects has a server-enforced result cap (default 100, maximum 1000). The main execute_sql tool has no row limit and no query timeout unless the operator sets max_rows and query_timeout in a TOML file, and neither can be set with --dsn. Tool handlers ignore MCP cancellation, so a cancelled or abandoned query keeps running on the database.

- **S L2:** Some operations are capped by the server (search_objects limit); execute_sql caps and timeouts exist but are opt-in. — [src/tools/search-objects.ts:47-53](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/search-objects.ts#L47-L53); [src/connectors/postgres/index.ts:209-212](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L209-L212) (verified)
  - *To reach the next level:* Caps on every operation plus concurrency or rate limits.
- **C L1:** Only search_objects is bounded by default; execute_sql and custom tools are unbounded. — [src/tools/search-objects.ts:47-53](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/search-objects.ts#L47-L53); [src/tools/execute-sql.ts:63-66](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/tools/execute-sql.ts#L63-L66) (verified)
  - *To reach the next level:* Default bounds on execute_sql and custom tools as well.
- **D L1:** The main tool is unlimited by default; the max-rows flag was removed in favour of TOML. — [src/config/env.ts:44-45](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/config/env.ts#L44-L45); [src/connectors/postgres/index.ts:209-212](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L209-L212) (verified)
  - *To reach the next level:* Sensible default row cap and query timeout on execute_sql.
- **B L1:** A runaway query has no ceiling by default, and stopping the call does not cancel it on the database. — searched `rg -n 'AbortSignal|signal\.aborted|extra\.signal'` in `src` → 0 hits (No tool handler observes MCP request cancellation, so a cancelled call keeps running on the database.); [src/connectors/postgres/index.ts:209-212](https://github.com/bytebase/dbhub/blob/4ddb26e73c5d8f1d54f5d04351c11c3caeb43cbb/src/connectors/postgres/index.ts#L209-L212) (verified)
  - *To reach the next level:* Moderate default ceilings and cancellation of in-flight queries.
- **Cap:** none

## Rule-of-Two check
[A] untrusted input: Database rows returned to the model by execute_sql (src/tools/execute-sql.ts:67) · [B] sensitive data/systems: Production database contents via the operator's DSN credential (src/config/env.ts:261-264) · [C] state change / egress: Write-enabled execute_sql by default (src/utils/sql-access-policy.ts:74, src/tools/registry.ts:189) · Same default session? Yes

## Highest-impact improvements
1. Make execute_sql read-only by default and require an explicit, loudly named setting (TOML readonly = false or an --allow-writes flag) to enable writes. — C2 D L1→L3, +0.100 before caps (Playbook 5)
2. Harden the HTTP transport's default network exposure and access control. — C1 D L0→L2, +0.100 before caps (Playbook 4)
3. Ship default max_rows and query_timeout values for execute_sql and expose them as CLI flags again. — C10 D L1→L2, +0.050 before caps (Playbook 3 step 3)
4. Write each tool call as a structured line to stderr or an append-only file in addition to the in-memory store. — C9 B L0→L2, +0.100 before caps (Playbook 1 step 3)
5. Mark returned rows as untrusted data in the tool result (for example a provenance block with an untrusted flag) so hosts can act on it. — C5 S L2→L3, +0.075 before caps (Playbook 1)

## Re-audit log
- No changes.

## Limitations
- Static source review of the pinned commit only; nothing was executed, installed, or probed.
- Scored the README's lead HTTP invocation with --dsn; the default stdio transport removes network exposure, and the MCPB bundle and Claude Code plugin ship read-only TOML configs with max_rows = 1000, which would score materially higher on C2, C3 and C5.
- The workbench frontend (frontend/, 104 files) and docs/ were not reviewed beyond a search for raw-HTML rendering (no dangerouslySetInnerHTML or innerHTML hits).
- Library behaviour relied on without running it: dotenv not overriding existing variables, the SSH library's verification behaviour, and database-engine features (COPY PROGRAM, INTO OUTFILE, xp_cmdshell, SQLite ATTACH) reachable from write-mode SQL.
- No reviewer-injection attempts were found in README, CLAUDE.md, skills, or source comments.
