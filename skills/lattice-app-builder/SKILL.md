---
name: lattice-app-builder
description: Design and build an application on Lattice — choosing a chain layout (own child chain, per-tenant sub-chains, or a shared chain), modelling data as key–value state with compare-and-swap actions, enforcing rules with a WASM validity policy, reading verified state by CID, writing through the Lattice wallet or a signing script, storing media by CID with pins, and hosting the app's chain. Use when a user wants to build a dApp, social app, marketplace, token, game or any data-backed product on Lattice, or asks how existing Lattice apps are structured.
---

# Building apps on Lattice

Lattice has **no smart contracts**. An app is usually built from four parts:

1. **A chain**: usually its own child chain.
2. **Key–value state** on that chain. The chain is the database.
3. **A WASM validity policy** that decides which writes are allowed.
4. **A client**, often a static web page, that reads verified state and
   writes through the user's wallet.

All logic beyond accept/reject (ranking, feeds, notifications, matching)
runs in the client or in off-chain services.

Public examples to study: [moniker](https://github.com/adalinxx/moniker)
(shared profiles and sellable handles), [fanfare](https://github.com/adalinxx/fanfare)
(tickets as resellable tokens) and [stoa](https://github.com/adalinxx/stoa)
(forum, one chain per community). Read their `chain/`, `policy/` and
`client/` folders.

## 1. Choose a chain layout

| Layout | When | Example |
|---|---|---|
| **Own home chain** `Nexus/<app>` | Most apps. You choose block time, fees, limits and rules. | `Nexus/stoa`, `Nexus/fanfare` |
| **Sub-chain per tenant** `Nexus/<app>/<tenant>` | Communities, artists or orgs that each want their own coin, throughput or rules, reusing the same spec and policy | one chain per community; one coin chain per artist with a premine to the artist |
| **Shared chain** | Data many apps should share. Namespace each app's fields. | one profile chain; apps write `user/<a>/app.<name>.<field>` |

Things to know:

- A path isn't owned. Anyone can mine a rival genesis at it.
- The spec and policy are immutable, so test thoroughly before launch.
- During development, run at a throwaway path such as
  `Nexus/testnet/<app>-<random>`. Make the node URL and chain path
  configurable in the client.

## 2. Model the data as key–value state

General state maps **string → string** (usually JSON). A write is an
action:

```json
{ "key": "user/<addr>/post/<id>", "oldValue": "<what you read>", "newValue": "<new JSON>" }
```

It applies **only if** the current value equals `oldValue` (omit it to
create a key; omit `newValue` to delete). Every write is therefore a
compare-and-swap, so a concurrent change makes your transaction fail
instead of being silently overwritten.

Conventions that work (all enforced by the policy, since consensus lets any
signer write any key):

- **Ownership by key shape**: `user/<owner>/…` may be written only if
  `<owner>` signed.
- **Shared indexes**: `index/<topic…>/<owner>` holds an owner's entry under
  a shared topic (followers, votes, replies), listable by prefix. Treat
  index entries as *claims* and verify them against the owner's own record.
- **Tokens**: a value of `<holder>` or `<holder>@<price>`. The holder can do
  anything with it. Anyone else can take a priced token by crediting the
  holder at least the price **in the same transaction**. Because the buyer
  names the value it read as `oldValue`, a price change makes the purchase
  fail instead of overcharging.
- **Random ids**: give each draft one id and keep it across retries, so a
  retry can't create a duplicate.
- **Time**: use the timestamp of the block that created the key, or store a
  `time` field that the policy bounds to the block timestamp within a window.
- **Private data**: encrypt values client-side and use opaque keys. The
  chain itself is public.
- **Size**: every insert costs `len(key)+len(value)` of the per-block
  `maxStateGrowth`. Keep values small and put files in content storage (§5).

## 3. Write the validity policy

A transaction-scope policy sees the whole transaction body and returns
accept or reject. A typical policy:

- checks every action's key against the allowed shapes;
- checks that the owner segment of the key is a signer;
- checks token transfer and purchase rules;
- bounds the `time` fields.

See `references/policy.md` for the ABI, a Rust skeleton and the build and
test workflow. **Rebuilding the policy changes its CID, and therefore the
chain.**

## 4. Deploy and host the chain

```bash
lattice child create Nexus/<app> --root R --spec chain/spec.json \
  --policy-module chain/<app>_policy.wasm [--premine-to <address>]
```

Then make it reachable and mined:

- Set `publicRead` and `publicReadURL` (HTTPS) so browsers can read it and
  parent hosts list it at `/api/chain/endpoints`.
- Optionally set `publicSubmit: true` so users without their own node can
  write.
- Set `minRelayFee: 0` if writes should be free.
- Get at least one miner to host the chain and set a recipient for it.

Collaborators and other hosts **join** the chain by adding it to
`hostedChains`. They must not `child create` it again, which would fork a
rival chain. See the `lattice-child-chains` and `lattice-node-operator`
skills.

## 5. Read state (verified)

Trust the node for **one** thing only, the tip hash, and verify everything
else by CID:

1. `GET /api/block/latest?chainPath=Nexus/<app>` with `cache: "no-store"`
   → `hash`.
2. `GET /volumes/<hash>?chainPath=…` → decode the block → `postStateCID`.
3. Fetch the post-state → `generalStateCID`, then walk the radix trie
   Volume by Volume, verifying each CID. `get(key)` follows the key's
   characters; `list(prefix)` enumerates a subtree.

The node has no generic "get key" HTTP route, so this trie walk *is* the
read path. The JS SDK (https://github.com/0xLattice/lattice-sdk) provides
`VolumeClient`, `HTTPVolumeTransport`, the block and state decoders, and
`ChainNodes` (it finds a node for any chain by walking
`/api/chain/endpoints`). It isn't reliably published to a registry. The
public apps vendor a pinned build (`app/vendor/lattice-sdk.js`). A
reader sketch is in `references/client-patterns.md`.

Other useful reads:

- `/api/chain/info` for `minRelayFee`, `acceptsSubmit` and `height`.
- `/api/state/account/:addr` for balance and nonce.
- `/api/chain/children` to discover tenant chains.
- `/api/deposits` for open cross-chain offers.

Integers come back as decimal strings, so parse them with `BigInt`.

Each trie node costs one `/volumes` read, and public hosts limit those per
client. Cache immutable nodes, reuse a state snapshot for a few seconds,
and expect **503 + Retry-After** while a host fetches from peers.

## 6. Write

- **Default: the app holds no key.** Call the Lattice wallet extension:

  ```js
  await window.lattice.connect();                         // from a user click
  const { transactionCID } = await window.lattice.writeRecords({
    chainPath: ["Nexus", "<app>"],
    actions:  [{ key, oldValue, newValue }],
    payments: [{ to: seller, amount: "250" }],            // optional, same transaction
    fee: String(minRelayFee),
  });
  ```

  The wallet picks the nonce, signs and submits. `window.lattice` may be
  injected after page load, so listen for `lattice#initialized`. For
  off-chain proofs, `signMessage({text})` signs
  `"lattice-message:" + origin + "\n" + text`.
- **Scripts, tests and bots**: sign with the SDK
  (`signTransactionBody`, `transactionPayload`) and POST to `/transactions`.
  See the `lattice-transactions` skill.
- **Confirm by reading back.** Every refusal is reported as the same
  `invalidState`. To explain one to the user, re-read the keys the write
  named ("sold to someone else", "price changed", "not enough coin"). If the
  wallet never answers, treat the write as "may still land" and don't
  resubmit.

## 7. Media and files

- Store one file as one DAG-CBOR block `{type, bytes}`. Its CID is the Volume
  root. Put **only the CID** in state, never a URL.
- The author pins the file on their own node with `POST /pins` on the
  operator port. That needs the cookie, plus the page origin in
  `rpcAllowedOrigins`.
- Readers fetch it with `GET /volumes/<cid>` from any public node. Re-verify
  the CID, accept only an allowlist of raster image types under a size cap,
  and never render SVG (it can run script).

## Pitfalls checklist

- [ ] Fetch the tip with no cache before building `oldValue`s. A stale tip
      makes the write fail.
- [ ] Read `minRelayFee` per node and chain. Don't trust a fee reported by
      an endpoint you merely discovered.
- [ ] A 404 from `/api/state/account` never means "zero balance". It means
      the node has no tip, or lacks that chain's state. An unseen address on
      a served chain returns `"0"`.
- [ ] Verify discovered endpoint URLs (`/api/chain/info?chainPath=`) and all
      content by CID.
- [ ] Names are first come, first served. For "verified" identities,
      cross-link the chain record with something the user controls, such as
      a `.well-known/lattice` file on their site.
- [ ] Commit the built `.wasm`, and test that its CID matches the spec.
- [ ] Pin SDK and policy-crate dependencies by tag or commit, never by branch
      or sibling path.
- [ ] In-page Ed25519 verification needs a recent browser (Chrome 137+,
      Firefox 130+, Safari 17+).
- [ ] Cross-chain trades: a deposit never expires, so make that clear in the
      UI.
