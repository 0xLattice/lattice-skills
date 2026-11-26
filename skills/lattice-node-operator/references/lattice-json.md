# Node configuration: quick reference

The authoritative list of every key, its default and its rules is the
node's [Configuration](https://lattice.build/docs/run/configuration/) page.
This file keeps the keys an operator touches most and the rules that catch
people out. When the two disagree, the page is right; on a user's machine,
`lattice config get` shows what the node actually reads.

## Where configuration lives

| File | Holds | Read by |
|---|---|---|
| `<root>/lattice.json` | The node's configuration. `lattice init` writes `listen` and `rpc`. | the node |
| `<root>/config.d/<name>.json` | Fragments layered over `lattice.json` in name order. `lattice config set` writes `config.d/operator.json`. | the node |
| `<root>/miner.json` | The miner's settings ([Mining](https://lattice.build/docs/run/mining/)). | `lattice mine` only |

An unknown key is refused. A configuration that does not check stops the
node at start with the reason (exit 78).

## What applies when

- A running node applies `hostedChains` and child-chain `peers` at once.
  **Every other key waits for the next start** (`lattice restart`);
  `lattice status` lists what waits.
- A change that does not check is refused and nothing is written.
- A file edited by hand is read at the next start, or on `SIGHUP`.
- Host and unhost chains with `lattice child host` and `lattice child
  unhost` rather than editing `hostedChains`.

## Keys used most

| Key | Default | Meaning |
|---|---|---|
| `listen` | 4001 | The overlay port, TCP. One port for every hosted chain. |
| `rpc` | 8080 | The operator API port. Loopback only. |
| `webSocket` | 4002 if it can bind | The port browsers reach the node on. `0` turns it off. |
| `externalAddress` | observed | The host other nodes can reach this one at. A host, no port. |
| `peers` | built-in seeds | Peers by chain path, each `<public-key>@host:port`. |
| `hostedChains` | none | Child chain paths this node hosts. Nexus is always hosted. |
| `publicRead` | off | Port of the public read listener, on all interfaces. |
| `publicReadURL` | none | This host's public read URL, declared for every chain it hosts. |
| `publicSubmit` | false | Also accept `POST /transactions` on the read listener. Needs `publicRead`. |
| `publicReadMaxRate` | 200 | Requests per second the read listener accepts, all clients together. `0` removes the limit. |
| `publicSubmitRate` | 10 | Requests per second for public submits. |
| `minRelayFee` | 1 | The smallest fee this node admits to its pool. Node policy, not consensus. |
| `blockBodies` | `all` | `all`, or `none` to keep only each chain's tip and genesis. |
| `state` | `all` | `all`, `recent` or `none`. |
| `stateWindow` | 64 | With `state: recent`, how many blocks of replaced state each chain keeps. |
| `rpcAllowedOrigins` | none | Browser origins allowed on the operator API. The cookie is still required. |
| `operatorListen`, `operatorTokens` | off | The remote operator listener's `address:port` and its token file. |

## Rules that catch people out

- **Ports** must be nonzero and unique. Only `webSocket` may be `0`.
- **Nexus's `peers` list replaces the built-in seeds.** An empty list means
  connect to no one. A child chain's list is extra seeds for that chain.
  An array instead of an object is Nexus's list.
- **`publicReadRate` and `publicReadExpensiveRate` are accepted and
  ignored.** The node's limits are listener-wide; put per-client limits in
  a proxy in front.
- **Mining settings are not in `lattice.json`.** They are in `miner.json`.
  The node ignores a `mine` object in `lattice.json`.
- **`state: tip`** is accepted and means `recent`.
- **A setting for a chain that is not hosted** (its peers or windows) is
  not an error. `lattice status` lists it as inert.

```json
"peers": {
  "Nexus": ["<public-key>@node.example:4001"],
  "Nexus/Alpha": ["<public-key>@alpha-host.example:4001"]
}
```

A peer string is `<public-key>@host:port`, as `lattice identity` prints it.

## Environment variables

| Variable | Effect |
|---|---|
| `LATTICE_SYNC_TRACE=1` | Sync tracing on the node's standard error. Any other non-empty value is a file prefix. It is verbose: use it only while diagnosing. |
| `LATTICE_FORBID_DEFAULT_PEERS=1` | Refuse to start on the built-in seeds. Use it in tests and CI so they never join the public network. |
