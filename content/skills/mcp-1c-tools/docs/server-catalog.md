# MCP server catalog

| Server id | Purpose | Details |
|---|---|---|
| `edt-mcp` | Live 1C:EDT workspace: metadata discovery and typed mutation, BSL read/write, search, references, call hierarchy, validation markers, forms rendering, event log, DB update, debug / YAXUnit / profiling | [edt-mcp.md](edt-mcp.md) |
| `1C-docs-mcp` | Platform reference (`docsearch`, `docinfo`), routed `standards`, `formatspec` | [1C-docs-mcp.md](1C-docs-mcp.md) |
| `1c-ssl-mcp` | БСП / SSL API search | [1c-ssl-mcp.md](1c-ssl-mcp.md) |
| `1c-templates-fastcode` | Code templates, project vector memory (`remember` / `recall`) | [1c-templates-fastcode.md](1c-templates-fastcode.md) |
| `v8std-local` | ITS «Стандарты разработки» mirror: standards search, diagnostic → standard mapping (ACC/АПК, BSLLS, EDT, v8-code-style), snippet classification | [v8std.md](v8std.md) |
| `cognee`, `openviking` *(optional)* | Memory providers | [memory-providers.md](memory-providers.md) |
| `1c-qa` | QA MCP: test manager driving a 1C test client — UI checks under `UI_TESTING` | `content/skills/1c-qa-testing/SKILL.md` |
| `1c-kd20` *(conditional)* | «Конвертация данных» 2.0 exchange rules: matching, authoring, validation by the БСП runtime, registration rules, review of existing rules, optional sandbox checks; read-only EnterpriseData (КД 3) exchange manager checks; profiles and launch strings of the two exchange bases. Installed separately; only for exchange-rule tasks | [mcp-conversion-data20.md](mcp-conversion-data20.md) |

The classic multi-server docker distribution (graph, code-metadata, syntax-checker, code-check, data servers) is **not** part of this ruleset's default set; [edt-mcp.md](edt-mcp.md) (`→ Remap from the classic docker distribution`) maps its tools to `edt-mcp`.

### Optional pre-alpha servers

These are experimental projects, separate from the nine main servers above and not required by normal development gates. Client aliases vary; use the tools actually exposed in this session. Read their catalog only for a task that needs them; do not install, start a client, replay UI actions or write conversion files merely to check availability.

| Project / runtime server name | Purpose | Details |
|---|---|---|
| `MCP_Test` / `1C Visual UI Test` | Testing knowledge base, scenario preparation, test-client processes and UI replay | [mcp-test.md](mcp-test.md) |
