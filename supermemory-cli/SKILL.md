---
name: supermemory-cli
description: Use the Supermemory CLI (`supermemory` 5.x on npm) from the terminal or from scripts and agents. Add content, search, read profiles, forget memories, manage documents, namespaces, connectors, API keys and team members, install coding-agent plugins, migrate a project to the v5 API, and run the local self-hosted server. Covers every command, subcommand, flag and the JSON output contract.
---

The CLI ships inside the `supermemory` npm package and calls the v5 API. Every content command works in one **namespace**: a user id, a project id, or any boundary you want isolated.

## Install and auth

```bash
# Run without installing
npx supermemory --version

# Or install globally
npm i -g supermemory

# Browser login, stores credentials in ~/.supermemory/credentials.json
supermemory login

# Or use an API key directly
supermemory login --api-key sm_xxx

# Show who you are, the key prefix and whether the key is scoped
supermemory whoami

supermemory logout
```

Commands that call the API need a login or `SUPERMEMORY_API_KEY`. `help`, `local` and `plugin` do not.

## Configuration

Three scopes. `init` writes them, `config set` edits them.

| Scope | File | Holds |
| --- | --- | --- |
| `project` | `~/.supermemory/projects/<encoded-cwd>/config.json` (per machine, never in git) | `namespace`, optional `apiKey` |
| `team` | `<git root>/.supermemory/config.json` (commit it) | `namespace` |
| `global` | `~/.supermemory/config.json` | `output`, `apiUrl`, `consoleUrl`, `telemetry` |

Namespace resolution order: `--namespace` flag, `SUPERMEMORY_NAMESPACE`, team config, project config. With none of these, content commands stop with "Namespace required".

```bash
# Interactive wizard
supermemory init

# Non-interactive
supermemory init --scope project --namespace meetings-bot
supermemory init --scope team --namespace my-project
supermemory init --scope global --output json

# Show resolved config and where each value came from
supermemory config

# Set a value; keys: apiKey, namespace, output, apiUrl, consoleUrl, telemetry
supermemory config set namespace my-project --scope project
supermemory config set output json --scope global
supermemory config set telemetry off --scope global

# Read one resolved value; keys: apiKey, namespace, apiUrl, consoleUrl, output
supermemory config get namespace
```

### Environment variables

| Variable | Effect |
| --- | --- |
| `SUPERMEMORY_API_KEY` | API key. Overrides stored credentials and prints a warning when they differ. |
| `SUPERMEMORY_NAMESPACE` | Default namespace. Renamed from `SUPERMEMORY_TAG`. |
| `SUPERMEMORY_API_URL` | API base URL, for self-hosted servers. |
| `SUPERMEMORY_CONSOLE_URL` | Console URL used by `open` and login. |
| `DO_NOT_TRACK=1` | Disables anonymous usage telemetry (command name, outcome, versions, OS; never content or keys). |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Export CLI traces to an OpenTelemetry collector. |

### Global flags

| Flag | Effect |
| --- | --- |
| `--namespace <name>` | Override the namespace for this command |
| `--json` | Force JSON output |
| `--help` | Help for any command; `--help --json` gives a machine-readable schema |

Output is JSON automatically when stdout is not a terminal, so pipes and agents get JSON without `--json`. `supermemory config set output human --scope global` forces text.

## Core commands

### `add`

Ingest text, a file or a URL into a namespace. The argument is detected: `http://` or `https://` is a URL, an existing path is uploaded as a file, anything else is text.

```bash
supermemory add "User prefers TypeScript over JavaScript"
supermemory add ./meeting-notes.pdf --namespace meetings
supermemory add https://docs.example.com/api --namespace docs

# From stdin
cat notes.txt | supermemory add --stdin --namespace notes

# Metadata and a stable id; repeating the id updates the same document
supermemory add "Design doc v2" --metadata '{"category":"engineering"}' --id design-doc

# Memories ready right away instead of batched in the background
supermemory add "Alex moved to Berlin" --dreaming instant

# Index for retrieval only, no memory extraction
supermemory add ./handbook.md --task-type superrag

# Batch: JSON array on stdin, items {content, id?, metadata?, namespace?}
echo '[{"content":"first","id":"d1"},{"content":"second"}]' | supermemory add --batch
```

Flags: `--namespace`, `--stdin`, `--metadata <json>`, `--id <string>`, `--dreaming dynamic|instant` (default `dynamic`), `--task-type memory|superrag` (default `memory`), `--batch`.

Processing continues after the command returns. Check it with `docs status <id>`.

### `search`

```bash
supermemory search "authentication patterns"
supermemory search "auth" --namespace api --limit 5 --rerank order
supermemory search "how do we handle auth" --rewrite

# Modes: hybrid (default, memories + chunks), memories, chunks
supermemory search "user prefs" --mode memories
supermemory search "refund policy" --mode chunks --include documents

# Typed filter
supermemory search "design" --filter '{"field":"category","operator":"eq","value":"engineering"}'
supermemory search "design" --filter '{"operator":"and","operands":[{"field":"source","operator":"eq","value":"slack"},{"field":"priority","operator":"gte","value":3}]}'

supermemory search "preferences" --threshold 0.5
```

Flags: `--namespace`, `--limit` (default 10), `--threshold` 0 to 1 (default 0.3), `--rerank none|order|aggregate` (default `none`), `--rewrite`, `--mode hybrid|memories|chunks` (default `hybrid`), `--include documents,related,forgotten`, `--filter <json>`.

### `profile`

The maintained profile for a namespace: `static` facts, `dynamic` recent context, and named `buckets`. No query argument; use `search` for ranked results.

```bash
supermemory profile
supermemory profile --namespace user_alex

# Build the profile from matching memories only
supermemory profile --filter '{"field":"region","operator":"eq","value":"us-west"}'
```

### `forget`

```bash
# By memory id, comma-separated for several
supermemory forget mem_abc123 --namespace user_alex
supermemory forget mem_1,mem_2

# By meaning: preview first, then run without --dry-run
supermemory forget --query "old office address" --dry-run
supermemory forget --query "old office address"
```

Flags: `--namespace`, `--query <text>`, `--dry-run`. There is no `--reason` and no direct memory write; memories come from documents, so change the source document to change a memory.

## Documents

```bash
supermemory docs list --namespace default --limit 20 --page 2
supermemory docs get doc_abc123
supermemory docs status doc_abc123        # pipeline stage: queued, extracting, chunking, embedding, indexing, done, failed
supermemory docs chunks doc_abc123
supermemory docs delete doc_abc123 --yes  # comma-separated ids for several
```

Subcommands: `list`, `get`, `delete`, `chunks`, `status`. Flags: `--namespace`, `--limit` (default 20), `--page`, `--yes`.

## Namespaces

Renamed from `tags`. A namespace is created the first time you add content with its name; there is no `create` subcommand.

```bash
supermemory namespaces list
supermemory namespaces info user_alex

# Supporting context guides memory extraction in this namespace
supermemory namespaces context user_alex --set "Paying customer on the Pro plan"
supermemory namespaces context user_alex --clear

# Delete, or move everything into another namespace first (replaces tags merge)
supermemory namespaces delete old-project --yes
supermemory namespaces delete old-project --move-to new-project --yes
```

Subcommands: `list`, `info`, `delete`, `context`. Flags: `--set <text>`, `--clear`, `--move-to <namespace>`, `--yes`.

## Connectors

Providers: `google-drive`, `notion`, `onedrive`, `gmail`, `github`, `web-crawler`, `s3`, `granola`. OAuth providers open the browser; `web-crawler` and `s3` take `--config` and start syncing at once.

```bash
supermemory connectors list
supermemory connectors list --namespace docs

supermemory connectors connect google-drive --namespace docs
supermemory connectors connect notion --namespace notes --no-browser
supermemory connectors connect web-crawler --namespace docs --config '{"startUrl":"https://docs.example.com"}'
supermemory connectors connect github --namespace code --document-limit 500

supermemory connectors sync conn_abc123
supermemory connectors history conn_abc123
supermemory connectors picker conn_abc123        # choose what the connector syncs
supermemory connectors disconnect conn_abc123 --delete-documents --yes
```

Subcommands: `list`, `connect`, `sync`, `history`, `picker`, `disconnect`. Flags: `--namespace`, `--document-limit <n>`, `--config <json>`, `--no-browser`, `--delete-documents`, `--yes`.

## API keys

```bash
supermemory keys list
supermemory keys create --name my-agent --permission write

# Scoped key: one or more namespaces, expires in 30 days
supermemory keys create --name bot-key --permission read --namespace user_123 --expires 30

supermemory keys revoke key_abc123 --yes
supermemory keys toggle key_abc123
```

Subcommands: `list`, `create`, `revoke`, `toggle`. Flags: `--name` (required for create), `--permission read|write` (default `write`), `--namespace <a,b>`, `--expires <days>`, `--yes`.

### Scoped keys

With a key scoped to a namespace, only `search`, `add`, `forget`, `profile` and `whoami` are available, and the namespace is set from the key. `help` shows only those commands.

## Team

```bash
supermemory team list
supermemory team invite user@example.com --role admin
supermemory team role member_123 admin
supermemory team remove member_123 --yes
supermemory team invitations
```

Subcommands: `list`, `invite`, `remove`, `role`, `invitations`. Flags: `--role admin|member` (default `member`), `--yes`.

## Status, logs, billing

```bash
supermemory status --period 7d            # 24h, 7d, 30d, all (default all)

supermemory logs                          # last 24h
supermemory logs --period 7d --status error --type search --limit 50 --page 2
supermemory logs get req_abc123

supermemory billing                        # plan and usage summary
supermemory billing usage
supermemory billing invoices
```

## Coding-agent plugins

`plugin` installs the Supermemory integration into coding agents on this machine and connects them with browser OAuth. `plugins` manages API-key based connections from the console side.

```bash
# Detect installed agents and pick interactively
supermemory plugin

# Non-interactive, targets: claude, cursor, opencode, codex
supermemory plugin --all
supermemory plugin --only claude,cursor --dry-run
supermemory plugin --all --no-auth                # install only, connect later
supermemory plugin --only codex --force           # run even if the app is not detected

supermemory plugin login --all
supermemory plugin login --only codex --no-browser
supermemory plugin uninstall --only cursor --dry-run
supermemory plugin uninstall --all
```

`plugin` flags: `--all`, `--only <list>`, `--dry-run`, `--force`, `--no-auth`. `plugin login`: `--all`, `--only`, `--no-browser`. `plugin uninstall`: `--all`, `--only`, `--dry-run`.

```bash
# Key-based plugin connections; ids: claude-code, cursor, opencode, openclaw
supermemory plugins list
supermemory plugins connect claude-code --auto-configure
supermemory plugins status
supermemory plugins revoke claude-code --yes
```

## `setup` and `migrate`

Both detect the project, build a prompt, and launch a coding agent with it. Agents: `claude-code`, `codex`, `cline`, `aider`, `opencode`, `goose`, `amp`.

```bash
# Integrate Supermemory into this project
supermemory setup
supermemory setup --prompt                 # print the prompt instead

# Move this project from the v3/v4 API to v5
supermemory migrate
supermemory migrate --agent codex
supermemory migrate --prompt
```

`migrate` scans for legacy packages and names (`containerTag`, `customId`, `/v3/`, `/v4/`) and exits early when there is nothing to migrate.

## Local self-hosted server

Installs and runs the free single-binary `supermemory-server` on macOS or Linux. Not available on Windows yet; download the `.exe` from GitHub Releases or use Docker instead.

```bash
supermemory local                          # same as local start
supermemory local install
supermemory local install --version 0.0.9
supermemory local start --port 9000
supermemory local start --no-install
supermemory local start -- --log-level debug   # args after -- go to the server
supermemory local upgrade
supermemory local status
supermemory local status --url http://localhost:9000
supermemory local version
supermemory local path
supermemory local env
```

Subcommands: `start`, `install`, `upgrade`, `status`, `version`, `path`, `env`. Point the CLI at it with `SUPERMEMORY_API_URL=http://localhost:6767`.

## `open` and `help`

```bash
supermemory open                           # console
supermemory open graph                     # pages: console, billing, graph, connectors, settings, docs, keys, team, agents
supermemory open billing --no-browser      # print the URL

supermemory help
supermemory help --json                    # full command schema for agents
supermemory search --help --json
```

## Scripting

```bash
export SUPERMEMORY_API_KEY=sm_xxx
export SUPERMEMORY_NAMESPACE=api-docs

# Pipes are JSON by default
git log --oneline -20 | supermemory add --stdin --id git-history
supermemory search "auth" | jq '.results[] | .memory // .chunk'
supermemory profile | jq '.profile.static[].memory'

# Ingest on deploy with a stable id so the doc updates in place
supermemory add ./docs/api-reference.md --id api-ref-latest

# Wait for processing
until supermemory docs status "$ID" | jq -e '.status == "done"' >/dev/null; do sleep 2; done
```

### Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Error, including validation and API errors |
| 2 | Authentication required |
| 3 | Permission denied |
| 4 | Not found |

Errors print to stderr as `✗ message` with a `hint:` line when there is a next step, so stdout stays clean JSON.

## Renamed or removed since 1.x

| Before | Now |
| --- | --- |
| `npm i -g @supermemory/cli` | `npm i -g supermemory` or `npx supermemory` |
| `--tag`, `SUPERMEMORY_TAG` | `--namespace`, `SUPERMEMORY_NAMESPACE` |
| `tags list|info|context|delete` | `namespaces list|info|context|delete` |
| `tags create` | add content with the new namespace |
| `tags merge a --into b` | `namespaces delete a --move-to b` |
| `remember`, `update` | removed; add or update the source document |
| `forget --content`, `--reason` | `forget --query` with `--dry-run`; no reason field |
| `search --mode documents`, `--rerank` (boolean) | `--mode chunks`, `--rerank order|aggregate` |
| `search --filter '{"AND":[{"key":..}]}'` | typed filter `{"operator":"and","operands":[{"field":..,"operator":"eq","value":..}]}` |
| `profile <namespace> --query` | `profile --namespace <name>`; use `search` for queries |
| `connectors resources` | `connectors picker` |
