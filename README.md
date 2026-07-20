# ChainLaunch Skills

Agent skills for working with [ChainLaunch Pro](https://chainlaunch.dev) via its
Bun-based CLI, [`@chainlaunch/pro-cli`](https://www.npmjs.com/package/@chainlaunch/pro-cli)
(`bunx @chainlaunch/pro-cli`). Compatible with [Claude Code](https://claude.com/claude-code)
and other agents supporting the [agent skills](https://github.com/vercel-labs/skills) format.

## Install

```bash
npx skills add chainlaunch/skills
```

Install a specific skill only:

```bash
npx skills add chainlaunch/skills --skill pro-cli-debug
```

List available skills without installing:

```bash
npx skills add chainlaunch/skills --list
```

## Skills

| Skill | Description |
|-------|-------------|
| [`pro-cli-debug`](skills/pro-cli-debug/SKILL.md) | Read-only debugging of a ChainLaunch Pro instance — node logs, chaincode logs, lifecycle timelines, channel config — using the `chainlaunch` CLI. |

## About ChainLaunch

ChainLaunch is an enterprise blockchain infrastructure platform for deploying,
managing, and monitoring Hyperledger Fabric and Besu networks. Learn more at
[chainlaunch.dev](https://chainlaunch.dev).

## Contributing

Skills live under `skills/<name>/SKILL.md`. Keep skills read-only and safe by
default; anything that mutates infrastructure should say so explicitly and
require confirmation.
