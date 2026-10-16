---
name: lattice-concepts
description: Mental model of the Lattice protocol and lattice-node — Nexus and child chains, chain paths, merged mining, fork choice, ASERT difficulty, fees and rewards, cross-chain exchange, WASM policies, and what Lattice can and cannot do. Use when a user asks how Lattice works, whether something is possible on Lattice, how it compares to other chains, or when you need the right vocabulary before helping with node, mining, transaction, or app work.
---

# Lattice concepts

Lattice is a **hierarchical proof-of-work protocol**, not a single blockchain.
There is exactly one root chain, **Nexus**. Every other chain is a **child**
named by an absolute path that starts with `Nexus` (`Nexus/Payments`,
`Nexus/Payments/Rollups`). One hash attempt can secure many chains at once
(merged mining). The data is recursive but the authority is not: each chain
validates its own blocks, owns its own parameters, and runs its own fork choice.

`lattice-node` is the full node. One process hosts one **Nexus-rooted tree**
of chains that the operator picks. Mining is external: the node serves block
templates, and separate `lattice-mining-coordinator` / `lattice-miner`
processes search nonces.

Written against lattice-node `main` (Oct 2026) and Lattice 45.2.0. If what you
see on a user's machine disagrees (help text, error strings), trust the
machine and say so.

## The model in ten facts

1. **One root.** Only Nexus is a root chain. Its genesis CID is pinned in the
   node binary (`bafyreiggtg4ezifboyekbf4gxst2jr3mpjqcxsbmopy7w6fp46g4ngpgxa`)
   and committed in a Bitcoin `OP_RETURN`. It is never taken from a peer and
   never re-pinned.
2. **Paths are identities.** A chain *is* its absolute path. A **directory**
   is one parent-relative label (`Payments`), not a global name. Paths are
   case-sensitive. `nexus/x`, `/Nexus/x`, `Nexus/` and `Nexus//x` are all
   invalid. Visible ASCII only, no `/` inside a component; keep names short.
3. **Parents commit, children decide.** A parent block's `children` map
   commits child block CIDs. The parent supplies **work** and **state
   continuity** (so withdrawals can trust parent receipts). It never validates
   a child or picks the child's tip. An invalid child cannot hurt its parent
   or its siblings.
4. **Child security is opt-in.** A child gets only the work from hashes whose
   proof path actually commits it, plus work from later parent blocks that
   descend from such a carrier. It does **not** inherit all of Nexus's
   hashpower. A child that nobody mines does not advance.
5. **Anyone can create a child.** There is no deploy transaction and no
   permission. Write a spec, host it, and the genesis gets mined in by the
   next template that carries it. Rival geneses at the same path compete
   through fork choice; a path is not owned.
6. **Fork choice is GHOST on total subtree work.** Starting from genesis, a
   node repeatedly steps to the child block with the most `trueCumWork` (the
   block plus all its descendants, with each hash attempt counted once). On
   an exact tie, the smaller CID wins. First-seen does not matter. **Work
   weighs, validity selects**: an invalid block's work still counts for its
   ancestors, but the node never steps into it. **Unavailable is not
   invalid**: a missing body is retried and is never a verdict. **There is no
   finality**: a strictly heavier subtree can reorg at any depth.
7. **The node acts on its heaviest executed tip.** A header that is weighed
   but not yet executed is never served as head or built on.
8. **Difficulty is ASERT.** It follows an absolute schedule anchored at block
   1, with no clamp, no floor and no emergency rules. A child genesis uses
   the easiest possible target; block 1 sets the real starting difficulty.
9. **There is no fee field.** A transaction's fee is its *balance surplus*:
   (debits + withdrawals) − (credits + deposits). The block's
   `rewardRecipient` gets reward + fees. If that field is empty, they are
   **burned**.
10. **Policies are validators, not smart contracts.** A chain can name WASM
    modules in its immutable spec. They only accept or reject a transaction
    or action. They cannot hold funds, call each other, or change state.
    Automation lives in apps.

## How the software fits

| Component | Does | Never does |
|---|---|---|
| Lattice (library) | validation, state transition, work, fork choice | networking, storage, RPC |
| lattice-node | sync, storage (`state.db` + `volumes.db`), mempool, templates, RPC | redefine consensus, run a nonce loop |
| Ivy | authenticated P2P transport, discovery | validate content |
| cashew / VolumeBroker | content-addressed tries, Volume storage | decide validity |
| Tally | local peer reputation | protocol validity |
| coordinator | fetch templates, split nonce ranges, submit | build blocks |
| worker (`lattice-miner`, GPU) | nonce search only | see keys or topology |

One process means one identity key, one P2P port, one store and one RPC for
the whole tree. There are no per-chain processes, ports or databases. **To
host `Nexus/A/B`, the same process must also host `Nexus` and `Nexus/A`**,
because a child trusts only the parent it can check for itself.

## What users can do

- Run a full node that follows Nexus (`lattice init`, `lattice up`).
- Create a child chain at any depth with custom block time, reward, premine,
  limits and WASM policies (see the `lattice-child-chains` skill).
- Store app data in a chain's key–value state (`generalState`) using
  compare-and-swap actions, with rules enforced by a WASM policy.
- Pin app content (images, files) by CID on their own node and fetch others'.
- Exchange value between a parent and its **direct** child with
  deposit → receipt → withdraw (see `lattice-transactions`).
- Mine Nexus and every hosted child at once, with a CPU or any worker binary
  that follows the contract (see `lattice-mining`).
- Run public read and submit endpoints and get listed by explorers without
  asking anyone.
- Build apps that read verified state by CID and write through a wallet (see
  `lattice-app-builder`).

## Hard limits — say these plainly

- No smart contracts, contract-held funds or on-chain automation.
- A `ChainSpec` (parameters and policies) is **immutable**. To change rules,
  launch a new chain at a new path.
- Cross-chain exchange works only between a parent and its direct child. A
  deposit has **no timeout, cancel or refund**: without a matching receipt
  and withdrawal it stays locked forever. Moving between a grandchild and
  Nexus takes two separate exchanges.
- No finality and no standard confirmation depth.
- No storage migrations. An incompatible release is a **flag day**: wipe the
  chain data, keep identity and config, and resync.
- No `assumevalid`. New nodes re-execute from genesis.
- Data availability is the operators' choice: a chain lives only while
  someone stores and serves it.
- Policy execution has no fuel or gas bound. A node that can't afford a
  chain's policies simply doesn't host it.

## Misconceptions to correct

Read `references/misconceptions.md` when a user's framing sounds like
Ethereum, Bitcoin sidechains, Cosmos zones or rollups. The most common ones:
"the parent validates the child", "children inherit Nexus security", "N
confirmations is final", "transactions have a fee field", "a cross-chain
transfer can be refunded", and "omitting a reward recipient defers the
reward" (it burns it).

## References

- `references/glossary.md` — precise definitions of about 50 terms (carrier,
  grind, Volume, prevState/parentState, attributed work, …).
- `references/economics.md` — the reward schedule, premine, fee rule, Nexus
  parameters, and security economics.
- `references/misconceptions.md` — the full list, with corrections.

Upstream specs: Lattice `docs/spec.md`, `docs/philosophy.md` and
`docs/cross-chain.md` in https://github.com/adalinxx/Lattice; the node's
`docs/` folder in https://github.com/adalinxx/lattice-node.
