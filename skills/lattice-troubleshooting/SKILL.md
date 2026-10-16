---
name: lattice-troubleshooting
description: Diagnose and fix lattice-node problems — node won't start (wipe required, missingMaterializedVolume, storage lock, lattice.json errors), height not advancing, child stuck in awaitingGenesis, no peers, RPC 401/403/404/429, transactions not landing, mining not producing blocks. Use when a user reports a Lattice node error message, a stuck or stalled chain, failing lattice CLI commands, or unexpected API responses.
---

# Troubleshooting lattice-node

Gather facts before you change anything. Only `lattice wipe` deletes data, but
a wrong restore can mix snapshots, so read first, then act.

## 1. Gather facts

```bash
lattice status --root R                                   # per-chain phase / height / tip
tail -n 200 R/log/lattice-node.log                        # or: journalctl -u lattice-node -n 200
tail -n 100 R/log/mine.log                                # if mining
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

## 2. Match the symptom

### The node won't start

| Message / symptom | Cause | Fix |
|---|---|---|
| `The node store is incompatible (…); stop the process, wipe …` | The store belongs to another schema epoch or another genesis (a flag-day release). | Without systemd: `lattice down && lattice wipe --root R && lattice up --root R`. Under systemd: `systemctl stop lattice-mining-coordinator lattice-node && lattice wipe --root R && systemctl start lattice-node`, because the supervisor would otherwise respawn the node. There is no migration. Confirm with the user before wiping. |
| `missingMaterializedVolume` | A hosted child's spec, or a journaled pending transaction, is missing from `volumes.db` and from `specs/`. | Restore the **whole** matched backup (`state.db` + `volumes.db` + `specs/`), or wipe. |
| Saved proof corruption | Saved child proofs don't decode or don't match. | Restore the matched backup, or wipe. |
| Storage lock held / `tree storage is in use by a running node` | Another process owns `chains/Nexus/storage.lock`. | Stop the other node. Give each node its own `--root`. |
| `unknown lattice.json key(s): …` | A typo, or a retired key (`mine.chain`, `mine.rewards`). | Remove it. |
| `hosted chain X is not a new Nexus-rooted path listed after its parent` | Wrong order, a duplicate, or a bad path. | List each parent first; paths start with `Nexus`. |
| `peers are listed for X, which is not a hosted chain` | A `peers` key names a chain that isn't hosted. | Host it, or drop the key. |
| `port N is zero or used twice` | Port collision. | Make `listen`, `rpc`, `publicRead` and `webSocket` distinct. |
| `publicSubmit requires publicRead` | — | Set `publicRead`. |
| `identity key … group/other-readable; chmod 600 it` | Permissions are too open. | `chmod 600 R/identity/Nexus.key` |
| `lattice-node not found beside lattice or in /usr/local/bin` | The binaries were installed separately. | Install all four binaries in one directory. |
| Bind failure on the WebSocket port | Fatal only when `webSocket` is set explicitly and the port is taken. When it is unset, the node starts without browser peers. | Give each node on a host its own `webSocket`, or set it to 0. |
| `LATTICE_FORBID_DEFAULT_PEERS=1: pass --peer or --no-default-peers` | That env var is set. | Configure `peers`, or unset the variable. |
| A `peers` object is refused after a downgrade | Older nodes accept only the array form. | Use the array form, or upgrade. |

### The node runs but is wrong

| Symptom | Check |
|---|---|
| `running, rpc unreachable` | The node is still booting (cold start can take minutes), the `rpc` port is wrong, or the node is wedged. Read the log. |
| Height doesn't advance even though peers are connected | Every node must run the same wire revision and genesis. Also check: a stale tip at the peers themselves; the clock (`parent.ts < ts <= now`, so a fast clock elsewhere holds blocks); `--peer-search-interval 0`; unreachable endpoints. Run `LATTICE_SYNC_TRACE=1` briefly and read the trace. |
| No peers | A `peers: []` setting means isolation. Under NAT, outbound must work; set `externalAddress` if you are behind a proxy. Deleting `peers.json` is harmless. Make sure 4001/tcp is reachable outbound. |
| Child stuck at `awaitingGenesis` (created by you) | Check, in order: every ancestor is hosted; the spec is in `specs/`; the parent has executed a non-genesis block; the parent changes state (has a reward **recipient** or gets transactions); a recipient and worker are configured; **the miner was restarted after the child was added**. |
| Child stuck at `awaitingGenesis` (followed) | No peer has served it yet. Check that it exists (`/api/chain/children` on a parent host) and add `peers["Nexus/X"]` seeds. |
| Config change had no effect | `lattice up` on a running node only reconciles `hostedChains`. Run `lattice down && lattice up` (or `systemctl restart lattice-node`). |
| The node comes back after `lattice down` | systemd is supervising it (`up --foreground`). Run `systemctl stop lattice-node`. |
| Disk keeps growing | By default the fact history is unbounded. Consider `blockBodies: none` / `state: tip`, but **not on a chain's only public miner or archive node**. `volumes.db` reuses freed space and never shrinks. |
| `state.db` was restored without `volumes.db` | The node serves 404s and leaves transactions pooled. Restore a matched pair, or resync. |

### The HTTP API

| Status | Meaning |
|---|---|
| 401 | Missing or stale cookie. It rotates on every restart, so re-read `R/chains/Nexus/.cookie`. |
| 403 | A browser `Origin` that isn't in `rpcAllowedOrigins`, or an operator POST with a non-loopback `Host`. |
| 404 on a chain read | The `chainPath` isn't hosted by this node (`GET /hosted-chains`), or it is a CID or account the node doesn't hold. |
| 404 on `/api/state/account` | The node has no tip, or doesn't hold that chain's state. It does **not** mean the account is empty. |
| 415 | An operator POST without `Content-Type: application/json`. |
| 429 on public reads | Rate limits. Behind an L4 proxy that collapses client addresses, set `publicReadRate` and `publicReadExpensiveRate` to 0 and keep `publicReadMaxRate`. Also check whether nginx, rather than the node, answered. |
| 503 + `Retry-After` on `/volumes` | The node is fetching from peers. Back off and retry. |

### Transactions

| Symptom | Check |
|---|---|
| 400 `invalidState` | This one name covers every consensus refusal. Re-read the balance (is it enough for amount + fee?), the nonce, the `chainPath`, and for a withdrawal the deposit and receipt state. On a chain with a policy, the policy may have rejected it. |
| A 200 was returned, but the transaction never lands | It is probably parked as `future` (a nonce gap from an earlier missing transaction) or `unavailable`. Check `/api/mempool` and the signer's nonce. Fee below other nodes' `minRelayFee`, so it doesn't relay. Public submits are lost if the node restarts. |
| `belowMinRelayFee` | Raise `--fee` to the node's `minRelayFee` (from `/api/chain/info`). |
| A withdrawal fails right after the receipt | Wait for one more parent block after the receipt's block. |

### Mining

| Symptom | Check |
|---|---|
| `NOT MINING: no usable template answer` | The node is still syncing (409) or a parent is unavailable (503). Wait until `lattice status` shows `active` for Nexus, and check `log/lattice-node.log`. On a slow host, template builds that take longer than 60 s can't be fixed with configuration. |
| Blocks are mined but the balance stays 0 | No recipient for that chain, so the reward was burned. Check `lattice mine status`, and look at `rewardRecipient` on the block. |
| A new child never gets rewards | `lattice mine` wasn't restarted after the child was added. |
| `ROUND DEADLINE EXCEEDED`, or stray `lattice-miner` processes | Raise `roundDeadlineMultiplier`, and kill the strays. |

## 3. Before a destructive fix

- `lattice wipe` and restoring a backup both discard data. Tell the user
  exactly what will be lost: chain data resyncs from peers, and only content
  this node uniquely held is gone for good. Get their confirmation.
- Stop mining and the node, and with systemd stop the units. Then act.
- Keep `identity/`, `lattice.json`, `specs/` and the wallet files. A wipe
  already preserves the first three.
