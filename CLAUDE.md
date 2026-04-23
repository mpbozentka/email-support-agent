# Email Support Agent — Gadget & More Electronics

AI-powered customer support email agent built with n8n. Monitors Gmail for support questions about returns, warranties, shipping, billing, privacy, and gift cards. Classifies emails, searches a vector store knowledge base, and drafts responses using Anthropic Claude.

## Architecture

Two n8n workflows:

### Workflow 1: KB Ingestion (`3BXsF6LwP9yjVYyW`)
Ingests FAQ and Policy documents into a vector store for semantic search.
- **Manual Trigger** → **Code Node** (FAQ & Policy text) → **Simple Vector Store** (insert mode)
- Sub-nodes: **Embeddings Google Gemini** (text-embedding-004), **Default Data Loader**, **Character Text Splitter**
- Memory key: `gadget_more_kb` (shared across workflows)
- Data table `CROWUZ4zTKrjomNv` also holds structured KB rows (15 entries, 6 categories)

### Workflow 2: Email Agent (`Zqj6jqE8uFI5nMCe`)
Monitors Gmail, classifies support emails, drafts responses.
- **Gmail Trigger** (polls for unread) → **Text Classifier** → routes:
  - `support_relevant` → **AI Agent** → **Create Draft Reply** (Gmail draft)
  - `not_relevant` → **Skip** (NoOp)
- AI sub-nodes: **Anthropic Chat Model** (claude-sonnet-4-20250514) for classifier + agent
- Tool: **Knowledge Base** (Simple Vector Store, retrieve-as-tool mode, key: `gadget_more_kb`)
- Embeddings: **Google Gemini** (text-embedding-004)

## Credentials Required
- **Gmail OAuth2** — for Gmail Trigger and Create Draft Reply nodes
- **Anthropic API** — for both Anthropic Chat Model nodes
- **Google Gemini API** — for Embeddings Google Gemini nodes (both workflows)

## Source Documents
- `FAQs.md` — 7 Q&A pairs covering shipping, returns, warranty, billing, privacy, gift cards
- `Policies.md` — Full store policies (effective Jan 1, 2026): shipping, returns, warranty, pricing, billing, privacy, gift cards

## Key Details
- Vector store memory key `gadget_more_kb` is shared between ingestion and agent workflows
- Ingestion must run before the email agent (vector store is in-memory, clears on n8n restart)
- Email agent creates **drafts** (not sends) — human reviews before sending
- Text Classifier outputs: `support_relevant` (index 0), `not_relevant` (index 1)

---

AI agency workflow automation workspace. Uses n8n-mcp (MCP server) + n8n-skills (Claude skills) to build, validate, and deploy n8n workflows.

## Workflow Building Process

Follow this sequence when building workflows:

1. **Plan** — Identify the pattern (webhook, HTTP API, database, AI agent, scheduled)
2. **Discover** — `search_nodes` to find the right nodes, `search_templates` to find existing solutions
3. **Configure** — `get_node` with `detail="standard"` for node setup (covers 95% of cases)
4. **Build** — `n8n_create_workflow` or `n8n_deploy_template` to start, then `n8n_update_partial_workflow` for incremental changes
5. **Validate** — `n8n_validate_workflow` after every update (expect 2-3 validation cycles)
6. **Fix** — `n8n_autofix_workflow` for common issues, manual fix for the rest
7. **Test** — `n8n_test_workflow` to execute and verify
8. **Activate** — `n8n_update_partial_workflow` with `activateWorkflow` operation

## n8n-mcp Tools — Quick Reference

### Discovery
- `search_nodes` — Find nodes by keyword
- `get_node` — Node info with detail levels: `minimal`, `standard` (default), `full`; modes: `info`, `docs`, `search_properties`, `versions`
- `search_templates` — Modes: `keyword`, `by_nodes`, `by_task`, `by_metadata`
- `get_template` — Full template details
- `tools_documentation` — Meta-docs for all MCP tools

### Workflow Management
- `n8n_create_workflow` — Create new workflow
- `n8n_get_workflow` — Get workflow (modes: `full`, `structure`, `minimal`)
- `n8n_list_workflows` — List with filtering/pagination
- `n8n_update_partial_workflow` — Incremental updates (19 operation types). **Most used tool** (99% success rate)
- `n8n_update_full_workflow` — Full workflow replacement (use sparingly)
- `n8n_delete_workflow` — Permanent delete
- `n8n_deploy_template` — Deploy template directly to instance
- `n8n_workflow_versions` — Version history and rollback

### Validation & Testing
- `validate_node` — Single node validation (profiles: `runtime`, `ai-friendly`, `strict`)
- `validate_workflow` — Offline workflow structure validation
- `n8n_validate_workflow` — Validate workflow by ID on instance
- `n8n_autofix_workflow` — Auto-fix common issues
- `n8n_test_workflow` — Execute workflow for testing
- `n8n_executions` — View execution history

### Instance Management
- `n8n_health_check` — Check instance status
- `n8n_audit_instance` — Audit instance configuration
- `n8n_manage_credentials` — Credential management
- `n8n_manage_datatable` — Data table CRUD + filtering + dry-run

## n8n Skills — When They Activate

| Skill | Triggers When |
|-------|--------------|
| `n8n-mcp-tools-expert` | Finding nodes, using MCP tools, workflow management |
| `n8n-node-configuration` | Configuring node properties, understanding field dependencies |
| `n8n-workflow-patterns` | Designing workflow structure, choosing architecture |
| `n8n-expression-syntax` | Writing `{{ }}` expressions, data mapping between nodes |
| `n8n-validation-expert` | Interpreting validation errors, fixing false positives |
| `n8n-code-javascript` | Writing JS in Code nodes, `$input`/`$json`/`$helpers` |
| `n8n-code-python` | Writing Python in Code nodes (use JS for 95% of cases) |

Skills are complementary — use multiple in sequence: patterns -> tools expert -> node config -> expressions -> validation.

## Critical Gotchas

### nodeType Format Mismatch
- **search/validate tools**: `nodes-base.slack` (short format)
- **workflow JSON**: `n8n-nodes-base.slack` (full format)
- Always use the correct format for the context

### Webhook Data Path
- Webhook data lives at `$json.body.field`, NOT `$json.field`
- This is the #1 expression bug — always check webhook payloads

### Code Node Return Format
- Must return `[{json: {...}}]` — array of objects with `json` key
- Missing this structure = silent failure

### Expressions vs Code Nodes
- Expressions use `{{ }}` wrapper: `{{$json.body.email}}`
- Code nodes use direct JS: `$input.first().json.body.email`
- Never mix these contexts

### Validation Loop
- Expect 2-3 validation cycles per workflow update
- Average: 23s thinking + 58s fixing per cycle
- Use `runtime` profile for most cases, `strict` for production deploys

### IF/Switch Node Parameters
- IF node: `branch="true"` or `branch="false"`
- Switch node: `case=0`, `case=1`, etc.

### Binary Operators
- Auto-sanitized on updates: binary operators remove `singleValue`, unary operators add it
- Don't manually set these — let the system handle it

## Workflow Patterns

Use these proven architectures (from analysis of real n8n workflows):

1. **Webhook Processing** (35% of workflows) — Webhook trigger -> validate -> process -> respond
2. **HTTP API Integration** — Schedule/trigger -> HTTP Request -> transform -> output
3. **Database Operations** — Trigger -> query -> transform -> update
4. **AI Agent Workflows** — Trigger -> AI Agent node -> tool nodes -> output
5. **Scheduled Tasks** — Cron trigger -> fetch -> process -> notify

## Tool Usage Tips

- `get_node(detail="standard")` covers 95% of configuration needs — don't default to `full`
- `n8n_update_partial_workflow` supports `patchNodeField` for surgical edits — prefer over full node replacement
- `search_templates(mode="by_task")` finds workflows by what they do, not just keywords
- Always run `n8n_health_check` at session start to confirm instance connectivity
- Use `n8n_manage_datatable` with `dry_run=true` before destructive operations

## Project Structure

```
n8n-builder/
├── n8n-mcp/       # MCP server — node DB, validation, workflow management
│   ├── src/       # Server source code
│   ├── data/      # Versioned node data
│   ├── docs/      # 26 setup/deployment/reference docs
│   └── tests/     # Unit + integration tests
└── n8n-skills/    # Claude skills — expert guidance layer
    ├── skills/    # 7 skill implementations
    ├── docs/      # Usage, development, code node best practices
    └── evaluations/ # Test scenarios per skill
```
