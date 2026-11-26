# Transaction format

The JSON a node accepts at `POST /transactions`, and how to sign it. Use
this when you build a transaction yourself rather than with the `lattice tx`
commands, the [Lattice SDK](https://github.com/0xLattice/lattice-sdk/blob/main/docs/getting-started.md#build-sign-and-submit-a-transfer)
or the user's wallet.

## Body

```json
{"transaction":{
  "signatures":{"<multikey-pubkey-hex>":"<ed25519-signature-hex>"},
  "body":{
    "accountActions":[{"owner":"<signer>","delta":"-26"},{"owner":"<to>","delta":"25"}],
    "actions":[{"key":"user/<signer>/profile","newValue":"{\"name\":\"Ada\"}"}],
    "depositActions":[],
    "receiptActions":[],
    "withdrawalActions":[],
    "signers":["<signer-address>"],
    "nonce":"3",
    "chainPath":["Nexus","Alpha"]
  }}}
```

| Field | Meaning |
|---|---|
| `signatures` | One entry per signer: the signer's public key (Multikey hex, `ed01` then 64 hex digits) to its signature (128 hex digits). |
| `accountActions` | Balance changes, each `{owner, delta}`. A negative `delta` is a debit and needs the owner's signature. A positive one is a credit and needs none. A `delta` is never 0. |
| `actions` | Key–value writes, each `{key, oldValue?, newValue?}` ([below](#key-value-actions)). |
| `depositActions`, `receiptActions`, `withdrawalActions` | The three steps of an exchange between a chain and its parent ([below](#cross-chain-actions)). |
| `signers` | The addresses that sign. The addresses of the keys in `signatures` must equal this list exactly. |
| `nonce` | The first signer's next nonce on this chain ([Nonce](#nonce)). |
| `chainPath` | The chain the transaction is for. It is part of the signed data, so a transaction cannot be replayed on another chain. |

Every 64- and 128-bit integer (`delta`, `nonce`, amounts) is a **decimal
string**. A JSON number is refused.

### The fee

There is no fee field. The fee is what the transaction debits beyond what
it credits: in the example, 26 − 25 = 1. It goes to the recipient of the
block that includes the transaction.

### Key-value actions

An action applies only if the key's current value equals `oldValue`.

| To | Send |
|---|---|
| Create a key | `newValue` only. The key must not exist yet. |
| Change a key | `oldValue`, the value you read, and `newValue`. |
| Delete a key | `oldValue` only. |

### Cross-chain actions

```text
deposit:    {nonce, demander, amountDemanded, amountDeposited}           on the child; the demander signs
receipt:    {withdrawer, nonce, demander, amountDemanded, directory}     on the parent; the withdrawer signs
withdrawal: {withdrawer, nonce, demander, amountDemanded, amountWithdrawn}  on the child; the withdrawer signs
```

A withdrawal credits its recipient with an `accountActions` entry. How the
three fit together is in
[Cross-chain transfers](https://lattice.build/docs/protocol/cross-chain/).

## Signing

Sign with Ed25519 over the UTF-8 bytes of `"lattice-tx-v1:" + envelope`.
The envelope is these lines joined with `\n`, with no trailing newline:

```text
domain:13:lattice-tx-v1
chainPath.count:<n>
chainPath.component:<utf8 length>:<component>      (one line per component)
nonce:<decimal>
bodyCID:<utf8 length>:<CID of the DAG-CBOR body>
```

A signature over the bare body CID is invalid.

Do not hand-roll DAG-CBOR and CIDs unless you have to:

- In JavaScript or TypeScript, the SDK's `@adalin/lattice-core` provides
  `accountFromPrivateKey`, `signTransactionBody`, `transactionPayload` and
  `signedTransactionCID`
  ([API reference](https://github.com/0xLattice/lattice-sdk/blob/main/docs/api.md#transactions)).
- In Swift, use the [Lattice](https://github.com/0xLattice/Lattice) package
  itself.

## Nonce

Use the `nonce` field of `GET /api/state/account/<signer>?chainPath=…`. It
is already the next value to use. Fetch it with caching disabled.

Only the first signer, `signers[0]`, has a nonce checked and advanced.
Co-signers' nonces are neither checked nor advanced.

A node pools only a transaction that can execute on the current tip. One
whose nonce is ahead is refused with `nonceAhead`: wait until the signer's
earlier transaction is mined, then send it again.

## Submit

To your own node's operator port, with its cookie:

```bash
curl -s --user "$(cat <root>/chains/Nexus/.cookie)" -H 'Content-Type: application/json' \
  -X POST http://127.0.0.1:8080/transactions -d @tx.json
# {"transactionCID":"bafy…","mempoolCount":4,"mempoolBytes":2310}
```

Or, with no cookie, to a public node whose `/api/chain/info?chainPath=…`
reports `acceptsSubmit: true`. There the body is at most 1 MiB, and the
transaction is not written to the node's journal, so it is lost if the node
restarts before relaying it.

- `chainPath` must name a chain the receiving node hosts; otherwise the
  answer is 404.
- A refusal names its reason, such as `belowMinRelayFee`, `feeTooLow`,
  `nonceAhead` or `invalidState`. Each reason, and whether to retry, is in
  the [HTTP API](https://lattice.build/docs/build/http-api/reads-and-transactions/).
- **An accepted submit is not inclusion.** Confirm with
  `GET /api/transaction/<cid>`, whose `blockHeight` and `blockHash` appear
  once the transaction is in a block of the node's chain, or by reading the
  state back.
