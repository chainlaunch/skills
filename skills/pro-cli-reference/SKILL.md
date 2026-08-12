---
name: pro-cli-reference
description: >
  Full command and flag reference for the npm/bunx CLI (@chainlaunch/pro-cli,
  bin `chainlaunch`). Covers every command including mutations:
  login/logout/whoami, context, nodes (start/stop/restart/delete), networks
  (delete), organizations (create/delete), chaincodes (read-only), keys
  (delete), key-providers, backups, users, api-keys (create/revoke), plugins,
  settings, env vars, contexts, output-format flags.
  TRIGGER: user names `@chainlaunch/pro-cli` or `bunx chainlaunch`, or wants to
  run/script/build against the ChainLaunch REST-API CLI, incl. mutations
  (create an org, revoke an api-key, delete a node/network).
  SKIP: read-only troubleshooting of a running instance (node won't start,
  chaincode failing, "why is my network unhealthy") — use pro-cli-debug
  instead, it has the diagnostic workflows this skill doesn't.
---

# @chainlaunch/pro-cli — Full Command Reference

CLI: `chainlaunch` from npm package `@chainlaunch/pro-cli`. REST-API client
for a ChainLaunch Pro server's `/api/v1`.

```bash
bunx @chainlaunch/pro-cli@<version> --help
```

Prefer a pinned `bunx @chainlaunch/pro-cli@<version>` over `bun add -g` /
`npm install -g` — the global install's bin is also named `chainlaunch`,
which can collide with a Go `chainlaunch` binary already on PATH from a
different ChainLaunch install. `bunx` never touches PATH.

Global: every read command accepts `--json` (suppresses spinners/tables, pipes cleanly to `jq`).
Output defaults to table; a context can default to JSON — see Contexts below.

**Before running any command, confirm the CLI is authenticated:**
```bash
chainlaunch context list --json
```
If this errors or returns an empty list, there's no logged-in context yet —
walk the user through Auth below (env vars for a one-off session, or
`chainlaunch login` for a persisted context) before attempting any other
command. Don't guess at a server URL or credentials; ask if they weren't
already provided.

**Commands below marked destructive prompt for confirmation by default** —
`-f, --force` / `-y, --yes` skip that prompt. Only pass those flags when the
user has explicitly asked for the mutation; otherwise let the prompt stand.

## Auth

Resolution order: env vars → active context → legacy secrets file (`~/.chainlaunch-pro-cli/.secrets`).

```bash
chainlaunch login [url]
  --api-key <key>              # clpro_… — sent as Bearer
  -u, --username <username>    # basic auth
  -p, --password <password>
  -c, --context <name>         # name to store credentials under
  -k, --insecure                # skip TLS verification; persisted on the context
  --default-format <table|json> # skip the interactive format prompt
  --no-interactive              # for automation — omit for a real login so secrets aren't in shell history

chainlaunch logout [-c, --context <name>]
chainlaunch whoami [--json]
```

Env vars (highest priority, good for CI): `CHAINLAUNCH_API_URL`, `CHAINLAUNCH_API_KEY` /
`CHAINLAUNCH_TOKEN` (Bearer), `CHAINLAUNCH_USERNAME` + `CHAINLAUNCH_PASSWORD` (basic),
`CHAINLAUNCH_CONTEXT` (pin a context without changing the active flag),
`CHAINLAUNCH_INSECURE=1` (skip TLS verify), `CHAINLAUNCH_JSON=1|0` (force output format
for one invocation).

Prefer env vars over `--api-key`/`-p` flags for credentials — flags land in shell
history and `ps` output in plaintext.

## Contexts

Multiple servers, one CLI. Stored in `~/.chainlaunch-pro-cli/.contexts.json` (mode 0600).

```bash
chainlaunch context list [--json]
chainlaunch context use <name>              # alias: switch
chainlaunch context current [--json]        # prints just the name; script-friendly
chainlaunch context rename <old> <new>
chainlaunch context remove <name>           # local only — does NOT revoke the server-side key
chainlaunch context format [name]           # show default output format
chainlaunch context set-format <table|json> [name]
```

## Commands

### nodes (alias `n`)
```bash
chainlaunch nodes list [--platform fabric|besu] [--status <status>] [--network <id>] [--page <n>] [--limit <n>] [--json]
chainlaunch nodes show <node> [--json]        # by ID or slug
chainlaunch nodes start <node> [--json]
chainlaunch nodes stop <node> [--json]
chainlaunch nodes restart <node> [--json]
chainlaunch nodes logs <node> [-n, --tail <lines>] [-f, --follow]
chainlaunch nodes delete <node> [-f, --force] [-y, --yes]     # destructive
```

### networks (alias `net`)
```bash
chainlaunch networks list [--platform fabric|besu|fabricx] [--json]
chainlaunch networks show <platform> <id> [--json]
chainlaunch networks config <id>              # Fabric only — dumps decoded channel config as JSON
chainlaunch networks map <id> [--json]        # which nodes are joined
chainlaunch networks delete <platform> <id> [-f, --force] [-y, --yes]     # destructive
```

### organizations (alias `orgs`)
```bash
chainlaunch organizations list [--json]
chainlaunch organizations show <org> [--json]     # by ID or MSP ID
chainlaunch organizations create [-n, --name <name>] [-d, --description <description>] [--provider <id>] [--json]
chainlaunch organizations delete <org> [-f, --force] [-y, --yes]     # destructive
```

### chaincodes (alias `cc`) — read-only
```bash
chainlaunch chaincodes list [--json]
chainlaunch chaincodes show <id> [--json]
chainlaunch chaincodes logs <definitionId> [-n, --tail <lines>] [-f, --follow]
chainlaunch chaincodes timeline <definitionId> [--json]
```

### keys
```bash
chainlaunch keys list [--page <n>] [--limit <n>] [--json]
chainlaunch keys show <id> [--json] [--cert] [--public-key]   # --cert/--public-key print PEM
chainlaunch keys delete <id> [-f, --force] [-y, --yes]     # destructive — irreversible if unbacked up
```

### key-providers
```bash
chainlaunch key-providers list [--json]
chainlaunch key-providers show <id> [--json]
```

### backups
```bash
chainlaunch backups list [--json]
chainlaunch backups show <id> [--json]
chainlaunch backups targets [--json]
chainlaunch backups schedules [--json]
```

### users
```bash
chainlaunch users list [--json]
chainlaunch users show <id> [--json]
```

### api-keys
```bash
chainlaunch api-keys list [--json]
chainlaunch api-keys create [-d, --description <description>] [-r, --role <admin|operator|viewer>] [--expires-in <duration>] [--json]
chainlaunch api-keys revoke <id> [-f, --force] [-y, --yes]     # destructive — immediately invalidates the key
```

### plugins
```bash
chainlaunch plugins list [--json]
chainlaunch plugins status <name> [--json]
```

### settings
```bash
chainlaunch settings show [--json]
```

## Notes

- `-f, --force` and `-y, --yes` on destructive commands are aliases of each other — either skips the confirmation prompt.
- SSE-streamed endpoints (`nodes logs`, `chaincodes logs`) auto-reconnect and strip `data:` framing; `-f`/`--follow` streams until Ctrl-C.
- Anything the CLI lacks: every ChainLaunch Pro instance serves its own OpenAPI
  spec and Swagger UI at `<url>/swagger/index.html`. Read-only GETs can be done
  directly, e.g. `curl -H "Authorization: Bearer $CHAINLAUNCH_API_KEY" <url>/api/v1/<path>`
  — reference the env var, never paste the literal key into the command.
