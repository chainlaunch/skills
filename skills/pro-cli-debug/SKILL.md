---
name: pro-cli-debug
description: >
  Read-only debugging of a running ChainLaunch Pro instance with the Bun CLI
  (@chainlaunch/pro-cli, run via `bunx @chainlaunch/pro-cli`). Inspect node logs,
  chaincode container logs, lifecycle timelines, network/channel config, orgs, keys,
  and backups without mutating anything. Use when diagnosing a node that won't start,
  a chaincode that fails invoke/query, endorsement/policy mismatches, TLS errors, or
  "why is my network unhealthy" questions against a ChainLaunch Pro deployment.
  Triggers: "debug node", "chaincode logs", "why is the peer failing", "inspect
  channel config", "tail logs", "network map", "chainlaunch cli".
---

# ChainLaunch Pro — Read-Only Debug with the Bun CLI

CLI: `chainlaunch` from the `@chainlaunch/pro-cli` npm package. No install needed:

```bash
bunx @chainlaunch/pro-cli <command>
# or install globally
bun add -g @chainlaunch/pro-cli
chainlaunch <command>
```

**This skill is READ-ONLY.** Never run mutating commands while debugging:
no `start`/`stop`/`restart`/`delete`/`create`/`revoke`/`use`. If a fix requires
mutation, report findings and let the user decide.

## Setup / auth (once)

```bash
chainlaunch login <url> --api-key clpro_…      # or -u user -p pass
chainlaunch context list                        # verify which server is active
chainlaunch whoami                              # verify identity + role
```

Non-interactive/CI: `CHAINLAUNCH_API_URL=<url> CHAINLAUNCH_API_KEY=clpro_… chainlaunch …`
Pin a context per shell: `CHAINLAUNCH_CONTEXT=<name>`.

**Self-signed HTTPS** (k3d/dev clusters, `Login failed: self signed certificate`):

```bash
chainlaunch login https://10.30.0.121:8100 --api-key clpro_… --insecure
```

`--insecure` (`-k`) is persisted on the context, so later commands against that
server skip verification automatically. One-off alternatives: global flag
before the subcommand (`chainlaunch --insecure nodes list`) or
`CHAINLAUNCH_INSECURE=1`. Never use against servers you don't control.

Every read command accepts `--json` (suppresses spinners/tables → pipe to `jq`).
`login` also asks whether this context should default to JSON or table output;
`chainlaunch context set-format json|table [name]` changes it later, and
`CHAINLAUNCH_JSON=1` forces JSON for a single invocation regardless. If you're
scripting against this skill's output, set the context to JSON once instead of
adding `--json` to every command.

## Debug command matrix

| What | Command |
|------|---------|
| All nodes + status | `chainlaunch nodes list` (`--platform fabric\|besu`, `--status error`) |
| Node detail (by ID or slug) | `chainlaunch nodes show <node>` — check `Status`, `Error`, `Endpoint` |
| Node logs (tail) | `chainlaunch nodes logs <node> -n 200` |
| Node logs (live) | `chainlaunch nodes logs <node> -f` |
| Networks + status | `chainlaunch networks list` |
| Network detail | `chainlaunch networks show fabric\|besu\|fabricx <id>` |
| Decoded channel config | `chainlaunch networks config <id>` (Fabric; JSON to stdout) |
| Which nodes serve a network | `chainlaunch networks map <id>` |
| Chaincodes | `chainlaunch chaincodes list` |
| Chaincode + definitions | `chainlaunch chaincodes show <id>` — version, sequence, docker image, address |
| Chaincode container logs | `chainlaunch chaincodes logs <definition-id> -n 200` (`-f` to follow) |
| Chaincode lifecycle history | `chainlaunch chaincodes timeline <definition-id>` |
| Orgs / MSPs | `chainlaunch orgs list`, `chainlaunch orgs show <mspId>` |
| Key material (cert expiry etc.) | `chainlaunch keys list`, `chainlaunch keys show <id>` (`--cert` prints PEM) |
| Backup health | `chainlaunch backups list`, `chainlaunch backups schedules` |
| Server settings | `chainlaunch settings show` |

## Workflows

### Node won't start / unhealthy
1. `chainlaunch nodes list --json | jq '.[] | select(.status != "RUNNING") | {id,name,status,errorMessage}'`
2. `chainlaunch nodes show <node>` — read `Error` field.
3. `chainlaunch nodes logs <node> -n 300` — look for: TLS handshake failures
   (cert SANs / wrong CA), port bind errors, `panic`, ledger/genesis mismatches.
4. Cross-check cert expiry: `chainlaunch keys show <id> --cert | openssl x509 -noout -dates`.

### Chaincode invoke/query failing
1. `chainlaunch chaincodes show <id>` — confirm expected version + sequence and
   which definition is live; note the definition ID.
2. `chainlaunch chaincodes logs <definition-id> -n 200` — runtime errors, panics,
   connection refused to peer.
3. `chainlaunch chaincodes timeline <definition-id>` — was it installed → approved
   → committed → deployed in order? A missing commit after approve is the classic
   "endorsement policy failure" cause.
4. Endorsement mismatch → `chainlaunch networks config <net-id> | jq '..|.policies? // empty'`.

### Network / channel config questions
1. `chainlaunch networks show fabric <id>` — status + channel name (users can
   rename networks; `channelName` is the real channel ID).
2. `chainlaunch networks config <id> | jq 'keys'` then drill down — org MSPs,
   orderer endpoints, batch sizes, ACLs, capabilities.
3. `chainlaunch networks map <id>` — confirms which nodes are joined and their status.

## Notes

- Log output is normalized (SSE framing stripped) — pipe straight to `grep`/`less -R`.
  ANSI colors from Fabric are preserved.
- `nodes logs -f` and `chaincodes logs -f` stream until Ctrl-C.
- Anything the CLI lacks: every ChainLaunch Pro instance serves its own OpenAPI
  spec and Swagger UI at `<url>/swagger/index.html`. Read-only GETs can be done
  directly, e.g. `curl -H "Authorization: Bearer $CHAINLAUNCH_API_KEY" <url>/api/v1/<path>`.
