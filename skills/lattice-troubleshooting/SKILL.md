---
name: lattice-troubleshooting
description: Diagnose and fix lattice-node problems — node won't start (wipe required, missingMaterializedVolume, storage lock, configuration errors), height not advancing, child stuck in awaitingGenesis, no peers, RPC 401/403/404/429, transactions not landing, mining not producing blocks. Use when a user reports a Lattice node error message, a stuck or stalled chain, failing lattice CLI commands, or unexpected API responses.
---

# Troubleshooting lattice-node

Gather facts before you change anything. Only `lattice wipe` deletes data, but
a wrong restore can mix snapshots, so read first, then act.

## 1. Gather facts

```bash
lattice status --root R                                   # per-chain phase / height / tip / what it waits on
lattice logs --root R --tail 200                          # a stopped node: journalctl -u lattice-tree -n 200
lattice logs --miner --root R                             # if mining
lattice config get --root R                               # the configuration the node reads
curl -s http://127.0.0.1:8080/health | jq                 # no cookie needed
C="$(cat R/chains/Nexus/.cookie)"
curl -s --user "$C" http://127.0.0.1:8080/status | jq
curl -s --user "$C" http://127.0.0.1:8080/hosted-chains | jq
curl -s --user "$C" "http://127.0.0.1:8080/api/peers" | jq .count
ls -l "$(dirname "$(command -v lattice)")"/lattice*      # all four binaries from one build/release?
```

Compare `nexusGenesisCID` in `/health` with
`bafyreiggtg4ezifboyekbf4gxst2jr3mpjqcxsbmopy7w6fp46g4ngpgxa`. If they
differ, the node is on a different network or build.

`lattice status` also lists chains configured but not running, settings
waiting for a restart, and inert settings. `sync.blockedOn` in `GET /status`
names what a chain's tip waits on. The node's own troubleshooting page is
https://lattice.build/docs/run/troubleshooting/.

## 2. Match the symptom

### The node won't start

| Message / symptom | Cause | Fix |
|---|---|---|
| Exit 78 | The configuration does not check. The reason is on standard error. | Fix `lattice.json` or the fragment it names. systemd leaves the unit stopped until then; launchd and Docker retry. |
| `The node store is incompatible (…); stop the process, wipe …` (wipe required) | The store was written for another schema version or another Nexus genesis. | There is no migration: `lattice wipe --root R`, then start the node (under systemd, stop the units first and start them after). A wipe keeps the identity, configuration, specs and tokens. Confirm with the user before wiping. |
| `missingMaterializedVolume` | A hosted child's genesis spec is in neither its Volume store nor `specs/`, or one of the node's own pending transactions is missing from its store. | Restore the **whole** matched backup (`state.db` + `volumes/` + `specs/`), or wipe. |
| Saved proof corruption | Saved child proofs don't decode or don't match. | Restore the matched backup, or wipe. |
| Storage lock held / `tree storage is in use by a running node` | Another node, or a `lattice` command editing a stopped node, is using the data directory. | Only one may at a time. Give each node its own `--root`. |
| `unknown lattice.json key(s): …` | A typo, or a key from another release. | Remove it. Mining settings belong in `miner.json`, not `lattice.json`. |
| `hosted chain X is not a new Nexus-rooted path listed after its parent` | Wrong order, a duplicate, or a bad path in `hostedChains`. | List each parent first; paths start with `Nexus`. Prefer `lattice child host`. |
| `port N is zero or used twice` | Port collision. | Make `listen`, `rpc`, `publicRead`, `operatorListen` and `webSocket` distinct. |
| `publicSubmit requires publicRead` | — | Set `publicRead`. |
| `identity key … group/other-readable; chmod 600 it` | Permissions are too open. | `chmod 600 R/identity/Nexus.key` |
| `lattice-node not found beside lattice or in /usr/local/bin` | The binaries were installed separately. | Install all four binaries in one directory. |
| Bind failure on the WebSocket port | Fatal only when `webSocket` is set explicitly and the port is taken. When it is unset, the node starts without browser peers. | Give each node on a host its own `webSocket`, or set it to 0. |
| `LATTICE_FORBID_DEFAULT_PEERS=1: pass --peer or --no-default-peers` | That env var is set. | Configure `peers`, or unset the variable. |
| A `peers` object is refused after a downgrade | Older nodes accept only the array form. | Use the array form, or upgrade. |

### The node runs but is wrong

| Symptom | Check |
|---|---|
| `running, rpc unreachable` | The node is still starting, or it is wedged. Read the log. |
| Height doesn't advance even though peers are connected | Every node must run the same wire revision and genesis. Also check: a stale tip at the peers themselves; the clock (`parent.ts < ts <= now`, so a fast clock elsewhere holds blocks); `--peer-search-interval 0`; unreachable endpoints. Run `LATTICE_SYNC_TRACE=1` briefly and read the trace. |
| No peers | A `peers: []` setting means isolation. Under NAT, outbound must work; set `externalAddress` if you are behind a proxy. Deleting `peers.json` is harmless. Make sure 4001/tcp is reachable outbound. |
| Child stuck at `awaitingGenesis` (created by you) | Check, in order: every ancestor is hosted; the spec is in `specs/`; the parent has executed a non-genesis block; the parent changes state (has a reward **recipient** or gets transactions); a miner is running with recipients for the parent and the child in `miner.json`. |
| Child stuck at `awaitingGenesis` (hosted, not created here) | No peer has served it yet. Check that it exists (`/api/chain/children` on a parent host) and add seeds for it under `peers`. |
| Config change had no effect | Only `hostedChains` and child-chain `peers` apply to a running node. `lattice status` lists what waits for a restart; `lattice restart` applies it. A file edited by hand is read at the next start or on `SIGHUP`. |
| `lattice restart` or `lattice wipe` leaves the node stopped | They rely on a process manager to start it again. Run by hand, start it yourself with `lattice up`. |
| The node stops when the terminal closes | `lattice up` runs in the foreground. Put it under systemd, launchd or a container (`lattice emit-systemd`). |
| Disk keeps growing | By default history is kept without bound. Consider `blockBodies: none` or `state: recent`, but **not on a node that mines a chain others should follow, or one an explorer reads**. The database files reuse freed space and never shrink; to reclaim it, sync again from an empty data directory. |
| `state.db` was restored without `volumes/` | The node runs but holds only what it has stored since, and its public reads answer not found for the rest. Restore a matched pair, or sync again. |
| A chain stops at a block: `blockedOn: bodiesUnavailable`, "receipt proofs not obtained" | See the node's troubleshooting page (link above): it is repaired once from a node that re-executes the parent from genesis. |
| A block never connects: `cidMismatch` naming this node | Bytes in a Volume store were damaged on disk. Restore the matched backup, or wipe. |

### The HTTP API

| Status | Meaning |
|---|---|
| 401 | Missing or stale cookie. It changes at every start, so re-read `R/chains/Nexus/.cookie`. On the remote operator listener: no known bearer token. |
| 403 | A browser `Origin` that isn't in `rpcAllowedOrigins`, an operator POST with a non-loopback `Host`, or a route the remote listener's token does not allow. |
| 404 on a chain read | The `chainPath` isn't hosted by this node (`GET /hosted-chains`), or it is a CID or account the node doesn't hold. |
| 404 on `/api/state/account` | The node has no tip, or doesn't hold that chain's state. It does **not** mean the account is empty. |
| 415 | An operator POST without `Content-Type: application/json`. |
| 429 on public reads | First find out whether the node or a proxy in front answered. The node's limits are listener-wide (`publicReadMaxRate`, `publicSubmitRate`), so one busy client can spend the budget for everyone: put per-client limits in the proxy, then raise the node's limit or set it to `0`. |
| 503 + `Retry-After` on `/volumes` | The node is reading the content from peers, or its content budget is full. Back off and retry. |
| 404 for old blocks on public reads | The node let go of those bodies (`blockBodies: none`). |

### Transactions

| Symptom | Check |
|---|---|
| 400 `invalidState` | The transaction can never be pooled as it is. Re-read the balance (is it enough for amount + fee?), the nonce, the `chainPath`, and for a withdrawal the deposit and receipt state. On a chain with a policy, the policy may have rejected it. |
| 400 `nonceAhead` | An earlier transaction from the same first signer is not mined yet. Nothing is parked: send it again once that one is mined. |
| 400 `unavailable` / `insufficientFunds`, 429 `full` | Retriable as they are: the node cannot judge it yet, pooled transactions already spend the balance, or the pool is full of better-paying ones. |
| 400 `feeTooLow` | It conflicts with a pooled transaction and does not pay enough to displace it. Raise the fee. |
| A 200 was returned, but the transaction never lands | Check `/api/mempool` and `/api/transaction/<cid>`. Its fee may be below other nodes' `minRelayFee`, so it does not relay; it may have been displaced by a better-paying one. A public submit is lost if the node restarts before relaying it. |
| `belowMinRelayFee` | Raise `--fee` to the node's `minRelayFee` (from `/api/chain/info`). |
| A withdrawal fails right after the receipt | Wait for one more parent block after the receipt's block. |

### Mining

| Symptom | Check |
|---|---|
| `NOT MINING: no usable template answer` | The node gave no usable answer within `templateTimeoutSeconds` (60 at most). Check that the node runs and that `lattice status` shows `active` for Nexus, and read `lattice logs`. |
| Blocks are mined but the balance stays 0 | No recipient for that chain, so the reward was burned. Check `lattice mine status`, and look at `rewardRecipient` on the block. |
| A new child never gets rewards | It has no entry under `recipients` in `miner.json`. Add one; it applies to the next round. |
| `ROUND DEADLINE EXCEEDED`, or stray `lattice-miner` processes | Raise `roundDeadlineMultiplier` in `miner.json`, and kill the strays. |
| `no miner settings at …/miner.json` | Write `miner.json` in the root. A `mine` block in `lattice.json` is ignored by the node. |

## 3. Before a destructive fix

- `lattice wipe` and restoring a backup both discard data. Tell the user
  exactly what will be lost: chain data resyncs from peers, and only content
  this node uniquely held is gone for good. Get their confirmation.
- Before restoring a backup, stop mining and the node (with systemd, stop
  the units). `lattice wipe` on a running node wipes as it restarts.
- Keep `identity/`, `lattice.json`, `config.d/`, `specs/` and the wallet
  files. A wipe already preserves all but the wallet files.
