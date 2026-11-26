---
name: lattice-child-chains
description: Create, follow, host, unhost and publish Lattice child chains with lattice-node — choosing a ChainSpec (block time, reward, premine, limits, WASM policies), `lattice child create`, joining someone else's chain with `lattice child host`, discovering children and their read endpoints, and diagnosing a child stuck in awaitingGenesis. Use when a user wants to launch their own chain, run or join an existing child chain such as Nexus/<name>, or make a child chain discoverable.
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
| to run or join a chain **someone else launched** | `lattice child host <path>` | `child create` the same path. That mines a *rival* genesis, i.e. a second competing chain with the same name. |

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

# 3. Add a recipient for Nexus/Alpha to "recipients" in R/miner.json.
#    The mining loop reads the file again before its next round: no restart.
lattice mine status --root R

# 4. Watch it go from awaitingGenesis to active
lattice status --root R
curl -s 'http://127.0.0.1:8080/health?chainPath=Nexus%2FAlpha' | jq
```

What `child create` does:

- Keeps the spec, the premine recipient and the policy modules under
  `specs/`. A spec is immutable.
- Adds the path to `hostedChains`.
- Hosts the chain at once on a running node, with no restart.

The genesis is built in the **next mining template** that carries it, so
**someone must be mining against a node that holds the spec**. Creating the
same path again with a different spec is refused.

Flag rules:

- `--spec` overrides `--block-time`, `--reward` and `--premine`.
- A nonzero premine requires `--premine-to`. `--premine-to` without a premine
  is an error.
- Every module named in `spec.wasmPolicies` must be passed with
  `--policy-module`. Each file's CID must match, and extra files are refused.
- A nested path's parent must already be Nexus or a hosted child.
- `--genesis-transactions <file>` puts signed transactions in the genesis,
  after the premine: a JSON array of what `POST /transactions` takes. Each
  must be for this chain, carry no coin or cross-chain actions, and pass the
  spec's policies.
- With `--node`, these files are read on the machine running `lattice`.

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

## Host an existing child

You follow a chain someone else launched by hosting it **without** a spec:

```bash
lattice child host Nexus/Alpha --root R     # a running node starts it at once
lattice status --root R
```

The node hosts the chain's ancestors too, finds the chain's peers through
its parent's peers and the rendezvous keyed by its path, then syncs and
validates the chain from its genesis. A stopped node hosts it from its next
start.

- Optional seeds for the chain, which help when it has few hosts:
  `lattice config set --patch '{"peers": {"Nexus/Alpha": ["<pubkey>@host:4001"]}}'`.
  Child-chain peers apply at once.
- To mine it too, add a recipient for it to `miner.json`.

## Unhost

```bash
lattice child unhost Nexus/Alpha --root R
```

- **The chain's data is deleted.** Hosting it again syncs it from peers.
- A chain with a hosted chain beneath it stays hosted, and the command says
  so (`no longer listed; still hosted for Nexus/Alpha/Beta`). It stops once
  those are unhosted too.
- A chain that another configuration fragment lists is refused, naming the
  fragment.
- Peers and windows set for the chain stay in the configuration, inert
  until it is hosted again.
- A chain you created keeps its spec under `specs/`: `lattice child host`
  brings it back. Running `child create` again with another spec is
  refused.

## The API behind the commands

Prefer the commands. The routes, on the operator port with the cookie:

| Route | Does |
|---|---|
| `GET /hosted-chains` | `{"chains": […], "configured": […]}`: the child chains running, and those the configuration hosts. Read-only. |
| `POST /chains` | Creates a chain once: `{path, spec, premineTo?, genesisTransactions?, policyModules?}`. It does not host it. |
| `PUT /config/fragments/{name}` | Replaces one configuration fragment. A chain is hosted when a fragment lists it in `hostedChains`. |

```bash
C="$(cat R/chains/Nexus/.cookie)"
curl -s --user "$C" http://127.0.0.1:8080/hosted-chains | jq
```

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
`publicSubmit` (all with `lattice config set`), then run `lattice restart`.
Verify through a
parent host's `/api/chain/endpoints?chainPath=<your path>`. The full steps
are in the `lattice-node-operator` skill.

## Stuck at `awaitingGenesis`

Check these in order:

1. Every ancestor is hosted (`lattice status`).
2. The spec exists in `specs/` and decodes.
3. The immediate parent has executed at least one **non-genesis** block.
4. The parent actually changes state. That needs a positive reward **with a
   recipient**, or state-changing transactions. A parent whose rewards burn
   and which gets no transactions can never carry a nested genesis.
5. A miner is running with recipients for the parent and the child
   (`lattice mine status`).
6. That miner mines through a node that holds the spec.

If you only *host* the chain (no spec), it stays in `awaitingGenesis` until
a peer serves its blocks. Check peers and add seeds for it under `peers`.

## Committing a child block yourself

Normally a miner that hosts the child carries its blocks. For a chain whose
blocks should be carried by the parent's miners that do not host it:

```bash
lattice child commit Nexus/Alpha --root R --key ./wallet.json [--fee N]
```

It builds the child's next block, commits it on the parent in a
transaction, and pushes the block until it is in the chain's tree. `--key`
signs and pays the commit on the parent and is paid the child block's
reward. Details: https://lattice.build/docs/run/child-chains/.

## Gotchas

- Creating a chain twice at one path, on two hosts, makes two competing
  chains. Coordinate, or let fork choice pick a winner.
- Losing `specs/` (or its `.wasm` files) can make the node fail at start with
  `missingMaterializedVolume`. Back up `specs/` along with the configuration.
- Unhosting deletes the chain's data. If this node is the chain's only
  holder, the chain is gone.
- Keep path components short ASCII names without `/`. Paths are
  case-sensitive.
- A `chainPath` that isn't hosted returns 404 on every read and on
  transaction submits.
