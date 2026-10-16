---
name: lattice-node-operator
description: Install, configure, run, upgrade and back up a lattice-node host with the `lattice` operator CLI — init, up/down, status, lattice.json, peers and NAT, public read/submit endpoints, systemd, flag-day wipes. Use when a user wants to run a Lattice node, edit lattice.json, expose a node publicly, set up systemd, upgrade binaries, or back up and restore node data.
---

# Running a lattice-node host

One `lattice-node` process hosts Nexus plus every child chain listed in
`lattice.json`. It has one identity, one P2P port, one store and one RPC.
The `lattice` CLI is how operators drive it. For the concepts, see the
`lattice-concepts` skill.

Written against lattice-node `main` (Oct 2026). Before you rely on a flag,
check `lattice <subcommand> --help` on the user's build.

## Install

Build from source (Swift 6.1+, macOS 15+ or Linux with SQLite dev headers):

```bash
git clone https://github.com/adalinxx/lattice-node.git && cd lattice-node
swift build -c release
for b in lattice lattice-node lattice-mining-coordinator lattice-miner; do
  sudo install -m 0755 .build/release/$b /usr/local/bin/$b
done
```

Alternatively, use a release archive and verify it first:
`shasum -a 256 -c <archive>.sha256` and
`gh attestation verify <archive> --repo adalinxx/lattice-node`.

**Install all four binaries from the same build, in the same directory.**
`lattice` looks for `lattice-node` next to itself (falling back to
`/usr/local/bin`), and mining looks for the coordinator and miner next to
`lattice-node`. Mixed versions are unsupported.

## First run

```bash
mkdir -p node && lattice init --root node [--peer <pubkey>@<host>:4001 ...]
lattice up --root node          # background; log at node/log/lattice-node.log
lattice status --root node      # Nexus: active height=… tip=… mempool=…
lattice identity --root node    # peer string to give other operators
```

Every subcommand takes `--root <dir>`, which defaults to the current
directory. `lattice key generate` is the one exception.

## CLI reference

| Command | What it does |
|---|---|
| `init [--peer P]...` | Writes `lattice.json` (`listen` 4001, `rpc` 8080, optional `peers`) and creates `identity/Nexus.key` (0600). Refuses if `lattice.json` exists. |
| `up [--foreground]` | Starts the node. If it is already running, it only reconciles **the hosted chain set**, live through `POST /hosted-chains`, and restarts as a fallback. `--foreground` is a supervisor for systemd or container PID 1 that respawns the node when it exits. |
| `down` | SIGTERM, 30 s grace, then SIGKILL. Prints nothing if the node isn't running. |
| `status` | One line per hosted chain from the local `/health`: `down`, `running, rpc unreachable`, or `<phase> height= tip= mempool=`. |
| `identity` | Prints `Nexus: <pubkey>@<host>:<listen>`. Creates the key if it is missing. |
| `wipe` | **Destructive, no prompt.** Deletes all of `<root>/chains/Nexus`. Keeps `lattice.json`, `identity/` and `specs/`. Refuses while the node is running. |
| `key generate --out F` | Creates a wallet key file. See `lattice-transactions`. |
| `child create …` | See `lattice-child-chains`. |
| `mine start\|stop\|status` | See `lattice-mining`. |
| `tx send\|deposit\|receipt\|withdraw` | See `lattice-transactions`. |
| `emit-systemd [--binary P]` | Prints the systemd units to stdout. See below. |

The CLI has no `peers`, `logs`, `restart`, `upgrade` or `backup` subcommand.

**Gotcha: `lattice up` on a running node ignores every change except the
hosted chain set.** It does not apply new ports, peers, public-read settings,
rates or retention. Apply those with `lattice down && lattice up`. Under
systemd, run `systemctl restart lattice-node` instead.

## Layout

```text
<root>/
  lattice.json            # the only config file; unknown keys are rejected
  identity/Nexus.key      # P2P identity (not a wallet). 0600. Back it up separately.
  specs/                  # immutable child specs, premine recipients, policy .wasm
  chains/Nexus/           # ENTIRE runtime store; this is what `wipe` deletes
    state.db  volumes.db  volume-bundles.db (cache)  .cookie  peers.json  storage.lock
  run/  log/              # pid files; lattice-node.log, mine.log
```

The node rewrites `.cookie` at every start, so clients must re-read it.

## lattice.json

Minimal example:

```json
{
  "listen": 4001,
  "rpc": 8080,
  "peers": ["<pubkey>@seed.example.org:4001"],
  "hostedChains": ["Nexus/Alpha"]
}
```

`references/lattice-json.md` has every key, its default and its validation
rule, including `publicRead*`, `publicSubmit`, `minRelayFee`, `blockBodies`,
`state`, `rpcAllowedOrigins` and the `mine` block. The rules that catch
people out:

- Ports must be nonzero and distinct (`listen`, `rpc`, `publicRead`, and
  `webSocket` unless it is 0).
- `hostedChains` lists every parent before its children, with no
  duplicates.
- `peers` as an array is Nexus's list and **replaces** the built-in seeds.
  `[]` means "connect to no one". If `peers` is absent, the node uses the
  built-in seeds `seed1…4.lattice.build`. An object keyed by chain path adds
  per-chain seeds. Older nodes refuse to start on the object form.

## Network surfaces

| Port | Purpose | Exposure |
|---|---|---|
| 4001/tcp (`listen`) | Ivy P2P for the whole tree | public if you accept inbound |
| 4002/tcp+udp (`webSocket`) | browser peers (WS) + STUN, relay for NAT'd peers | public; 0 disables |
| 8080/tcp (`rpc`) | operator RPC | **loopback only**, enforced by the daemon |
| `publicRead` | GET-only reads (+ `POST /transactions` if `publicSubmit`) | 0.0.0.0, rate-limited |

- **Operator RPC auth**: every route except `/health` needs
  `curl --user "$(cat <root>/chains/Nexus/.cookie)"`. Requests with a
  browser `Origin` get 403 unless that origin is in `rpcAllowedOrigins`, and
  they still need the cookie.
- **NAT**: outbound connections work without any configuration, and
  hole-punching via relaying peers handles NAT-to-NAT links (but not
  symmetric NAT). Behind NAT or a proxy, set `externalAddress` (a host only,
  no port) so the node doesn't announce an unreachable address.
- **No TLS in the node.** Put a TLS-terminating proxy in front of public
  reads. Never expose `/status`, `/metrics`, `/core/snapshot` or operator
  POST routes.

## Public reads, explorer listing, public submit

1. Set `"publicRead": 8081` and put HTTPS in front of it.
2. Set `"publicReadURL": "https://reads.example.org"` (absolute, no query,
   no credentials). One URL covers every hosted level via `?chainPath=`.
3. Optionally set `"publicSubmit": true` (requires `publicRead`; default
   budget 10 submits/s). Public submits are never journaled.
4. Run `lattice down && lattice up`.
5. Verify from a parent host:
   `curl -s 'https://<parent-reads>/api/chain/endpoints?chainPath=Nexus/Alpha'`.
   Explorers walk this endpoint one level at a time, so no permission is
   needed.

Behind a proxy that collapses client addresses (common for L4 proxies), set
`publicReadRate` and `publicReadExpensiveRate` to 0 and keep
`publicReadMaxRate`. Otherwise every client shares one bucket and gets 429s.

## systemd

lattice-node ships with `deploy/lattice-node.service` (`User=lattice`,
`ExecStart=/usr/local/bin/lattice up --foreground --root /var/lib/lattice`,
`LimitNOFILE=65536`). Use it as follows:

```bash
sudo useradd --system --home /var/lib/lattice lattice   # if absent
sudo install -d -o lattice -g lattice -m 0750 /var/lib/lattice   # useradd --system doesn't create it
sudo install -m 0644 deploy/lattice-node.service /etc/systemd/system/
sudo -u lattice lattice init --root /var/lib/lattice --peer '<key>@<host>:4001'
sudo systemctl daemon-reload && sudo systemctl enable --now lattice-node
```

`lattice emit-systemd` prints **two** units, `lattice-tree.service` and (if
`mine` is configured) `lattice-mine.service`, concatenated on stdout. Split
them into two files under `/etc/systemd/system/`. Do not redirect the output
into one file as the docs suggest, because systemd won't load it. The
generated units have no `User=` or `LimitNOFILE`, so add both.

Under systemd, **`lattice down` is undone by the supervisor**. Run
`systemctl stop lattice-node` instead.

## Upgrades and flag days

There are no migrations and no mixed-version operation.

- **Compatible release**: stop the miner and the node, replace all four
  binaries from one release, and start them again.
- **Incompatible release (flag day)**: the node refuses to start with
  `The node store is incompatible (…); stop the process, wipe its complete
  configured storage directory, and restart.` Run:

  ```bash
  systemctl stop lattice-mining-coordinator lattice-node   # or: lattice mine stop; lattice down
  lattice wipe --root /var/lib/lattice                     # keeps identity, lattice.json, specs
  # install all four binaries from the same release
  systemctl start lattice-node lattice-mining-coordinator
  ```

  Nexus genesis is rebuilt deterministically. Child chains resync from peers,
  or are re-mined from `specs/`.

Before a wipe, confirm with the user and check what's at `--root`. It
deletes all chain data.

## Backups

Back up only while the node is **stopped**. `state.db` and `volumes.db` are
one unit: never combine files from different snapshots, and never repair
them by hand.

```bash
systemctl stop lattice-mining-coordinator lattice-node
tar -C /var/lib/lattice -czf lattice-backup.tgz lattice.json identity specs chains/Nexus
systemctl start lattice-node lattice-mining-coordinator
```

Wallet key files live wherever `--out` put them, so back them up separately.

## Storage retention

`blockBodies: none` keeps only the tip and genesis bodies. `state: tip|none`
serves state only at the tip, or only what the last block touched. These
settings change what the node stores and serves, never which tip it picks.
**If a chain's only miner runs `blockBodies: none`, nobody else can sync
that chain**, so explorer and public nodes should keep `all`.
`volumes.db` does not shrink; freed space is reused.

## Safety checklist

- Back up `identity/Nexus.key`, `lattice.json`, `specs/` and wallet keys.
- Don't run `lattice-node` directly without `--identity-key`. Its default
  identity location is inside the wipeable store.
- Keep the RPC on loopback. Expose public reads only through `publicRead` or
  an allowlisting proxy (`deploy/read-replica/` in the lattice-node repo).
- Configure a mining recipient for every chain that should pay out.

For error messages and stuck nodes, use the `lattice-troubleshooting` skill.
