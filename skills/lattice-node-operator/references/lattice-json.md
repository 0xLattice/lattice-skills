# lattice.json — every key

`lattice up` turns each key into a `lattice-node` flag. Any key not listed
here is rejected with `unknown lattice.json key(s): …`.

| Key | Type | Node flag | Default | Validation / notes |
|---|---|---|---|---|
| `listen` | port | `--listen-port` | 4001 (written by `init`) | nonzero, unique |
| `rpc` | port | `--rpc-port` | 8080 (written by `init`) | nonzero, unique; always loopback |
| `webSocket` | port | `--websocket-port` | the node tries 4002 and continues without it if the bind fails | 0 = off; if set explicitly, a failed bind is fatal |
| `peers` | array, or object keyed by chain path | `--peer` / `--no-default-peers` / `--chain-peer P=peer` | built-in Nexus seeds | every object key must be a hosted chain |
| `externalAddress` | host (no port) | `--external-address` | none | set when the observed address isn't dialable |
| `publicRead` | port | `--public-read-port` | off | nonzero, unique; binds 0.0.0.0 |
| `publicReadRate` | req/s | `--public-read-rate` | 25 | per client; 0 = no per-client cap |
| `publicReadExpensiveRate` | req/s | `--public-read-expensive-rate` | 1 | per client, for expensive routes |
| `publicReadMaxRate` | req/s | `--public-read-max-rate` | 200 | for the whole listener |
| `publicReadFetches` | int | `--public-read-fetches` | 4 | concurrent peer fetches on a miss; 0 = serve only content the node holds |
| `publicReadURL` | URL | `--public-read-url` | none | absolute http(s), no credentials, query or fragment |
| `publicSubmit` | bool | `--public-submit` | false | requires `publicRead` |
| `publicSubmitRate` | req/s | `--public-submit-rate` | 10 | for the whole listener |
| `minRelayFee` | uint64 | `--min-relay-fee` | 0 | relay policy, not consensus |
| `servingMaxConcurrent` | int | `--serving-max-concurrent` | 64 | content-serving concurrency for peers |
| `servingMaxInFlightVolumeBytes` | int | `--serving-max-in-flight-volume-bytes` | Ivy default | |
| `blockBodies` | `"all"` / `"none"` | `--block-bodies` | all | `none` prunes at the next start |
| `state` | `"all"` / `"tip"` / `"none"` | `--state` | all | |
| `announceLifetime` | seconds | `--announce-lifetime` | 1200 | 1…86400; DHT provider record lifetime |
| `hostedChains` | [path] | `--host-chain P[=spec]` | none | Nexus-rooted, each parent before its children, no duplicates |
| `rpcAllowedOrigins` | [origin] | `--rpc-allowed-origin` | none (browsers refused) | e.g. `chrome-extension://<id>`; the cookie is still required |
| `mine` | object | used only by `lattice mine` | none | see below |

## `peers`

```jsonc
"peers": ["<pubkey>@a.example.org:4001", "<pubkey>@b.example.org:4001"]   // Nexus list; replaces the seeds
"peers": []                                                                // connect to no one (inbound only)
"peers": { "Nexus": ["…"], "Nexus/Alpha": ["<pubkey>@c.example.org:4001"] } // per-chain seeds
```

A peer string is `<pubkey>@host:port`, the output of `lattice identity`.
When a chain has fewer than 8 peers (`--peer-target`), or has stalled, the
node dials remembered peers (`peers.json`, advisory, safe to delete). For
child chains it also searches the DHT rendezvous. Below 2 peers
(`--peers-before-seeds`) it dials the seeds.

## `mine`

```json
"mine": {
  "worker": "cpu",
  "workers": 2,
  "batchSize": 2000000000,
  "recipients": { "Nexus": "<address>", "Nexus/Alpha": "<address>" },
  "minWork": { "Nexus": "2^32" },
  "minBlockIntervalSeconds": 10,
  "templateTimeoutSeconds": 60,
  "roundDeadlineMultiplier": 10
}
```

| Key | Default | Meaning |
|---|---|---|
| `worker` | `cpu` | `cpu` = the bundled `lattice-miner`; otherwise a path to a worker executable |
| `workers` | 1 | parallel worker processes |
| `batchSize` | 2,000,000,000 | total nonces searched per round |
| `recipients` | none | address for each chain. **A chain left out burns its reward and fees.** |
| `minWork` | none | local search filter, `2^N` or a decimal. Not consensus. |
| `minBlockIntervalSeconds` | none | minimum spacing between parent template builds |
| `templateTimeoutSeconds` | 60 | 1…60 |
| `roundDeadlineMultiplier` | 10 | ≥1; headroom on the round deadline |

Every chain named in `recipients` or `minWork` must be hosted. The retired
keys `mine.chain` and `mine.rewards` are rejected.

## Daemon-only flags (no lattice.json key)

`--rpc-bind` (loopback values only), `--rpc-cookie-file`,
`--peer-target` (8), `--peers-before-seeds` (2),
`--minimum-peer-key-bits` (0), `--overlay-max-connections-per-netgroup`,
`--peer-search-interval` (600 s; 0 turns search off, for isolated nodes
only).

## Environment variables

| Var | Effect |
|---|---|
| `LATTICE_SYNC_TRACE=1` | Sync and acquisition traces to stderr (that is, `log/lattice-node.log`). Any other non-empty value is a file path. Produces a lot of output, so use it only while diagnosing. |
| `LATTICE_SYNC_TRACE_MAX_BYTES` | Size cap for the trace file (default 64 MiB). |
| `LATTICE_FORBID_DEFAULT_PEERS=1` | Refuse to start on the built-in seeds. Use it in tests or CI so they never join the public network. |

The CLI itself reads no environment variables.
