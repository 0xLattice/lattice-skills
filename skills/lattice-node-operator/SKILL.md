---
name: lattice-node-operator
description: Install, configure, run, upgrade and back up a lattice-node host with the `lattice` operator CLI — init, up/down/restart, status, logs, `lattice config`, lattice.json and config.d fragments, peers and NAT, public read/submit endpoints, remote operator tokens, systemd/launchd, wipes after an incompatible release. Use when a user wants to run a Lattice node, change its configuration, expose a node publicly, set up systemd, upgrade binaries, or back up and restore node data.
---

# Running a lattice-node host

One `lattice-node` process hosts Nexus plus every child chain in its
configuration. It has one identity, one overlay port, one data directory and
one operator API. The `lattice` CLI is how operators drive it. For the
concepts, see the `lattice-concepts` skill.

The operator documentation at https://lattice.build/docs/run/getting-started/
is authoritative. Before you rely on a flag, check
`lattice <subcommand> --help` on the user's build, and trust the machine
over this text when they disagree.

## Install

A release, on macOS (Apple silicon) or Linux (x86-64):

```bash
curl -fsSL https://lattice.build/install.sh | sh    # checks the SHA-256; installs to ~/.lattice/bin
```

Or from source (Swift 6.1+, and the SQLite development headers on Linux):

```bash
git clone https://github.com/0xLattice/lattice-node.git && cd lattice-node
swift build -c release
for b in lattice lattice-node lattice-mining-coordinator lattice-miner; do
  sudo install -m 0755 .build/release/$b /usr/local/bin/$b
done
```

**Install all four binaries from the same release, in the same directory.**
`lattice` looks for `lattice-node` next to itself (falling back to
`/usr/local/bin`). Mixed versions are unsupported.

## First run

```bash
lattice init --root node [--peer <pubkey>@<host>:4001 ...]
lattice up --root node          # runs in the FOREGROUND and logs to the terminal
# from another shell:
lattice status --root node      # Nexus: active height=… best=… tip=… mempool=…
lattice identity --root node    # the peer string to give other operators
lattice down --root node        # or Ctrl-C
```

`lattice up` does not return and does not restart the node after a failure.
To keep a node running, put it under a process manager (below). Every
subcommand takes `--root <dir>`, which defaults to the current directory.

## CLI reference

| Command | What it does |
|---|---|
| `init [--peer P]...` | Writes `lattice.json` (`listen` 4001, `rpc` 8080) and creates `identity/Nexus.key` (0600). Refuses to overwrite a configuration. |
| `up` | Runs the node in the foreground. |
| `status` | One line per hosted chain: phase, height, best header height, tip, and what it waits on. Also chains configured but not running, settings waiting for a restart, and inert settings. |
| `logs [--tail N]` | The running node's recent log lines. `--miner` reads the local mining loop's. |
| `restart` | Restarts the node and waits until it answers. Refused while the configuration would not start the node. **Needs a process manager to start it again.** |
| `down` | Stops the node cleanly (exit 0). |
| `wipe` | **Destructive, no prompt.** Deletes chain storage. Keeps the identity, configuration, specs and tokens. A running node wipes as it restarts. |
| `identity` | Prints the node's public key and peer string. |
| `config get\|set\|unset`, `apply -f F --fragment N` | Read and change the configuration (below). |
| `child create\|host\|unhost\|commit` | See `lattice-child-chains`. |
| `mine start\|stop\|status\|run` | See `lattice-mining`. |
| `key generate --out F`, `tx send\|deposit\|receipt\|withdraw` | See `lattice-transactions`. |
| `pins list\|add <cid> [--fetch]\|remove <cid>` | Content the operator keeps. A pin holds that one Volume, not what it links to. |
| `tokens list\|create <name>\|revoke <name>` | Tokens for the remote operator listener. |
| `emit-systemd`, `emit-launchd` | Print service definitions for this root. |

Most commands ask the running node over its operator API. `init`,
`identity`, `key`, `up`, `mine stop`, `emit-systemd` and `emit-launchd`
work on the local machine only. `--node <url> --token-file <file>` in place
of `--root` operates a node on another machine through its remote operator
listener.

## Layout

```text
<root>/
  lattice.json             configuration
  config.d/                configuration fragments
  identity/Nexus.key       the node's identity (not a wallet). 0600. Back it up separately.
  specs/                   the specs of chains created here, and their policy modules
  operator-tokens.json     remote operator tokens, when configured
  miner.json               the miner's settings, when it mines here
  run/  log/               the miner's pidfile and log, when it mines here
  chains/Nexus/            the data directory; this is what `wipe` deletes
    state.db               facts for every chain
    volumes/               one content store per hosted chain
    volume-bundles.db  peers.json  serving.json  storage.lock  .cookie
```

The node writes a new `.cookie` at every start, so clients must read it
again. The node itself writes no log file and no pidfile: its log goes to
standard error, where the process manager keeps it.

## Configuration

`lattice.json` plus the fragments in `config.d/`, merged. Change it with the
CLI rather than by hand:

```bash
lattice config get [a.b.c]
lattice config set minRelayFee=5
lattice config unset minRelayFee
lattice status          # shows "waiting for a restart: …"
lattice restart
```

**A running node applies `hostedChains` and child-chain `peers` at once.
Every other key waits for the next start.** A change that does not check is
refused and nothing is written.

`references/lattice-json.md` has the keys used most and the rules that
catch people out. Every key is at
https://lattice.build/docs/run/configuration/.

## Network surfaces

| Surface | Default | Exposure |
|---|---|---|
| Overlay (`listen`) | 4001/tcp | Public, to accept inbound peers. One port for every chain. |
| Browser peers (`webSocket`) | 4002/tcp and udp | Public, to accept browsers. `0` turns it off. |
| Operator API (`rpc`) | 8080/tcp | **Never.** Loopback only; the node refuses other addresses. |
| Remote operator listener (`operatorListen`) | off | A private network only, or behind a TLS proxy: it is plain HTTP. |
| Public reads (`publicRead`) | off | Public, once configured. |

- **Operator API auth**: every route except `/health` needs
  `curl --user "$(cat <root>/chains/Nexus/.cookie)"`. A request with a
  browser `Origin` gets 403 unless that origin is in `rpcAllowedOrigins`,
  and it still needs the cookie.
- **NAT**: a node behind a home router connects out, and reaches other such
  nodes through peers that relay, with nothing configured. Set
  `externalAddress` (a host, no port) when the address peers observe is not
  dialable, as behind a proxy.
- **No TLS in the node.** Put a TLS-terminating proxy in front of public
  reads. The proxy must allow no more than the listener does: never
  `/status`, `/metrics`, `/core/snapshot` or a POST route other than public
  submit.

## Public reads, explorer listing, public submit

1. `lattice config set publicRead=8081`, and put HTTPS in front of it.
2. `lattice config set publicReadURL=https://reads.example.org`. One URL
   covers every chain the host serves, selected with `?chainPath=`.
3. Optionally `lattice config set publicSubmit=true` (needs `publicRead`).
   A public submit is not written to the node's journal.
4. `lattice restart`.
5. Verify from the parent's side:
   `curl -s 'https://<parent-reads>/api/chain/endpoints?chainPath=Nexus/Alpha'`.
   An explorer walks this route one level at a time, so no permission is
   needed.

The node's rate limits are listener-wide (`publicReadMaxRate`, 200 requests
a second; `publicSubmitRate`, 10). Put per-client limits in the proxy.
`publicReadRate` and `publicReadExpensiveRate` are accepted and ignored.

## Remote control

```bash
# Beside the node, once:
lattice config set operatorListen=10.0.0.5:8090 operatorTokens=operator-tokens.json --root R
lattice restart --root R
lattice tokens create ops --role admin --out ops.token --root R
# Anywhere on that network:
lattice status --node http://10.0.0.5:8090 --token-file ops.token
```

`--role read` allows every GET; `--route "METHOD /path"` (repeatable) allows
only those routes. The secret is shown once.

## Process managers

```bash
lattice emit-systemd --root /var/lib/lattice
```

prints two units: `lattice-tree.service`, and `lattice-mine.service` when
the root holds miner settings. Save each as its own file under
`/etc/systemd/system/`, then
`systemctl daemon-reload && systemctl enable --now lattice-tree lattice-mine`.

- The node's unit runs `lattice up --root <root>` with `Restart=on-failure`.
  Its log is the journal: `journalctl -u lattice-tree`.
- `systemctl reload lattice-tree` re-reads the configuration.
- The lattice-node repository also ships ready-made units in `deploy/`,
  named `lattice-node.service` and `lattice-mining-coordinator.service`. Use
  one set or the other, and use that set's unit names.
- macOS: `lattice emit-launchd --root ~/lattice` prints a launchd job. Its
  output goes to `<root>/log/lattice-node.log`.

Exit codes a manager acts on: 0 stopped on request (leave it stopped), 75
restart requested (start it again), 78 the configuration does not check
(leave it stopped until fixed; the reason is on standard error).

## Upgrades

Install all four binaries from the same release, then restart the node and
the miner.

Releases do not migrate stored data. When a release changes the storage
schema, the node refuses to start with `The node store is incompatible
(…); stop the process, wipe its complete configured storage directory, and
restart.` Then:

```bash
systemctl stop lattice-mine lattice-tree
lattice wipe --root /var/lib/lattice    # keeps identity, configuration, specs and tokens
# install all four binaries from the same release
systemctl start lattice-tree lattice-mine
```

Nexus's genesis is recreated from the release. Child chains are synced from
their peers, or mined again from their specs.

**Before a wipe, confirm with the user and check what is at `--root`.** It
deletes all chain data.

## Backups

Back up only while the node is **stopped**. `state.db` and `volumes/` are
one recovery unit: never combine them from different backups, and never
repair them by hand.

```bash
systemctl stop lattice-mine lattice-tree
tar -C /var/lib/lattice -czf lattice-backup.tgz lattice.json config.d identity specs chains/Nexus
systemctl start lattice-tree lattice-mine
```

Back up `identity/Nexus.key` separately as well. Wallet key files live
wherever `--out` put them: back them up separately too.

## Storage retention

| Setting | Values |
|---|---|
| `blockBodies` | `all` (default) keeps every block body; `none` keeps only each chain's tip and genesis. |
| `state` | `all` (default); `recent` keeps each chain's current state and what its last `stateWindow` blocks (64) replaced; `none` keeps only what the blocks at the tip read and created. |

These change what the node stores and serves, never which tip it picks.
**Run `blockBodies: all` on a node that mines a chain others should follow,
and on a node an explorer reads**: if a chain's only miner runs `none`,
nobody else can sync that chain. The database files do not shrink when rows
are deleted; to reclaim space, sync the node again from an empty data
directory.

Unhosting a chain deletes its data.

## Safety checklist

- Back up `identity/Nexus.key`, `lattice.json`, `config.d/`, `specs/` and
  wallet keys.
- Keep the operator API on loopback. Serve the public only from
  `publicRead`, or a proxy that allows no more than it does.
- Keep the remote operator listener on a private network.
- Start the node with `lattice up`. `lattice-node` started with flags and
  no `--identity-key` keeps its identity inside the data directory, which a
  wipe deletes.
- Give the miner a recipient for every chain that should pay out.

For error messages and stuck nodes, use the `lattice-troubleshooting` skill.
