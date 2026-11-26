# Mining worker contract: quick reference

The authoritative contract is the node's
[Mining worker contract](https://lattice.build/docs/run/mining-workers/)
page. This file keeps the rules a worker author must not get wrong. When
the two disagree, the page is right.

The coordinator runs a worker once per assignment:

```text
<worker> --work-id <opaque> --block-hex <hex nonce-0 Block> --target <hex UInt256> \
         --start-nonce <n> --count <n> --prefix-hex <hex PoW preimage prefix>
```

## Rules

1. **Search.** A nonce wins when
   `SHA256(prefix || nonce as 8 big-endian bytes) <= target`, both compared
   as 256-bit big-endian integers. Equality wins.
2. **Use `--prefix-hex` as given.** Never derive the prefix yourself. Its
   length is fixed, so hash it once into a midstate and append only the
   nonce per attempt. Accept `--block-hex` too, or ignore it.
3. **Print exactly one JSON object on stdout, then exit 0**, whether or not
   a nonce was found:

   ```json
   {"workId":"…","status":"found","nonce":12345,"hash":"<64 hex>","rangeStart":0,"rangeCount":1000000}
   {"workId":"…","status":"exhausted","nonce":null,"hash":null,"rangeStart":0,"rangeCount":1000000}
   ```

   Anything else on stdout, or another exit status, is a worker failure and
   the batch is retried. Log to stderr.
4. **Report the lowest winning nonce in the range.** One search serves
   several chains' targets. The coordinator resumes just past the nonce you
   report, with a harder target, so a higher winner silently skips the
   nonces below it.
5. **Do not assume one invocation per range, or one target per
   `--work-id`.**
6. **A worker only searches.** It never builds blocks, talks to peers,
   knows the chain topology or sees a private key. The coordinator hashes
   your nonce again itself and does not trust your `hash`.

For remote hardware, write a local program that follows this contract,
forwards the work, and kills its child processes when it is stopped.

## Check your worker before mining

With the Nexus genesis preimage prefix (the page linked above gives how it
begins; the whole hex is in lattice-node's `MiningWorkerContractTests`):

```text
SHA256(prefix || be64(12345)) = 0179e03e9b0545582bb0aba1729d83b611b16e49abcefe44d6c1d35e85c4b3a3
```

Run the worker with that prefix, `--target` set to that hash, and
`--start-nonce 12345 --count 1`. It must report `found` at nonce 12345 with
exactly that hash.
