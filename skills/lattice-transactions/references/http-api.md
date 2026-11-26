# lattice-node HTTP API: quick reference

The authoritative reference is the node's HTTP API, in four pages:

| Page | Covers |
|---|---|
| [HTTP API](https://lattice.build/docs/build/http-api/) | Listeners, authentication, conventions, and an index of every route. |
| [Reads and transactions](https://lattice.build/docs/build/http-api/reads-and-transactions/) | The read routes, `POST /transactions` and its refusals, fees. |
| [Mining](https://lattice.build/docs/build/http-api/mining/) | The miner's routes. |
| [Operator](https://lattice.build/docs/build/http-api/operator/) | Configuration, chains, pins, tokens, restart and wipe. |

This file keeps only what is needed at hand to call a node correctly. When
it and those pages disagree, the pages are right.

## Listeners

| Listener | Where | Credential | Serves |
|---|---|---|---|
| Operator API (`rpc`, 8080) | loopback only | the node's cookie, on every route except `GET /health` | every route |
| Remote operator listener (`operatorListen`) | an address the operator chooses; off by default | `Authorization: Bearer <token>` on every route, `/health` included | the operator routes its token's role or route list allows |
| Public reads (`publicRead`) | all interfaces; off by default | none; rate-limited for all clients together | the GET read routes, plus `POST /transactions` when `publicSubmit` is on |

## Calling it

- **Cookie.** `curl --user "$(cat <root>/chains/Nexus/.cookie)" …` (Basic,
  the file's content), or `Authorization: Bearer <token>` with the part
  after `__cookie__:`. The cookie changes at every start: read the file
  again after a restart.
- **Chain.** Select a chain with `?chainPath=Nexus/Alpha`. Without it the
  route answers for Nexus. A path the node does not host is 404.
- **Integers.** Every 64- or 128-bit integer (balances, nonces, heights,
  amounts, fees) is a decimal **string**, in answers and in
  `POST /transactions`. A JSON number in its place is refused with 400.
- **Operator POSTs** need `Content-Type: application/json` and a loopback
  `Host` (403 or 415 otherwise).
- **Browsers.** A request with an `Origin` header is refused with 403 on the
  operator API unless that origin is in `rpcAllowedOrigins`, and it still
  needs the cookie.
- **Errors** are `{"error":{"message":"<reason>"}}`.
- **Caching.** Fetch tips and nonces with caching disabled: live reads are
  cached for a few seconds.

## Routes used most

Field lists are on the pages linked above.

| Route | For |
|---|---|
| `GET /health` | A chain's phase (`active` or `awaitingGenesis`), tip and height. No credential needed on the operator API. |
| `GET /api/chain/info` | `minRelayFee`, `acceptsSubmit`, height and tip. |
| `GET /api/chain/spec` | The chain's ChainSpec. |
| `GET /api/block/latest`, `/api/block/:height-or-cid`, `/api/blocks` | Blocks, with `rewardRecipient` and `rewardCredited`. |
| `GET /api/state/account/:addr` | `{owner, balance, nonce}` at the tip; `nonce` is the next one to use. |
| `GET /api/transaction/:cid` | A transaction's summary; `blockHeight` and `blockHash` only once it is in a block of the canonical chain. |
| `GET /api/mempool`, `GET /api/fees` | Pending transactions, and what to pay to be in the next blocks. |
| `GET /api/chain/children`, `/api/chain/endpoints?chainPath=P/D` | A chain's child directories, and a child's declared read and submit URLs (**unverified**). |
| `GET /api/deposits`, `/api/deposit-state`, `/api/receipt-state` | Cross-chain exchange state, each with a state proof. |
| `GET /volumes/:cid` | One Volume as a binary archive. On the public listener, 503 with `Retry-After` while the node reads it from peers. |
| `POST /transactions` | Submit ([Transaction format](transaction-format.md)). |

Operator-only: `GET /status`, `GET /metrics`, `GET /hosted-chains`, the
`/config`, `/chains`, `/pins` and `/tokens` routes, `POST /restart`, `/stop`
and `/wipe`, and the miner's `/mining/*`. Prefer the `lattice` commands that
drive them ([Operator CLI](https://lattice.build/docs/run/operator-cli/)).

## Submit refusals

`POST /transactions` names why it refused.

| Reason | Status | Do |
|---|---|---|
| `nonceAhead` | 400 | Resend unchanged once the first signer's earlier transaction is mined. |
| `unavailable` | 400 | Resend unchanged later: the node cannot judge it yet. |
| `insufficientFunds` | 400 | Resend unchanged once the pooled transactions that debit the same account are mined. |
| `full` | 429 | Resend later: the pool is full of transactions paying a higher rate. |
| `busy` | 503 | Resend after `Retry-After`. |
| `feeTooLow` | 400 | Raise the fee: it conflicts with pooled transactions and does not pay enough to displace them. |
| `belowMinRelayFee` | 400 | Raise the fee to this node's `minRelayFee`. |
| `tooLarge` | 400 | It cannot fit in a block by itself. |
| `invalidState` | 400 | Change it: bad signature, stale nonce, a balance that cannot pay, a missing deposit or receipt, or a policy's reject. Re-read the balance, the nonce and the keys it names. |
| `unknownChain` | 404 | This node does not host the transaction's `chainPath`. |

On the public listener: a body over 1 MiB is 413, a submit over the
listener's rate is 429 with no reason, and the route is 404 when public
submit is off.
