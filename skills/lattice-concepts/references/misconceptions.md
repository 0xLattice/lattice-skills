# Common misconceptions about Lattice

Lattice looks like other systems in places and behaves differently. Each
row gives a common assumption, often carried over from Bitcoin sidechains,
Ethereum, Cosmos zones or rollups, and what is true instead. Terms are
defined in the [glossary](glossary.md).

## Chains and security

| Misconception | What is true |
|---|---|
| There are several root chains. | Only Nexus is a root. |
| Nexus, or a parent, validates its children's blocks. | A parent commits the child block's CID and supplies work and state continuity. The child validates itself. |
| A child chain inherits all of Nexus's security. | A child gets only the verified work of the hashes that actually commit it. A child nobody mines does not advance. |
| The parent's canonical chain decides the child's chain. | Parent canonicity never feeds into the child's fork choice. Work from a non-canonical carrier still counts. |
| `parentState` points at the parent block. | It is the carrying parent block's `prevState`: a state CID. |
| Creating a child chain needs a transaction or someone's permission. | Write a spec and host it. The genesis is mined in by the next template that carries it. |
| The first chain at a path owns that path. | Rival geneses at the same path compete through fork choice. A path is not owned. |
| A nested child can start right away. | It starts only after its parent has executed a block of its own, which needs the parent to change state. |
| A chain's parameters can be changed later. | A spec is immutable. To change a rule, launch a chain at a new path. |

## Fork choice and finality

| Misconception | What is true |
|---|---|
| Equal-work ties go to the block seen first. | The smaller CID wins. |
| N confirmations means final. | There is no finality and no standard confirmation depth. A strictly heavier branch can reorganize a chain at any depth. |
| A missing block body means the block is invalid. | Missing data is retried and is never a verdict. |
| An invalid block's work disappears. | Its work still counts for its ancestors. The block is just never selected. |
| A node serves the heaviest header it has seen. | It acts on its heaviest *executed* tip. |
| Difficulty uses a retarget window with clamps. | It is ASERT, anchored at block 1, with no clamp and no floor. |

## Transactions and fees

| Misconception | What is true |
|---|---|
| Transactions have a fee field. | The fee is what a transaction debits beyond what it credits. |
| A submit the node accepts will be in the next block. | It is pooled. It can still be outbid, replaced or left out, and a block that includes it can be reorganized away. Confirm inclusion by reading it back. |
| A transaction with a nonce ahead of the account's waits in the pool. | The node refuses it (`nonceAhead`). Send it again once the earlier transaction is mined. |
| A node's minimum fee is a rule of the chain. | `minRelayFee` is one node's policy. Other nodes choose their own. |
| Omitting a reward recipient makes the reward wait. | The reward and fees are burned. |

## Cross-chain exchange

| Misconception | What is true |
|---|---|
| A cross-chain transfer is a bridge and can be refunded. | It is an exchange between two parties. A deposit has no timeout, cancel or refund. |
| Coin can move from a grandchild chain straight to Nexus. | Only a chain and its direct child can exchange, so that takes two exchanges. |

## Policies and applications

| Misconception | What is true |
|---|---|
| Policies are smart contracts. | A policy only accepts or rejects a transaction. It cannot hold funds, change state or call another policy. |
| The chain decides who may write a key. | Consensus lets any signer write any key. Only a chain's validity policy restricts that. |

## Running a node

| Misconception | What is true |
|---|---|
| Each chain runs as its own process, port or database. | `lattice-node` runs one process with one data directory and one overlay port for the whole tree of chains it hosts. |
| The miner's `minWork` sets a chain's difficulty. | It is a local filter on which hashes the miner submits. Blocks still commit their scheduled target. |
| The `blockBodies` and `state` settings change consensus. | They change only what the node stores and serves, never which tip it picks. |
