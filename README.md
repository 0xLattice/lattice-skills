# Lattice skills

Agent skills for people who run, mine on, transact with, or build apps on
[lattice-node](https://github.com/adalinxx/lattice-node), the full node for
the [Lattice](https://github.com/adalinxx/Lattice) hierarchical proof-of-work
protocol.

Install these and your coding agent will know the `lattice` CLI,
`lattice.json`, the HTTP API, how fees and cross-chain exchange really work,
and the pitfalls operators and app authors hit.

| Skill | Use it for |
|---|---|
| [`lattice-concepts`](skills/lattice-concepts/SKILL.md) | How Lattice works: Nexus, child chains, merged mining, fork choice, economics, what is and isn't possible |
| [`lattice-node-operator`](skills/lattice-node-operator/SKILL.md) | Install, configure, run, expose, upgrade and back up a node |
| [`lattice-child-chains`](skills/lattice-child-chains/SKILL.md) | Launch, follow, host and publish child chains; ChainSpec and WASM policies |
| [`lattice-transactions`](skills/lattice-transactions/SKILL.md) | Keys, balances, sends, fees and nonces, parent↔child exchange, the HTTP API |
| [`lattice-mining`](skills/lattice-mining/SKILL.md) | Mining setup, reward recipients, the coordinator, GPU and remote workers |
| [`lattice-troubleshooting`](skills/lattice-troubleshooting/SKILL.md) | Error messages, stuck chains, RPC errors, transactions that don't land |
| [`lattice-app-builder`](skills/lattice-app-builder/SKILL.md) | Designing and building apps: chain layout, KV state, policies, verified reads, wallet writes |

## Install

### Claude Code

```text
/plugin marketplace add adalinxx/lattice-skills
/plugin install lattice@lattice-skills
```

Claude loads a skill automatically when your request matches its
description. You can also name one, e.g. "use lattice-troubleshooting".

### Other agents

Each skill is a standard `SKILL.md` folder. Copy the folders under
`skills/` into wherever your agent loads skills from (for Claude Code
without the plugin, that's `~/.claude/skills/`):

```bash
git clone https://github.com/adalinxx/lattice-skills
cp -R lattice-skills/skills/* ~/.claude/skills/
```

## Versions

Written against lattice-node `main` (October 2026) and Lattice 45.2.0.
Lattice has flag-day releases, so CLI flags, defaults and error strings can
change. The skills tell the agent to trust the user's installed binaries
(`lattice <cmd> --help`) over the text whenever the two disagree.

The node's own docs are the source of truth: `docs/getting-started.md`,
`docs/operator-cli.md`, `docs/rpc-api.md` and `docs/operations.md` in
lattice-node, and `docs/spec.md` in Lattice. Where the code and those docs
disagree, the skills follow the code and say so.

## Contributing

Corrections are welcome. Please cite the source file or doc section for any
behaviour you add or change. Keep each `SKILL.md` short, and put detail in
its `references/` folder.
