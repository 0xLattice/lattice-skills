# Lattice glossary

## Chains and addressing

- **Nexus**: the single root chain. Every absolute chain path begins with
  `Nexus` (case-sensitive). It is the only chain with a pinned genesis.
- **Chain path**: a chain's identity, i.e. its full path from Nexus, such as
  `Nexus/Payments/Rollups`. It is used for transaction replay protection
  (`body.chainPath`), wire routing, storage keys, reward routing and the
  `?chainPath=` query on reads.
- **Directory**: one parent-relative edge label, the key under which a parent
  block's `children` map commits a child block. It is not global (two
  different parents may each have a child called `Payments`) and it is not a
  spec field.
- **Parent / child**: the parent is the next level toward Nexus. A chain can
  be a child of one chain and the parent of others.
- **Hosted tree / hosted chains**: the set of chain paths one node process
  serves. Nexus is always included. `hostedChains` in `lattice.json` lists
  the child paths with each parent before its children, and every ancestor
  must be listed. The set can be changed while the node runs.
- **Level**: one hosted path inside the node process. Each level has its own
  consensus state, fork choice, sync, mempool and mining candidates.

## Blocks, content, storage

- **Block**: `(version, previousBlock, transactions, target, nextTarget, spec,
  parentState, prevState, postState, children, height, timestamp,
  rewardRecipient, nonce)`.
- **prevState / postState**: the chain's state before and after the block.
- **parentState**: for a child block, this is the *carrier's* `prevState`. It
  is a state CID, never a pointer back to a parent block.
- **Carrier**: a parent-level block that commits child blocks in `children`.
  This holds even if its own hash missed the parent's target. A carrier does
  not need to be valid or canonical on its own chain for the work it carries
  to count for the child.
- **Grind**: one physical proof-of-work attempt, identified by its root CID.
- **Mined root**: the outer block whose hash is evaluated along a proof path.
- **Predecessor**: the previous block on the same chain.
- **Chain root**: a genesis block in a chain path's accepted forest.
- **Forest**: a chain path's accepted same-chain blocks. It can have several
  roots, because any genesis carried by a valid proof competes; only Nexus pins
  one genesis.
- **CID**: content identifier, DAG-CBOR + SHA-256. Blocks, transactions,
  specs, state and WASM modules are all addressed by CID. Any provider may
  supply the bytes, because the consumer re-hashes them.
- **Address**: `CID(PublicKey{key: <Multikey hex of an Ed25519 key>})`. It
  looks like `bafyrei…`.
- **Volume**: one root CID plus the exact `(CID, bytes)` entries published
  with it. It is the unit of storage, transfer and pinning. Pins are flat:
  pinning a Volume does not pin the Volumes it links to.
- **state.db / volumes.db**: the node's single recovery unit. `state.db` holds
  facts, headers, proofs, indexes and the mempool journal. `volumes.db` holds
  content. Back them up together, only while the node is stopped, and never
  mix snapshots. `volume-bundles.db` is a cache that can be deleted.
- **Retention**: `blockBodies: all|none` and `state: all|tip|none`. These
  change what a node stores and serves, never which tip it picks.

## State

- **LatticeState**: four sparse Merkle tries:
  - `accountState`: address → balance, plus per-signer nonces.
  - `generalState`: string → string, the app key–value store.
  - `depositState`: `demander/amountDemanded/nonce` → locked amount.
  - `receiptState`: `directory/demander/amountDemanded/nonce` → withdrawer.
- **Action (KV)**: `{key, oldValue?, newValue?}`. It applies only if the
  current value equals `oldValue`; an absent `oldValue` means the key must
  not exist yet. An absent `newValue` deletes the key. Keys are non-empty
  visible ASCII. An insert costs `len(key)+len(value)` bytes of the block's
  `maxStateGrowth`. Consensus lets any signer write any key; only a chain
  policy can restrict that.

## Mining and work

- **PoW hash**: `SHA256(prefix || uint64BE(nonce))`. The prefix covers every
  consensus field, including `rewardRecipient`, so a reward cannot be
  redirected after the fact.
- **Work**: `floor(2^256 / (target+1))`, the same chainwork formula Bitcoin
  uses.
- **Merged mining**: one template carries Nexus plus the latest candidate of
  each hosted child, recursively. Each grind is checked against every target
  it carries. A hash that misses Nexus can still be a valid block for an
  easier child; the Nexus block is then just a carrier.
- **ChildBlockProof**: the proof from a mined root down the directory path to
  one child block. Child work is derived from this proof.
- **Attributed run work**: later parent blocks that descend from a carrier
  (by parent pointer, not by canonicity) also add work to the child block it
  carried. Each grind is counted once.
- **Reward recipient**: one address per chain, committed in the header. It
  needs no signature and no key on the mining host. If it is missing, that
  chain's reward and fees are burned.
- **minWork**: a miner-local search filter. It never changes consensus.
- **Template digest**: a fingerprint of every hosted level. Any new
  transaction or candidate at any level makes outstanding work stale.

## Fork choice

- **Weighed / validated / canonical**: *weighed* means the work has been
  verified from the header and proof. *Validated* means the block has been
  executed and its `postState` matches. *Canonical* means it is on the
  preferred path. A node never acts on a tip that is only weighed.
- **Accepted**: locally validated consensus evidence retained in the forest.
- **trueCumWork**: the total work of a block plus all its same-chain
  descendants, with work de-duplicated per grind.
- **Hierarchical GHOST**: starting from genesis, step to the child with the
  greatest `trueCumWork`; on an exact tie, take the lexicographically smaller
  CID. Each chain runs its own GHOST, and work flows into it from ancestor
  grinds.
- **Reorg**: local to one chain. A reorg on a parent sends no command to its
  children.

## Difficulty

- **ASERT**: `nextTarget = anchor.target · 2^(drift / halfLife)`. The anchor is
  block 1 on the block's own branch. `drift` is actual elapsed time minus
  scheduled time. `halfLife` is given in blocks and multiplied by
  `targetBlockTime`.
- A block may be harder than scheduled (`target <= parent.nextTarget`), never
  easier.
- Timestamps must satisfy `parent.ts < B.ts <= now`. A block from the future
  is held, not rejected.

## Transactions and cross-chain

- **Transaction**: signatures plus a body with `accountActions, actions,
  depositActions, receiptActions, withdrawalActions, signers, nonce,
  chainPath`.
- **Signature**: Ed25519 over the domain-separated envelope
  `lattice-tx-v1:` + chain path + nonce + body CID. The signer set must equal
  `signers` exactly. A debit requires the owner's signature; a credit does
  not.
- **Nonce**: one per signer, contiguous. A transaction with several signers
  advances every signer's nonce.
- **Fee**: the balance surplus. The node's `minRelayFee` is a local policy
  floor, not a consensus rule.
- **Deposit / receipt / withdrawal**: the two-party parent↔child exchange.
  See the `lattice-transactions` skill.

## Policies, genesis, operations

- **WasmPolicy**: a module referenced by CID in `ChainSpec.wasmPolicies`,
  with scope `transaction` or `action`. It returns 1 to accept; anything else,
  or a trap, rejects. It must export `memory`, `lattice_alloc(i32)->i32` and
  an entrypoint `(i32,i32)->i32`.
- **ChainSpec**: `maxNumberOfTransactionsPerBlock, maxStateGrowth,
  maxBlockSize, premine, targetBlockTime (ms), initialReward,
  halvingInterval, halfLife (blocks), wasmPolicies`. Every numeric field
  except `premine` must be > 0. It is immutable.
- **Genesis**: height 0, with empty `prevState` and no reward recipient. A
  child genesis needs a real executed parent state, so a nested child can
  start only after its parent has executed a block of its own.
- **Flag day**: a schema, consensus or wire cutover. Stop the node, keep the
  identity, `lattice.json` and `specs/`, run `lattice wipe`, install all
  binaries from one release, and restart. Additive wire changes that are
  gated on a capability don't need one.
- **Rendezvous**: each hosted child is announced in the DHT under a key
  derived from its path, so a node joining a child can find providers before
  it holds any block.
- **Public read port / URL**: an opt-in, rate-limited, GET-only listener.
  `publicReadURL` declares an https URL that parent hosts relay at
  `/api/chain/endpoints`. Readers must verify what they get from it.
