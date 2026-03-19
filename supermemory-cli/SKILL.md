---
name: supermemory-cli
description: Use the Supermemory CLI to programmatically manage memories, documents, profiles, tags, connectors, keys, and teams from the terminal. Covers all commands, flags, and usage patterns.
---

The Supermemory CLI is the complete interface to Supermemory from the terminal. It lets you add memories, search, manage documents, configure projects, connect external data sources, and administer teams — all programmatically.

## Installation & Auth

```bash
# Install globally
npm install -g @supermemory/cli

# Authenticate (opens browser OAuth flow)
supermemory login

# Or use an API key directly
supermemory login --api-key sm_abc_xxx

# Check auth status
supermemory whoami
```

## Configuration

The CLI supports three config scopes:

| Scope | File | Use Case |
|-------|------|----------|
| `project` | `.supermemory/config.json` (gitignored) | Per-machine secrets, API keys |
| `team` | `.supermemory/team.json` (committed) | Shared tag, team-wide defaults |
| `global` | `~/.config/supermemory/config.json` | All projects on this machine |

```bash
# Interactive setup wizard
supermemory init

# Non-interactive
supermemory init --scope project --tag my-bot
supermemory init --scope team --tag shared-project
supermemory init --scope global --output json

# View/set config values
supermemory config
supermemory config set tag my-project
supermemory config set verbose true
supermemory config get tag
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `SUPERMEMORY_API_KEY` | API key for authentication |
| `SUPERMEMORY_TAG` | Override default container tag |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry collector endpoint |

### Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Force JSON output (machine-readable) |
| `--tag` | Override default container tag for this command |
| `--help` | Show help for any command |

---

## Core Commands

### `add` — Ingest content

Ingest text, files, or URLs and extract memories into a container tag.

```bash
# Add text
supermemory add "User prefers TypeScript over JavaScript"

# Add a file (PDF, markdown, text, etc.)
supermemory add ./meeting-notes.pdf --tag meetings

# Add a URL
supermemory add https://docs.example.com/api --tag docs

# Read from stdin
cat file.txt | supermemory add --stdin
echo "some content" | supermemory add --stdin --tag notes

# With metadata
supermemory add "Design doc v2" --metadata '{"category": "engineering", "priority": "high"}'

# Custom document ID
supermemory add "My doc" --id custom-doc-id --tag project

# Batch mode (JSONL from stdin, each line: {"content": "...", "tag": "..."})
cat batch.jsonl | supermemory add --batch
```

**Options:**
- `--tag <string>` — Container tag
- `--stdin` — Read content from stdin
- `--title <string>` — Document title
- `--metadata <json>` — JSON metadata object
- `--id <string>` — Custom document ID
- `--batch` — Batch mode, reads JSONL from stdin

### `search` — Search memories

Semantic search across memories with filtering and reranking.

```bash
# Basic search
supermemory search "authentication patterns"

# Scoped to a tag with limit
supermemory search "auth" --tag api --limit 5

# With reranking for better relevance
supermemory search "database migrations" --rerank

# Query rewriting (LLM rewrites query for better results)
supermemory search "how do we handle auth" --rewrite

# Different search modes
supermemory search "user prefs" --mode memories    # extracted memories only
supermemory search "user prefs" --mode hybrid      # memories + document chunks
supermemory search "user prefs" --mode documents   # full documents only

# Include specific fields
supermemory search "api design" --include summary,chunks,memories

# Metadata filtering
supermemory search "design" --filter '{"AND": [{"key": "category", "value": "engineering"}]}'

# Similarity threshold (0-1)
supermemory search "preferences" --threshold 0.5
```

**Options:**
- `--tag <string>` — Filter by container tag
- `--limit <number>` — Max results (default: 10)
- `--threshold <number>` — Similarity threshold 0-1 (default: 0)
- `--rerank` — Enable reranking for better relevance
- `--rewrite` — Rewrite query using LLM
- `--mode <string>` — `memories` | `hybrid` | `documents` (default: memories)
- `--include <string>` — Fields: summary, chunks, memories, document
- `--filter <json>` — Metadata filter object

### `remember` — Store a memory directly

Store a specific fact or memory without ingesting a full document.

```bash
# Store a memory
supermemory remember "User prefers dark mode" --tag user_123

# Mark as permanent/static (won't decay)
supermemory remember "User is a senior engineer at Acme Corp" --static --tag user_123

# With metadata
supermemory remember "Discussed Q3 roadmap" --tag meetings --metadata '{"type": "decision"}'
```

**Options:**
- `--tag <string>` — Container tag
- `--static` — Mark as permanent memory
- `--metadata <json>` — JSON metadata

### `forget` — Delete a memory

```bash
# Forget by memory ID
supermemory forget mem_abc123 --tag default

# Forget by content match
supermemory forget --content "outdated preference" --tag default

# With reason
supermemory forget mem_abc123 --reason "User corrected this information"
```

**Options:**
- `--tag <string>` — Container tag
- `--reason <string>` — Reason for forgetting
- `--content <string>` — Forget by content match instead of ID

### `update` — Update an existing memory

```bash
supermemory update mem_123 "Updated preference: prefers Bun over Node"
supermemory update mem_123 "New content" --metadata '{"updated": true}'
```

**Options:**
- `--tag <string>` — Container tag
- `--metadata <json>` — Updated metadata
- `--reason <string>` — Reason for update

### `profile` — Get user profile

Retrieve the auto-generated user profile for a container tag.

```bash
# Get profile for default tag
supermemory profile

# Get profile for specific tag
supermemory profile user_123

# Search within the profile
supermemory profile user_123 --query "programming preferences"
```

Returns static facts (long-term) and dynamic context (recent activity).

---

## Document Management

### `docs` — Manage documents

```bash
# List documents
supermemory docs list --tag default --limit 20

# Get a specific document
supermemory docs get doc_abc123

# Check processing status
supermemory docs status doc_abc123

# View document chunks
supermemory docs chunks doc_abc123

# Delete a document
supermemory docs delete doc_abc123 --yes
```

**Subcommands:** `list`, `get`, `delete`, `chunks`, `status`

---

## Container Tags

### `tags` — Manage container tags

Container tags scope memories to users, projects, or any logical grouping.

```bash
# List all tags
supermemory tags list

# Get tag info (memory count, doc count, etc.)
supermemory tags info user_123

# Create a new tag
supermemory tags create my-new-project

# Set context on a tag (injected into profile responses)
supermemory tags context user_123 --set "This user is a premium customer"

# Clear context
supermemory tags context user_123 --clear

# Merge tags (moves all memories from source to target)
supermemory tags merge old-tag --into new-tag --yes

# Delete a tag
supermemory tags delete old-tag --yes
```

**Subcommands:** `list`, `info`, `create`, `delete`, `context`, `merge`

---

## API Keys

### `keys` — Manage API keys

```bash
# List all keys
supermemory keys list

# Create a new key
supermemory keys create --name my-agent --permission write

# Create a scoped key (restricted to one container tag)
supermemory keys create --name bot-key --tag user_123 --expires 30

# Revoke a key
supermemory keys revoke key_abc123 --yes

# Toggle a key on/off
supermemory keys toggle key_abc123
```

**Subcommands:** `list`, `create`, `revoke`, `toggle`

---

## Connectors

### `connectors` — External data sources

Connect Google Drive, Notion, OneDrive, and other services to automatically sync documents.

```bash
# List connected services
supermemory connectors list

# Connect a service (opens OAuth flow)
supermemory connectors connect google-drive --tag docs
supermemory connectors connect notion --tag notes

# Trigger a sync
supermemory connectors sync conn_abc123

# View sync history
supermemory connectors history conn_abc123 --limit 10

# List synced resources
supermemory connectors resources conn_abc123

# Disconnect
supermemory connectors disconnect conn_abc123 --yes
```

**Subcommands:** `list`, `connect`, `sync`, `history`, `disconnect`, `resources`

---

## Plugins

### `plugins` — IDE and tool integrations

Connect the CLI to Claude Code, Cursor, and other tools.

```bash
# List available plugins
supermemory plugins list

# Connect a plugin (with auto-configuration)
supermemory plugins connect claude-code --auto-configure
supermemory plugins connect cursor --auto-configure

# Check plugin status
supermemory plugins status claude-code

# Revoke a plugin connection
supermemory plugins revoke claude-code --yes
```

**Subcommands:** `list`, `connect`, `revoke`, `status`

---

## Team Management

### `team` — Manage team members

```bash
# List team members
supermemory team list

# Invite a member
supermemory team invite user@example.com --role admin
supermemory team invite user@example.com --role member

# Change a member's role
supermemory team role member_123 admin

# Remove a member
supermemory team remove member_123 --yes

# View pending invitations
supermemory team invitations
```

**Subcommands:** `list`, `invite`, `remove`, `role`, `invitations`

---

## Monitoring

### `status` — Account dashboard

```bash
supermemory status
supermemory status --period 7d    # 24h, 7d, 30d, all
```

Shows memory count, document count, API usage, and storage.

### `logs` — Request logs

```bash
# Recent logs
supermemory logs

# Filter by time period
supermemory logs --period 7d

# Filter by status
supermemory logs --status error

# Filter by request type
supermemory logs --type search

# Get a specific log entry
supermemory logs get req_abc123
```

### `billing` — Usage and billing

```bash
supermemory billing          # Overview
supermemory billing usage    # Detailed usage breakdown
supermemory billing invoices # Past invoices
```

---

## Utility Commands

```bash
# Open Supermemory console in browser
supermemory open
supermemory open graph       # Memory graph view
supermemory open billing
supermemory open settings
supermemory open docs
supermemory open keys

# Machine-readable help (useful for LLM agents)
supermemory help --json
supermemory help --all

# Per-command help
supermemory search --help
supermemory tags --help --json
```

---

## Scoped Keys Mode

When using a scoped API key (restricted to a single container tag), only these commands are available:

`search`, `add`, `remember`, `forget`, `update`, `profile`, `whoami`

The tag is automatically set from the key's scope — no need to pass `--tag`.

---

## Common Patterns

### Pipe content from other tools

```bash
# Ingest git log
git log --oneline -20 | supermemory add --stdin --tag git-history

# Ingest command output
curl -s https://api.example.com/docs | supermemory add --stdin --tag api-docs

# Pipe search results to jq
supermemory search "auth" --json | jq '.results[].memory'
```

### Scripting with JSON output

```bash
# All commands support --json for machine-readable output
supermemory search "query" --json
supermemory tags list --json
supermemory profile user_123 --json
supermemory status --json
```

### CI/CD integration

```bash
# Use env var for auth
export SUPERMEMORY_API_KEY=sm_abc_xxx

# Ingest docs on deploy
supermemory add ./docs/api-reference.md --tag api-docs --id api-ref-latest

# Check status
supermemory status --json | jq '.memoryCount'
```
