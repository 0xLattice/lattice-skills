# ChainSpec reference

A ChainSpec is the set of parameters a chain is created with: its block
time, rewards, block limits and validity policies. This page lists every
field. A spec is fixed at the chain's genesis and cannot be changed
afterwards; to change a rule, create a chain at a new path.

## A complete spec file

`lattice child create <path> --spec <file>` takes a spec as JSON. The file
must contain **every** field: the decoder has no defaults.

```json
{
  "maxNumberOfTransactionsPerBlock": 1000,
  "maxStateGrowth": 1000000,
  "maxBlockSize": 1000000,
  "premine": 0,
  "targetBlockTime": 10000,
  "initialReward": 1000,
  "halvingInterval": 1000000,
  "halfLife": 100,
  "wasmPolicies": []
}
```

The keys `transactionFilters` and `actionFilters` are refused.

## Fields

Without `--spec`, `lattice child create` builds a spec from its flags and
the defaults in the last column.

| Field | Unit | Meaning | `child create` default |
|---|---|---|---|
| `maxNumberOfTransactionsPerBlock` | count, above 0 | Transactions per block. | 1000 |
| `maxStateGrowth` | bytes, above 0 | State a block may add. Inserting a key costs `len(key) + len(value)`. | 1,000,000 |
| `maxBlockSize` | bytes, above 0 | Unique content bytes a block owns: the block and its transactions. | 1,000,000 |
| `premine` | blocks, 0 or more | The first N scheduled rewards, paid at genesis to the `--premine-to` account. It also shifts the halving curve. | 0 (`--premine`) |
| `targetBlockTime` | **milliseconds**, above 0 | The spacing the difficulty schedule aims for. | 10000 (`--block-time`) |
| `initialReward` | coin units, above 0 | The reward at height 0, before the premine shift. | 1000 (`--reward`) |
| `halvingInterval` | blocks, above 0 | Blocks per halving of the reward. | 1,000,000 |
| `halfLife` | **blocks**, above 0 | How fast difficulty reacts: a drift of `halfLife × targetBlockTime` milliseconds moves the target by one doubling. Smaller reacts faster. | 100 |
| `wasmPolicies` | array | Validity policy modules ([below](#validity-policies)). | `[]` |

Fields without a flag can be set only with a full `--spec` file.

The premine amount is `Σ_{i<premine} initialReward >> (i / halvingInterval)`,
which is `premine × initialReward` while `premine < halvingInterval`.

## What the creator does not choose

- **The genesis target.** It is the maximum (easiest), and block 1 sets the
  real difficulty.
- **The genesis `parentState`.** It is the state the carrying parent block
  starts from, which is why a nested child needs a parent that has executed
  a block of its own.
- **A genesis reward recipient.** The genesis has none.
- **The directory.** It is the last component of the chain's path and is
  not part of the spec.

## Validity policies

Each `wasmPolicies` entry names one WebAssembly module that accepts or
rejects transactions on the chain:

```json
{
  "moduleCID": "bafy…",
  "abiVersion": 1,
  "scope": "transaction",
  "entrypoint": "lattice_validate_transaction"
}
```

| Key | Meaning |
|---|---|
| `moduleCID` | The CID of the module's bytes. |
| `abiVersion` | `1`. |
| `scope` | `transaction`: the policy sees the whole transaction body, once. `action`: it is called once per key–value action. |
| `entrypoint` | The name of the function to call. **Always include it**: the decoder has no default. By convention `lattice_validate_transaction` or `lattice_validate_action`. |
| `sourceCID` | Optional. |

Every module a spec names is validated at genesis, and each must be passed
to `lattice child create` with `--policy-module`.

What a module must export, what it receives, and how to write, build and
test one are in
[Writing a validity policy](../../lattice-app-builder/references/policy.md).
