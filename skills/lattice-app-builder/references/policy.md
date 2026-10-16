# Writing a validity policy

## ABI

- WASM module, scope `transaction` (recommended for apps) or `action`.
- Exports: `memory`, `lattice_alloc(len: i32) -> i32`, and
  `lattice_validate_transaction(ptr: i32, len: i32) -> i32` (scope `action`
  uses `lattice_validate_action`).
- Return `1` to accept. Any other value, or a trap, rejects.
- Input: the binary `LWPCTX` envelope. It holds the magic bytes, an
  encoding version, the ABI version, a scope tag, the block height and
  timestamp, a length-prefixed DAG-CBOR spec, the chain path strings, and the
  optional body, action and action index. Use
  `lattice_policy_rules::transaction_body` to extract the DAG-CBOR body.
  Reject anything you can't parse.
- **Integer math only.** Modules that use float or SIMD/vector instructions
  are rejected by the determinism scan, so `lattice child create` refuses
  them.
- The policy can't change state, read other keys, hold funds or call other
  policies. It can't see the coinbase either.
- There is no fuel limit, but keep the policy fast: every node hosting your
  chain runs it on every transaction.

Reference it from the spec:

```json
"wasmPolicies": [{
  "moduleCID": "<CID printed by your build/check step>",
  "abiVersion": 1,
  "scope": "transaction",
  "entrypoint": "lattice_validate_transaction"
}]
```

## Rust skeleton

The `rules/` crate in https://github.com/adalinxx/moniker provides the ABI
macro and a body parser. Depend on it by **git tag**, not by a path:

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
lattice-policy-rules = { git = "https://github.com/adalinxx/moniker", tag = "<tag>" }

[profile.release]
opt-level = "z"
lto = true
panic = "abort"
```

```rust
#![cfg_attr(not(test), no_std)]
use lattice_policy_rules::{transaction_body, Body};
lattice_policy_rules::lattice_policy_abi!(crate::validate);

pub fn validate(context: &[u8]) -> bool {
    transaction_body(context).and_then(|b| {
        let body = Body::parse(b)?;
        for action in body.actions()? {
            let action = action?;                       // Action { key, old_value, new_value } as &[u8]
            // Only keys shaped user/<owner>/... are allowed, and <owner> must sign.
            let rest = action.key.strip_prefix(b"user/")?;
            let owner = rest.split(|&b| b == b'/').next()?;
            body.is_signer(owner)?;                     // Option<()>: None rejects
        }
        Some(())
    }).is_some()
}
```

The crate also provides helpers for tokens (`check_token_write`,
`credited`). Read `rules/src/lib.rs` at the tag you pin.

Build:

```bash
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/<app>_policy.wasm chain/
```

## Testing

- Unit-test `validate` natively on hand-built contexts: accept cases, every
  reject case, and malformed input.
- Replay the same contexts through Lattice's own policy evaluator (for
  example, a small Swift package that depends on `Lattice` and prints the
  module CID). The public app repos call this `host-check`.
- Add a test that fails if the committed `.wasm`'s CID differs from
  `chain/spec.json`.
- Run an end-to-end test on a throwaway chain path. Sign with a local key,
  submit, then read the state back.
