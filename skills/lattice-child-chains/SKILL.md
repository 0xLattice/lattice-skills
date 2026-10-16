---
name: lattice-child-chains
description: Create, follow, host, unhost and publish Lattice child chains with lattice-node — choosing a ChainSpec (block time, reward, premine, limits, WASM policies), `lattice child create`, joining someone else's chain via hostedChains, discovering children and their read endpoints, and diagnosing a child stuck in awaitingGenesis. Use when a user wants to launch their own chain, run or join an existing child chain such as Nexus/<name>, or make a child chain discoverable.
---

# Lattice child chains

A child chain is an independently validated chain at an absolute path under
Nexus (`Nexus/Alpha`, `Nexus/Alpha/Beta`). Its blocks are committed inside
parent blocks and secured by merged mining. There is no deploy transaction,
no permission and no authority. **Whoever's genesis accumulates the most work
at a path wins that path.**

Hosting a child means hosting its whole ancestry in the same process, because
the node validates every parent itself.

## Before you start: create or follow?

| The user wants… | Do this | Never do this |
|---|---|---|
| a **new** chain they define | `lattice child create <path> …` | — |
| to run or join a chain **someone else launched** | add the path to `hostedChains`, then `lattice up` | `child create` the same path. That mines a *rival* genesis, i.e. a second competing chain with the same name. |

If you're not sure whether a path already exists, check it from any node
that hosts the parent:

```bash
curl -s 'https://<parent-reads>/api/chain/children?chainPath=Nexus' | jq   # {"directories":[…], "next":…}
```

## Create a new child

```bash
# 1. Address for the premine and/or mining rewards
lattice key generate --out ./alpha-wallet.json          # prints address:

# 2a. Quick: build the spec from flags
lattice child create Nexus/Alpha --root R --block-time 10000 --reward 1000 --premine 0

# 2b. Full control: a complete spec, plus policy modules and a premine
lattice child create Nexus/Alpha --root R --spec alpha-spec.json \
  --policy-module rules.wasm --premine-to <address>

# 3. Add a mining recipient for Nexus/Alpha in lattice.json mine.recipients,
#    then RESTART mining. `lattice mine` reads its config once at start.
lattice mine stop --root R && lattice mine start --root R

# 4. Watch it go from awaitingGenesis to active
lattice status --root R
curl -s 'http://127.0.0.1:8080/health?chainPath=Nexus%2FAlpha' | jq
```

What `child create` does:

- Writes `specs/Nexus%2FAlpha.json` (immutable), the premine recipient and
  the policy modules.
- Appends the path to `hostedChains`.
- Tells a running node live, with no restart.

The genesis is built in the **next mining template** that carries it, so
**someone must be mining against a node that holds the spec**. The command
refuses a path that already has a spec ("a child chain is created once").

Flag rules:

- `--spec` overrides `--block-time`, `--reward` and `--premine`.
- A nonzero premine requires `--premine-to`. `--premine-to` without a premine
  is an error.
- Every module named in `spec.wasmPolicies` must be passed with
  `--policy-module`. Each file's CID must match, and extra files are refused.
- A nested path's parent must already be Nexus or a hosted child.

## Choosing the spec

The full field reference, a sample JSON and the policy ABI are in
`references/chainspec.md`. The main choices:

- **`targetBlockTime`** (ms) and **`halfLife`** (blocks): the ASERT pace.
  Smaller `halfLife` means difficulty reacts faster.
- **`initialReward`, `halvingInterval`, `premine`**: issuance.
  `initialReward` must be > 0. The premine is paid in one go, at genesis, to
  `--premine-to`.
- **Limits**: `maxBlockSize`, `maxNumberOfTransactionsPerBlock` and
  `maxStateGrowth` (bytes of new state per block).
- **`wasmPolicies`**: optional accept/reject rules. Rebuilding a policy
  changes its CID, which changes the spec and therefore the chain.

The spec cannot be changed after genesis. To change any rule, launch at a
new path.

## Follow an existing child

There is no `child follow` command. You follow a chain by hosting it
**without** a spec:

```jsonc
// R/lattice.json
"hostedChains": ["Nexus/Alpha"],                         // ancestors first; Nexus is implicit
"peers": { "Nexus/Alpha": ["<pubkey>@host:4001"] }       // optional seeds; helps under NAT
```

```bash
lattice up --root R        # running node: hot-adds the chain; otherwise starts it
lattice status --root R
```

The node finds the chain's peers through the DHT rendezvous keyed by the
path, then syncs and validates the chain from its genesis. You can mine it
too: add a recipient for it and restart `lattice mine`.

## Host and unhost live

- `hostedChains` is the source of truth. `lattice up` on a running node
  sends `POST /hosted-chains` and restarts only if that fails.
- **Unhost**: remove the path and its descendants from `hostedChains`. Also
  remove their keys from `mine.recipients`, `mine.minWork` and `peers`, or
  validation fails. Then run `lattice up`. The chain and **everything below
  it** stops, but its data stays, so listing it again resumes it.
- To re-add a chain you created, put its path back in `hostedChains`. The
  spec is still in `specs/`. Running `child create` again is refused.
- Raw API (operator port, cookie). This sends the **full replacement set**,
  not an addition:

  ```bash
  C="$(cat R/chains/Nexus/.cookie)"
  curl -s --user "$C" -H 'Content-Type: application/json' -X POST \
    http://127.0.0.1:8080/hosted-chains -d '{"hostChain":["Nexus/Alpha"]}'
  ```

  A later `lattice up` reconciles the node back to `lattice.json`.

## Discover chains and endpoints

| Read | Returns |
|---|---|
| `GET /api/chain/children?chainPath=P[&after=]` | directories that P's executed blocks have committed (paged, cap 200) |
| `GET /api/block/:cid/children` | child commitments in one block |
| `GET /api/chain/endpoints?chainPath=P/D` | declared read URLs (`endpoints`) and submit URLs (`submitEndpoints`) for child D, answered by a host of P. **Unverified.** |
| `GET /api/chain/info\|spec\|genesis?chainPath=P/D` | the chain's info, spec and genesis on a node that hosts it |

Always confirm a discovered URL with
`GET <url>/api/chain/info?chainPath=P/D` before you trust it.

## Make your child discoverable

Set `publicRead`, put TLS in front of it, set `publicReadURL`, optionally set
`publicSubmit`, then run `lattice down && lattice up`. Verify through a
parent host's `/api/chain/endpoints?chainPath=<your path>`. The full steps
are in the `lattice-node-operator` skill.

## Stuck at `awaitingGenesis`

Check these in order:

1. Every ancestor is in `hostedChains`.
2. The spec exists in `specs/` and decodes.
3. The immediate parent has executed at least one **non-genesis** block.
4. The parent actually changes state. That needs a positive reward **with a
   recipient**, or state-changing transactions. A parent whose rewards burn
   and which gets no transactions can never carry a nested genesis.
5. A mining recipient and a worker are configured, **and the miner was
   restarted after the child was added**.
6. Someone is actually mining against a node that holds the spec.

If you only *follow* the chain (no spec), it stays in `awaitingGenesis`
until a peer serves its blocks. Check peers and add `peers["Nexus/X"]`
seeds.

## Gotchas

- Creating a chain twice at one path, on two hosts, makes two competing
  chains. Coordinate, or let fork choice pick a winner.
- Losing `specs/` (or its `.wasm` files) can make the node fail at start with
  `missingMaterializedVolume`. Back up `specs/` along with `lattice.json`.
- Keep path components short ASCII names without `/`. Paths are
  case-sensitive.
- A `chainPath` that isn't hosted returns 404 on every read and on
  transaction submits.
