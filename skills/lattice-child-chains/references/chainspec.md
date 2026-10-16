# ChainSpec reference

A spec file passed with `--spec` must contain **every** field, because the
decoder has no defaults. The legacy keys `transactionFilters` and
`actionFilters` are rejected.

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

| Field | Unit / type | Meaning | `child create` default |
|---|---|---|---|
| `maxNumberOfTransactionsPerBlock` | count > 0 | transactions per block | 1000 |
| `maxStateGrowth` | bytes > 0 | sum of state growth per block. A KV insert costs `len(key)+len(value)`. | 1,000,000 |
| `maxBlockSize` | bytes > 0 | unique content bytes a block owns (the block plus its transactions) | 1,000,000 |
| `premine` | blocks ≥ 0 | the first N scheduled rewards, paid at genesis to `--premine-to`. Also shifts the halving curve. | 0 (`--premine`) |
| `targetBlockTime` | **ms** > 0 | ASERT target spacing | 10000 (`--block-time`) |
| `initialReward` | units > 0 | reward at height 0 (before the premine shift) | 1000 (`--reward`) |
| `halvingInterval` | blocks > 0 | blocks per halving | 1,000,000 |
| `halfLife` | **blocks** > 0 | ASERT responsiveness: a drift of `halfLife × targetBlockTime` ms moves the target by one doubling | 100 |
| `wasmPolicies` | array | optional validity modules | `[]` |

Premine amount = `Σ_{i<premine} initialReward >> (i / halvingInterval)`,
which is `premine × initialReward` while `premine < halvingInterval`.

The deployer does not choose these:

- The genesis target is the maximum (easiest), and block 1 sets the real
  difficulty.
- The genesis `parentState` is the carrier's entering state, which is why a
  nested child needs a parent that has executed a block.
- The genesis has no reward recipient.
- The directory is the last path component; it is not part of the spec.

## WASM policies

Each `wasmPolicies` entry:

```json
{
  "moduleCID": "bafy…",
  "abiVersion": 1,
  "scope": "transaction",
  "entrypoint": "lattice_validate_transaction"
}
```

- `scope` is `transaction` (the policy sees the whole body once) or `action`
  (it is called once per action). **Always include `entrypoint` in the
  JSON**, conventionally `lattice_validate_transaction` or
  `lattice_validate_action`; the decoder has no default. `sourceCID` is
  optional.
- The module must export `memory`, `lattice_alloc(i32) -> i32` and the
  entrypoint `(i32 ptr, i32 len) -> i32`. Return `1` to accept. Any other
  value, or a trap, rejects.
- The policy receives a deterministic binary `LWPCTX` envelope: height,
  timestamp, spec, chain path, and the transaction body (DAG-CBOR) or the
  action. Float and SIMD instructions are rejected, so use integer math only. Rules must not depend on
  anything a miner could vary freely. Prefer height over timestamp, or bound
  timestamps to a window.
- No fuel or memory bound applies beyond the module's own declared maxima.
  The genesis validates every module named in the spec.
- Policies can only accept or reject. They cannot write state, hold funds or
  call other policies.

Practical advice:

- Write policies in Rust `no_std`, target `wasm32-unknown-unknown`.
- Commit the built `.wasm` next to the spec, so hosting the chain doesn't
  require Rust.
- Add a test that rebuilds the module and checks that its CID matches the
  spec.
- Pin the policy crate's dependencies by version or git tag. A sibling
  `path =` dependency won't build in a fresh clone.

A worked example of a policy that enforces key ownership is in the
`lattice-app-builder` skill.
