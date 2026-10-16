# Misconceptions to correct

| Users often say | What is actually true |
|---|---|
| "Nexus / the parent validates child blocks." | The parent commits the child's bytes and supplies work and state continuity. The child validates itself. |
| "Children inherit all of Nexus's security." | A child gets only the verified work that actually commits it. |
| "The parent's canonical chain decides the child's chain." | Parent canonicity never feeds into child fork choice. Work from a non-canonical carrier still counts. |
| "`parentState` points at the parent block." | It is the carrier's `prevState`, a state CID. |
| "Deploying a child needs a transaction or permission." | Write a spec and host it. The genesis is mined in. |
| "The first chain at a path owns it." | Rival geneses at the same path compete through fork choice. |
| "Policies are smart contracts." | Policies only accept or reject. They cannot hold funds or change state. |
| "Equal-work ties go to the block seen first." | The smaller CID wins. |
| "N confirmations means final." | There is no finality and no standard confirmation depth. |
| "A missing block body means the block is invalid." | Missing data is retried and is never a verdict. |
| "An invalid block's work disappears." | Its work still counts for its ancestors. It is just never selected. |
| "Each chain runs as its own process, port or database." | lattice-node runs one process with one store for the whole tree. |
| "Transactions have a fee field." | The fee is the surplus of debits over credits. |
| "A cross-chain transfer is a bridge and can be refunded." | It is a two-party exchange. A deposit never times out. |
| "I can move coins from a grandchild straight to Nexus." | Only a parent and its direct child can exchange, so that takes two exchanges. |
| "Difficulty uses a retarget window with clamps." | It is ASERT, anchored at block 1, with no clamp and no floor. |
| "`mine.minWork` sets difficulty." | It is a local filter on what the miner searches for. |
| "`blockBodies` and `state` settings change consensus." | They only change what the node stores and serves. |
| "There are several root chains." | Only Nexus is a root. |
| "A nested child can start right away." | It starts only after its parent has executed a block that changed state. |
| "Omitting a reward recipient means the rewards wait." | They are burned. |
| "The node serves the heaviest header it has seen." | It acts on the heaviest *executed* tip. |
| "A 200 from POST /transactions means it will be mined next." | The transaction may be pooled as `future` (nonce gap) or `unavailable`. Confirm inclusion. |
| "I can change my chain's parameters later." | A spec is immutable. Launch a new path instead. |
