# Building and signing a transaction yourself

Use this when you're not using `lattice tx` or a wallet extension.

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

- Every 64/128-bit integer is a **decimal string**.
- `delta`: negative is a debit and requires the owner's signature; positive
  is a credit and needs no signature. A delta can never be 0.
- `actions` are KV compare-and-swap operations: `{key, oldValue?, newValue?}`.
  Omit `oldValue` to create a key; omit `newValue` to delete one.
- Action shapes:

  ```text
  deposit:    {nonce, demander, amountDemanded, amountDeposited}      (on a child; the demander signs)
  receipt:    {withdrawer, nonce, demander, amountDemanded, directory}  (on a parent; the withdrawer signs)
  withdrawal: {withdrawer, nonce, demander, amountDemanded, amountWithdrawn}  (on a child; the withdrawer signs; credit the recipient with an accountAction)
  ```

- There is no fee field. In the example above the fee is 26 − 25 = 1.
- `chainPath` must name a chain that the receiving node hosts. It is part of
  the signed data, so a transaction can't be replayed on another chain.

## Signing

Sign with Ed25519 over the UTF-8 bytes of `"lattice-tx-v1:" + envelope`. The
envelope is these lines joined with `\n`, with no trailing newline:

```text
domain:13:lattice-tx-v1
chainPath.count:<n>
chainPath.component:<utf8 length>:<component>      (one line per component)
nonce:<decimal>
bodyCID:<utf8 length>:<CID of the DAG-CBOR body>
```

A signature over the bare body CID is **invalid**. The set of signature keys
(converted to addresses) must equal `signers` exactly.

Don't hand-roll DAG-CBOR and CIDs unless you have to. The JS SDK
(https://github.com/adalinxx/lattice-sdk; vendor a pinned build, as the
public apps do) provides `accountFromPrivateKey`, `signTransactionBody` and
`transactionPayload`. From Swift, use the Lattice package itself.

## Nonce

`nonce` = the `nonce` field of `GET /api/state/account/<signer>?chainPath=…`
(already the *next* value). If you have pending transactions, use
`max(that, highest pending + 1)`. Fetch it with caching disabled.

## Submit

```bash
curl -s --user "$(cat R/chains/Nexus/.cookie)" -H 'Content-Type: application/json' \
  -X POST http://127.0.0.1:8080/transactions -d @tx.json
# {"transactionCID":"bafy…","mempoolCount":4,"mempoolBytes":2310}
```

Or POST without a cookie to a public host whose
`/api/chain/info?chainPath=…` reports `acceptsSubmit: true`. The size limit
is 1 MiB. Public submits are not journaled, so they are lost if the node
restarts before the transaction is relayed.

A 200 doesn't mean the transaction will be included. Confirm with
`/api/transaction/:cid` or by reading the state back.
