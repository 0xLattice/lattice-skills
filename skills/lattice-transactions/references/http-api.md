# lattice-node HTTP API

The canonical reference is `docs/rpc-api.md` in the lattice-node repo.
This page summarizes it as of `main`, October 2026.

## Listeners and auth

| Listener | Bind | Auth | Serves |
|---|---|---|---|
| Operator RPC (`rpc`, 8080) | loopback only | cookie on every route except `GET/HEAD /health` | all read routes plus operator routes |
| Public read (`publicRead`) | 0.0.0.0 | none, rate-limited | read routes only; `POST /transactions` if `publicSubmit` |

- Cookie: `curl --user "$(cat <root>/chains/Nexus/.cookie)"` (Basic auth with
  the file's contents), or `Authorization: Bearer <token>`. It rotates on
  every restart.
- A browser `Origin` gets 403 unless it is listed in `rpcAllowedOrigins`.
- Operator POSTs need a loopback `Host` and `Content-Type: application/json`;
  otherwise they get 403 / 415.
- Public limits: 25 req/s per client, 1 req/s per client on *expensive*
  routes (marked ★ below), 200 req/s for the whole listener, and 10 submits/s.
  Over the limit returns 429.
- Choose a chain with `?chainPath=Nexus/Alpha` (URL-encode the `/` as
  `%2F` if your client needs it). Omitting it means Nexus. An unhosted path
  returns 404.
- Errors look like `{"error":{"message":"<reason>"}}`.
- Immutable by-CID reads are cached for a year. Live reads are cached for 3 s,
  so fetch nonces and tips with `cache: no-store`.

## Read routes (both listeners)

| Route | Returns |
|---|---|
| `GET /health` | `phase` (`active` / `awaitingGenesis`), `chainPath`, `nexusGenesisCID`, `tipCID`, `height`, `revision`, `mempoolCount`, `mempoolBytes`, `bestHeaderHeight`, `waiting` |
| `GET /api/block/latest` | `height, hash, transactionCount, timestamp, previousBlock, rewardRecipient, rewardCredited` |
| `GET /api/block/:height-or-cid` ★ | the above plus `nonce, version, target, nextTarget, transactionsCID, postStateCID, childBlockCount` |
| `GET /api/blocks?before=&limit=` | a page of blocks (limit ≤ 25) with `nextBefore` |
| `GET /api/block/:cid/transactions?offset=&limit=` ★ | transaction summaries (limit ≤ 100) |
| `GET /api/block/:cid/children` ★ | `{directory, blockHash, height?, transactionCount?}` |
| `GET /api/transaction/:cid` ★ | a summary: signers, nonce, account/deposit/receipt/withdrawal actions (**no KV `actions`**), plus `blockHeight`/`blockHash` only if it is in the canonical chain; covers pooled transactions too |
| `GET /transactions/:cid` | `{cid, transaction}`, the full transaction |
| `GET /api/state/account/:addr` | `{owner, balance, nonce}` at the tip (`nonce` = next) |
| `GET /accounts/:owner?block=<cid>` | balance and nonce at a specific block (`block` required) |
| `GET /api/mempool` | `{count, transactions:[cid…]}` (≤ 200) |
| `GET /api/peers` | peer count (plus the list on Nexus) |
| `GET /api/chain/info` | `genesisHash, height, tipCID, chain, acceptsSubmit, minRelayFee` |
| `GET /api/chain/spec` | the ChainSpec numeric fields |
| `GET /api/chain/genesis` | `{genesisHash}` |
| `GET /api/chain/children?limit=&after=` | child directories committed by executed blocks |
| `GET /api/chain/endpoints?chainPath=P/D` ★ | `endpoints`, `submitEndpoints` and `committedBlock` for child D; answered by a host of P; **unverified** |
| `GET /api/deposits?limit=&after=` ★ | open deposits on a child, with a proof |
| `GET /api/deposit-state?key=demander/amount/nonce` | `{exists, value, proof}` |
| `GET /api/receipt-state?demander=&amount=&nonce=&chainPath=P/D` | read from the parent P: `{exists, withdrawer, proof}` |
| `GET /volumes/:cid` ★ | a binary Volume archive (≤ 64 MiB). May return 503 + `Retry-After` while it fetches from peers. |

## Operator routes (loopback + cookie)

| Route | Notes |
|---|---|
| `GET /status`, `GET /metrics` | full status (including `templateDigest`) and Prometheus metrics |
| `POST /transactions` | `{"transaction":{signatures, body}}` → `{transactionCID, mempoolCount, mempoolBytes}` |
| `POST /mining/templates`, `POST /mining/work` | the miner protocol; see the `lattice-mining` skill |
| `GET` / `POST /hosted-chains` | list, or **replace**, the hosted set: `{"hostChain":[…], "premineTo":[…], "chainPeer":[…], "policyModule":[…]}` |
| `GET /pins`, `POST /pins`, `DELETE /pins/:cid` | flat pins: `{cid, entries?:{cid: base64}}` |
| `GET /fetch/:cid` | fetch a Volume from peers without storing it; the response has the `POST /pins` shape |

Pin errors: `invalidCID`, `invalidVolume`, 404 `notHeld` (the node doesn't
hold it and you didn't supply `entries`), 409 `conflictingVolume`, 503
`contextChanged`.
