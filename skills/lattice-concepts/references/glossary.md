# Lattice glossary

The terms the Lattice documentation uses, defined once. The first section
can be read top to bottom by a newcomer. Later sections go deeper, and the
last two are for people who run a node or implement the protocol. The
[protocol specification](https://lattice.build/docs/protocol/spec/) is the
authority for every consensus term.

## Basic terms

- **Lattice**: a proof-of-work protocol for a tree of chains. One root
  chain, Nexus, and any number of child chains under it.
- **Nexus**: the single root chain. Every chain path begins with `Nexus`
  (case-sensitive). It is the only chain whose first block is fixed in
  advance.
- **Child chain**: any chain other than Nexus. Its blocks are committed
  inside its parent's blocks. It validates its own blocks, has its own coin
  and parameters, and chooses its own tip.
- **Parent**: the chain one level toward Nexus. A chain can be the child of
  one chain and the parent of others.
- **Chain path**: a chain's identity: its full path from Nexus, such as
  `Nexus/Payments/Rollups`.
- **Block**: one step of a chain: a list of transactions, the state before
  and after them, a link to the previous block, and the proof of work that
  secures it.
- **Height**: a block's distance from its chain's first block, which is at
  height 0.
- **Genesis**: a chain's first block, at height 0.
- **Tip**: the last block of the path a chain's fork choice prefers. A node
  acts on its heaviest *executed* tip: a block it has only weighed is never
  served as the head or built on.
- **Transaction**: a signed request to change a chain's state: move coin,
  write key–value data, or take a step in a cross-chain exchange.
- **Address**: an account's name: the CID of its Ed25519 public key. It
  looks like `bafyrei…`.
- **Key**: the Ed25519 private key that signs for an address. Whoever holds
  it controls the account.
- **Signer**: an address that signs a transaction. A debit from an account
  requires that account's signature; a credit does not.
- **Nonce**: a counter that makes each transaction usable once. Only a
  transaction's first signer has a nonce checked and advanced: the
  transaction must carry that signer's next nonce. Co-signers' nonces are
  neither checked nor advanced.
- **Fee**: what a transaction debits beyond what it credits. There is no fee
  field.
- **Mempool** (or pool): a node's own set of transactions waiting to be put
  in a block. It is node policy, never consensus: each node decides what it
  admits, for example with a minimum fee (`minRelayFee`).
- **Mining**: searching for a nonce whose hash meets a block's target. The
  search is the proof of work.
- **Reward**: the new coin a block creates, by a schedule fixed in the
  chain's spec.
- **Coinbase**: a block's reward plus the fees of its transactions. It is
  credited to the block's reward recipient. With no recipient it is burned.
- **Reward recipient**: one address per block, committed in the header. It
  needs no signature and no key on the mining host.
- **Reorganization** (reorg): a chain's tip moving to a different branch
  because that branch became heavier. It is local to one chain.
- **Finality**: Lattice has none. A strictly heavier branch may reorganize a
  chain at any depth, and there is no standard confirmation depth. A node's
  operator may set a node-local finality window; that is one node's policy
  and not a protocol rule.
- **CID**: a content identifier: a hash of some bytes. Blocks, transactions,
  specs, state and policy modules are all named by CID, so anyone may supply
  the bytes and the receiver checks them.
- **Volume**: one root CID plus the exact `(CID, bytes)` entries published
  with it. It is the unit in which content is stored, transferred and
  pinned.
- **State**: what a chain holds after a block: balances, nonces, key–value
  data, and the records of cross-chain exchanges.
- **Node**: a program that follows chains, validates their blocks and serves
  them. `lattice-node` is the full node.

## Chains

- **Uses of a chain path**: besides naming the chain, the path protects
  transactions from replay (`body.chainPath`), routes messages between
  nodes, keys storage, routes rewards, and selects a chain on a node's read
  routes (`?chainPath=`).
- **Directory**: one edge label under a parent: the name under which a
  parent block's `children` map commits a child block. It is relative to the
  parent (two parents may each have a child called `Payments`) and is not a
  field of the child's spec.
- **ChainSpec**: the parameters a chain is created with:
  `maxNumberOfTransactionsPerBlock`, `maxStateGrowth`, `maxBlockSize`,
  `premine`, `targetBlockTime` (milliseconds), `initialReward`,
  `halvingInterval`, `halfLife` (blocks) and `wasmPolicies`. Every numeric
  field except `premine` must be above 0. It cannot be changed. See the
  [ChainSpec reference](../../lattice-child-chains/references/chainspec.md).
- **Validity policy** (WasmPolicy): a WebAssembly module a spec names by
  CID, with scope `transaction` or `action`. It returns 1 to accept;
  anything else, or a trap, rejects. See
  [Writing a validity policy](../../lattice-app-builder/references/policy.md).
- **Premine**: the first N scheduled rewards, paid at genesis to an account
  the chain's creator names.
- **Genesis rules**: a genesis has an empty `prevState` and no reward
  recipient. A child genesis is built on real, executed parent state, so a
  nested child can start only after its parent has executed a block of its
  own.

## Blocks and state

- **Block fields**: `(version, previousBlock, transactions, target,
  nextTarget, spec, parentState, prevState, postState, children, height,
  timestamp, rewardRecipient, nonce)`.
- **prevState / postState**: the chain's state before and after the block.
- **parentState**: for a child block, the `prevState` of the parent block
  that carries it. It is a state CID, never a pointer to a parent block.
- **LatticeState**: four content-addressed compressed radix tries (not
  sparse Merkle trees: no fixed depth, no default hashes):
  - `accountState`: address → balance, plus nonces under `_nonce_<signer>`
    keys.
  - `generalState`: string → string, the application key–value store.
  - `depositState`: `demander/amountDemanded/nonce` → locked amount.
  - `receiptState`: `directory/demander/amountDemanded/nonce` → withdrawer.
- **Action**: a key–value write, `{key, oldValue?, newValue?}`. It applies
  only if the current value equals `oldValue`; with no `oldValue` the key
  must not exist yet. With no `newValue` the key is deleted. Keys are
  non-empty visible ASCII. An insert costs `len(key) + len(value)` bytes of
  the block's `maxStateGrowth`. Consensus lets any signer write any key;
  only a validity policy restricts that.
- **Witness**: the trie nodes on a key's path. They prove, against a state
  root that a block commits, that the key holds a value or is absent. There
  is no separate proof object.
- **Pin**: an operator's instruction to keep one Volume. Pins are flat:
  pinning a Volume does not pin the Volumes it links to.

## Transactions and cross-chain exchange

- **Transaction body**: `accountActions, actions, depositActions,
  receiptActions, withdrawalActions, signers, nonce, chainPath`. See
  [Transaction format](../../lattice-transactions/references/transaction-format.md).
- **Signature**: Ed25519 over a domain-separated envelope: `lattice-tx-v1:`
  with the chain path, the nonce and the body's CID. The signing keys'
  addresses must equal `signers` exactly.
- **Replay protection**: a transaction names its `chainPath` in the signed
  data, so it is valid on that chain only.
- **`minRelayFee`**: the smallest fee one node admits to its pool. A local
  floor, not a consensus rule.
- **Deposit / receipt / withdrawal**: the three steps of a two-party
  exchange between a chain and its direct child. The demander locks coin on
  the child (deposit); the withdrawer pays the demander on the parent
  (receipt), then claims the locked coin on the child (withdrawal). See
  [Cross-chain transfers](https://lattice.build/docs/protocol/cross-chain/).

## Mining and work

- **Proof-of-work hash**: `SHA256(prefix || uint64BE(nonce))`. The prefix
  covers every consensus field, `rewardRecipient` included, so a reward
  cannot be redirected after the fact.
- **Target**: the number a block's hash must not exceed. A smaller target is
  harder.
- **Work**: `floor(2^256 / (target + 1))`, the chainwork formula Bitcoin
  uses.
- **Merged mining**: one search secures many chains. A template carries
  Nexus and the latest candidate block of each hosted child, recursively,
  and each hash is checked against every target it carries. A hash that
  misses Nexus's target can still make a valid block for an easier child.
- **Carrier**: a parent-level block that commits child blocks in
  `children`, even if its own hash missed the parent's target. A carrier
  need not be valid or canonical on its own chain for the work it carries to
  count for the child.
- **Grind**: one physical proof-of-work attempt, identified by its root
  CID.
- **Mined root**: the outermost block whose hash is evaluated along a proof
  path.
- **ChildBlockProof**: the proof from a mined root down the directory path
  to one child block. A child's work is derived from it.
- **Attributed run work**: later parent blocks that descend from a carrier
  (by parent pointer, not by canonicity) also add work to the child block it
  carried. Each grind is counted once.

## Fork choice

- **Weighed / validated / canonical**: a block is *weighed* once its work
  has been verified from its header and proof; *validated* once it has been
  executed and its `postState` matches; *canonical* when it is on the
  preferred path.
- **Accepted**: locally validated consensus evidence kept in the forest.
- **Forest**: a chain path's accepted blocks. It can have several roots,
  because any genesis carried by a valid proof competes; only Nexus fixes
  one genesis.
- **Chain root**: a genesis block in a chain path's forest.
- **Predecessor**: the previous block on the same chain.
- **trueCumWork**: the total work of a block plus all its same-chain
  descendants, with each grind counted once.
- **Hierarchical GHOST**: the fork-choice rule. Starting from genesis, step
  to the child block with the greatest `trueCumWork`; on an exact tie, take
  the smaller CID. Each chain runs its own, and work flows into it from
  grinds on its ancestors.

## Difficulty

- **ASERT**: the difficulty rule.
  `nextTarget = anchor.target · 2^(drift / halfLife)`. The anchor is block 1
  on the block's own branch. `drift` is the actual elapsed time minus the
  scheduled time. `halfLife` is given in blocks and multiplied by
  `targetBlockTime`.
- **Scheduled target**: a block may be harder than scheduled
  (`target <= parent.nextTarget`), never easier.
- **Timestamp rule**: `parent.timestamp < B.timestamp <= now`. A block from
  the future is held until its time comes, not rejected.

## Running a node

These terms belong to `lattice-node`. The operator documentation starts at
[Getting started](https://lattice.build/docs/run/getting-started/).

- **Hosted chains**: the chains one node process serves. Nexus is always
  one of them, and a hosted child's ancestors are hosted with it.
  `hostedChains` in the node's configuration lists the child paths.
- **Level**: one hosted chain inside the node process. Each level has its
  own consensus state, fork choice, sync, mempool and mining candidates.
- **Data directory**: where a node keeps its chains. `state.db` holds the
  facts for every chain (headers, proofs, indexes, the mempool journal);
  `volumes/` holds one content store per hosted chain. The two are one
  recovery unit: back them up and restore them together, only while the node
  is stopped. See [Storage](https://lattice.build/docs/run/storage/).
- **Retention**: `blockBodies` (`all` or `none`) and `state` (`all`,
  `recent` or `none`). They change what a node stores and serves, never
  which tip it picks.
- **Rendezvous**: each hosted child chain is announced under a key derived
  from its path, so a node joining the chain can find its hosts before it
  holds any block.
- **Public read listener**: an optional, rate-limited listener that serves
  only the read routes. `publicReadURL` declares an `https` URL that the
  parent chain's hosts relay at `/api/chain/endpoints`. Readers must verify
  what they get from it.
- **Miner settings**: `miner.json`, read by the mining loop and never by the
  node. `recipients` names the address each chain's coinbase goes to.
- **`minWork`**: a miner's local filter on which hashes it submits. It never
  changes a block's scheduled target.
- **Template digest**: a fingerprint of every hosted level's candidate. A
  new transaction or candidate at any level changes it, and the miner then
  fetches new work.
- **Wipe**: deleting a node's chain storage and syncing again. A wipe keeps
  the identity, the configuration and the specs.
- **Flag day**: a release that changes the storage schema, a consensus rule
  or the wire format with no migration. Every node installs the release,
  and one whose stored data it cannot read is wiped. An addition to the wire
  that is gated on a capability does not need one.
