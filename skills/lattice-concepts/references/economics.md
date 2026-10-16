# Lattice economics

## Rules every chain follows

- **Block reward**: `reward(h) = initialReward >> ((h + premine) / halvingInterval)`.
  The reward eventually reaches 0. There is no tail emission.
- **Premine**: the sum of the first `premine` scheduled rewards, which
  equals `premine × initialReward` when `premine < halvingInterval`. The
  genesis pays it as one unsigned credit to the account named at creation
  (`--premine-to`). The premine also shifts the halving curve forward.
- **Fee rule**: in every non-genesis block,
  `credits + deposited <= debits + withdrawn`. The difference `F` is the fee
  total. The coinbase pays `reward(h) + F` to the header's `rewardRecipient`.
  If the recipient is missing, the coinbase is burned. There is no reward
  transaction and no fee field.
- **Receipts** are balance-neutral (the withdrawer is debited and the
  demander credited the same amount), so they add nothing to `F`.
- **Ranking** pooled transactions by fee rate and the `minRelayFee` floor
  are node policy, not consensus. Pooled transactions never expire with
  time.
- Policies see transactions, not the coinbase, so a policy cannot govern
  rewards.
- Every chain has its own coin. In a cross-chain exchange, the two parties
  set the rate themselves (`amountDemanded` on the parent and
  `amountDeposited` on the child).

## Nexus parameters

| Parameter | Value |
|---|---|
| `targetBlockTime` | 3,600,000 ms (1 hour) |
| `initialReward` | 1,048,576 (2^20) |
| `halvingInterval` | 876,600 blocks (about 100 years) |
| `premine` | 175,320 blocks (about 10% of supply) |
| `halfLife` | 120 blocks (about 5 days) |
| `maxBlockSize` | 1,000,000 bytes |
| `maxStateGrowth` | 3,000,000 bytes per block |
| `maxNumberOfTransactionsPerBlock` | 5,000 |
| Total supply | 1,838,362,566,600 |

Check these against the live node with
`curl -s localhost:8080/api/chain/spec --user "$(cat <root>/chains/Nexus/.cookie)"`.

## `lattice child create` defaults

Without `--spec`, the defaults are:

| Field | Default | Flag |
|---|---|---|
| `targetBlockTime` | 10,000 ms | `--block-time` |
| `initialReward` | 1,000 | `--reward` |
| `premine` | 0 | `--premine` |
| `maxNumberOfTransactionsPerBlock` | 1,000 | none |
| `maxStateGrowth` | 1,000,000 | none |
| `maxBlockSize` | 1,000,000 | none |
| `halvingInterval` | 1,000,000 | none |
| `halfLife` | 100 | none |

Fields without a flag can only be set with a full `--spec` file.

## Security economics

- Reorging k blocks costs roughly `cost-per-hash × hashes-per-block × k`.
- Lattice's simulator finds deep reorgs likely only near or above 50% of the
  hashpower. Selfish mining becomes profitable above about 25% (γ = ½,
  because the smaller-CID tie-break settles equal-work races). Plan a chain's
  security budget against the 25% threshold.
- A child is only as secure as the hashpower of the miners who choose to
  carry it.
