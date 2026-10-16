# Mining worker contract

A worker is a stateless program. The coordinator runs it once per nonce
range:

```text
<worker> --work-id <opaque> --block-hex <hex nonce-0 Block> --target <hex UInt256> \
         --start-nonce <n> --count <n> [--prefix-hex <hex PoW preimage prefix>]
```

- Prefer `--prefix-hex`: with it you only need SHA-256, not a block parser.
  `--block-hex` is always sent too. Accept both arguments, or ignore the one
  you don't use.
- **Search**: a nonce wins iff
  `SHA256(prefix || nonce as 8 bytes big-endian) <= target`, comparing both
  as 256-bit big-endian numbers. Equality counts as a win.
- The preimage length is fixed, so you can hash the prefix once into a
  midstate. Never derive the prefix yourself; use the bytes you are given.
- **Output**: exactly one JSON object on stdout, then exit 0 in both cases:

  ```json
  {"workId":"…","status":"found","nonce":12345,"hash":"<64 hex>","rangeStart":0,"rangeCount":1000000}
  {"workId":"…","status":"exhausted","nonce":null,"hash":null,"rangeStart":0,"rangeCount":1000000}
  ```

  Anything else on stdout, or a non-zero exit, counts as a worker failure and
  the batch is retried. Use stderr for logs.
- **Merged mining**: report the **lowest** winning nonce in the range. The
  coordinator restarts the search just past that nonce, so reporting a higher
  one silently skips harder-target (parent) hits below it. Don't assume one
  invocation per range, or one target per `--work-id`.
- The coordinator re-hashes your nonce itself and doesn't trust your
  `hash`.
- A worker never builds blocks, gossips, knows the chain topology, or sees
  private keys.
- For remote hardware, write a local shim that follows this contract and
  forwards the work. Make sure its teardown kills any child processes.

Reference vector (from lattice-node's `MiningWorkerContractTests`): with the
Nexus genesis preimage prefix, `SHA256(prefix || be64(12345))` =
`0179e03e9b0545582bb0aba1729d83b611b16e49abcefe44d6c1d35e85c4b3a3`. Pass
`--target` set to that hash, `--start-nonce 12345 --count 1`, and your worker
must report `found` at nonce 12345 with that hash. The full prefix hex is in
that test file.
