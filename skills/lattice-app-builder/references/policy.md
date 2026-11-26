# Writing a validity policy

A validity policy is a WebAssembly module that a chain names in its spec.
Every node hosting the chain runs it on each transaction, and it answers
accept or reject. This page gives the module's interface, a Rust starting
point, and how to build and test one.

A policy is a validator, not a smart contract. It cannot change state, read
keys the transaction does not name, hold funds or call another policy, and
it does not see the block's reward.

## What a policy checks

A typical application policy, in scope `transaction`:

- checks every action's key against the shapes the application allows;
- checks that the owner named in a key is one of the transaction's signers;
- checks token transfer and purchase rules;
- bounds a `time` field to the block's timestamp, or better, keys the rule
  on the block's height.

Consensus alone lets any signer write any key. Whatever ownership an
application has comes from its policy.

## The module interface

| Requirement | Detail |
|---|---|
| Format | A WebAssembly module, named in the spec by its CID. |
| Scope | `transaction`: called once with the whole transaction body. `action`: called once per key–value action. Use `transaction` for an application. |
| Exports | `memory`; `lattice_alloc(len: i32) -> i32`; and the entrypoint `(ptr: i32, len: i32) -> i32`. |
| Entrypoint name | Whatever the spec's `entrypoint` says. By convention `lattice_validate_transaction` or `lattice_validate_action`. |
| Verdict | Return `1` to accept. Any other value, or a trap, rejects. |
| Arithmetic | Integers only. A module that uses float or SIMD instructions is refused when the chain is created. |
| Limits | No fuel, size or memory limit beyond the maxima the module itself declares. |

The host allocates with `lattice_alloc`, writes the policy context into the
module's memory, and calls the entrypoint with its pointer and length.

The context is a binary envelope that begins with the ASCII magic `LWPCTX`.
It holds, in order: an encoding version, the ABI version, the scope, the
height and timestamp of the block being validated, the chain's spec as
DAG-CBOR, the chain path, and then the transaction body, the action and the
action's index, each present or absent. The byte layout is in the
[protocol specification](https://lattice.build/docs/protocol/spec/), under
"WASM Policies".

Rules for the policy's own logic:

- Reject anything you cannot parse.
- Depend on nothing a miner can vary freely. The block's timestamp is the
  miner's choice within a range, so key expiry rules on height, or bound
  timestamps to a window.
- Keep it fast. There is no fuel limit, and every node hosting the chain
  runs the policy on every transaction.

## Name the policy in the spec

```json
"wasmPolicies": [
  {
    "moduleCID": "<CID printed by your build or check step>",
    "abiVersion": 1,
    "scope": "transaction",
    "entrypoint": "lattice_validate_transaction"
  }
]
```

The other spec fields are in the
[ChainSpec reference](../../lattice-child-chains/references/chainspec.md).
**Rebuilding the module changes its CID, which changes the spec, which
makes a different chain.** A spec cannot be changed after the chain starts.

## A Rust starting point

The example below is a fragment. It assumes the `lattice-policy-rules`
crate, which lives in the `rules/` directory of the
[moniker](https://github.com/adalinxx/moniker) example application. That
crate is an application's code, not part of Lattice: it supplies the three
exports through a macro, and a parser for the context and the transaction
body. Read `rules/src/lib.rs` at the tag you pin, or write the exports and
the parsing yourself from the specification.

`Cargo.toml`. Depend on the crate by git tag, never by path: a sibling
`path =` dependency does not build in a fresh clone.

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

`src/lib.rs`, a policy that allows only keys shaped `user/<owner>/...`
written by `<owner>`:

```rust
#![cfg_attr(not(test), no_std)]
use lattice_policy_rules::{transaction_body, Body};
lattice_policy_rules::lattice_policy_abi!(crate::validate);

pub fn validate(context: &[u8]) -> bool {
    transaction_body(context).and_then(|b| {
        let body = Body::parse(b)?;
        for action in body.actions()? {
            let action = action?;                       // Action { key, old_value, new_value } as &[u8]
            let rest = action.key.strip_prefix(b"user/")?;
            let owner = rest.split(|&b| b == b'/').next()?;
            body.is_signer(owner)?;                     // Option<()>: None rejects
        }
        Some(())
    }).is_some()
}
```

The crate also has helpers for tokens (`check_token_write`, `credited`).

## Build

```bash
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/<app>_policy.wasm chain/
```

Commit the built `.wasm` beside the spec, so that hosting the chain does
not need a Rust toolchain. Pass it to `lattice child create` with
`--policy-module`: the file is matched to the spec by its CID and checked
against this interface, and creation is refused when they do not match.

## Test

1. **Unit-test `validate` natively** on contexts you build by hand: the
   accept cases, every reject case, and malformed input.
2. **Replay the same contexts through Lattice's own policy evaluator**, for
   example from a small Swift package that depends on `Lattice` and prints
   the module's CID.
3. **Fail the build when the committed `.wasm` and the spec disagree**: a
   test that compares the module's CID with `moduleCID` in
   `chain/spec.json`.
4. **Run end to end on a throwaway chain path.** Sign with a local key,
   submit, and read the state back
   ([Client patterns](client-patterns.md#sign-and-submit-with-a-local-key)).
