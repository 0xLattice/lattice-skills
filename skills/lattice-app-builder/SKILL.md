---
name: lattice-app-builder
description: Design and build an application on Lattice — choosing a chain layout (own child chain, per-tenant sub-chains, or a shared chain), modelling data as key–value state with compare-and-swap actions, enforcing rules with a WASM validity policy, reading verified state by CID, writing through the user's wallet, a sponsor or a signing script, storing files by CID, and hosting the app's chain. Use when a user wants to build a dApp, social app, marketplace, token, game or any data-backed product on Lattice, or asks how existing Lattice apps are structured.
---

# Building apps on Lattice

Lattice has **no smart contracts**. An app is usually built from four parts:

1. **A chain**: usually its own child chain.
2. **Key–value state** on that chain. The chain is the database.
3. **A WASM validity policy** that decides which writes are allowed.
4. **A client**, often a static web page, that reads verified state and
   writes through the user's wallet or a key of its own.

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
- A node's `minRelayFee` is 1 by default. Set it to 0 if writes should be
  free, or, for users who hold no coin, run a lattice-sponsor
  (https://github.com/0xLattice/lattice-sponsor): it pays fees in return for
  proof of work done in the user's browser.
- Get at least one miner to host the chain and set a recipient for it in
  `miner.json`.

Collaborators and other hosts **join** the chain with
`lattice child host <path>`. They must not `child create` it again, which
would fork a rival chain. See the `lattice-child-chains` and
`lattice-node-operator` skills.

## 5. Read state (verified)

Use the JS SDK (https://github.com/0xLattice/lattice-sdk,
`npm install @adalin/lattice-sdk`, or its single-file `dist/browser.js`
from a CDN). `connect` gives one chain to read and write:

```js
const chain = await connect({ chain: "Nexus/<app>", node: [urlA, urlB], checkpoint });
await chain.get(key);                    // string | undefined
await chain.list(prefix, { limit });     // [{ key, value }]
```

What it does, and what to do by hand when you cannot use it:

1. `GET /api/block/latest?chainPath=Nexus/<app>` with `cache: "no-store"`
   → `hash`. With several nodes, a tip is read only once `minAgree` of them
   (two by default) hold it; with a `checkpoint` or `genesis` CID shipped in
   the app, the tip must link back to it.
2. `GET /volumes/<hash>?chainPath=…` → decode the block → `postStateCID`.
3. Fetch the post-state → `generalStateCID`, then walk the radix trie
   Volume by Volume, verifying each CID.

The node has no generic "get key" HTTP route, so this trie walk *is* the
read path. Reads are checked by CID; no block is executed, so give nodes of
**different operators**. Check `npm view @adalin/lattice-sdk version` before
relying on an API: `connect` and the browser bundle need 0.2.0 or later, so
build the repository at a pinned commit when the registry is behind. The
SDK also has `ChainNodes` (it finds a node for any chain by walking
`/api/chain/endpoints`). Complete recipes, including the hand-rolled walk,
are in `references/client-patterns.md`.

Other useful reads:

- `/api/chain/info` for `minRelayFee`, `acceptsSubmit` and `height`.
- `/api/state/account/:addr` for balance and nonce.
- `/api/chain/children` to discover tenant chains.
- `/api/deposits` for open cross-chain offers.

Integers come back as decimal strings, so parse them with `BigInt`.

Each trie node costs one `/volumes` read, and a public host limits requests
for all its clients together. Cache immutable nodes, reuse a state snapshot for a few seconds,
and expect **503 + Retry-After** while a host fetches from peers.

## 6. Write

Pick who holds the key:

- **The user's wallet (the app holds no key).** Call the wallet the user
  has installed through `window.lattice`:

  ```js
  await window.lattice.connect();                         // from a user click
  const { transactionCID } = await window.lattice.writeRecords({
    chainPath: ["Nexus", "<app>"],
    actions:  [{ key, oldValue, newValue }],
    payments: [{ to: seller, amount: "250" }],            // optional, same transaction
    fee: String(minRelayFee),                             // optional
  });
  ```

  The wallet picks the nonce, signs and submits. `window.lattice` may be
  injected after page load, so listen for `lattice#initialized`. For
  off-chain proofs, `signMessage({text})` signs
  `"lattice-message:" + origin + "\n" + text`.
- **A key in the page, fee paid in work.** `connect({ …, sponsors: [url] })`
  then `chain.set(key, value)`. With no `key` option the SDK makes a key and
  keeps it in the page's `localStorage`: any script on the origin can read
  it, so keep the origin free of third-party script.
- **Scripts, tests and bots**: sign with the SDK
  (`signTransactionBody`, `transactionPayload`) and submit with
  `HTTPTransactionSubmitter`. See the `lattice-transactions` skill.
- **Confirm by reading back.** A consensus refusal (a stale `oldValue`, a
  balance that cannot pay, a policy's reject) is reported as the same
  `invalidState`. To explain one to the user, re-read the keys the write
  named ("sold to someone else", "price changed", "not enough coin"). The
  node's other refusals are named and say whether to retry (`nonceAhead`,
  `unavailable`, `insufficientFunds`, `full`) or to raise the fee
  (`feeTooLow`, `belowMinRelayFee`). If the wallet never answers, treat the
  write as "may still land" and don't resubmit.

## 7. Media and files

- Put **only the file's CID** in state, never a URL.
- **Store** with the SDK: `await chain.put(blob)` returns the root CID. It
  goes through a sponsor that sells storage and is paid for in proof of
  work. The SDK lays a file out as one DAG-CBOR block `{type, size, bytes}`
  up to 1 MiB, else a root naming 1 MiB chunks.
- **Or pin on a node you run**: `lattice pins add <cid>`, or `POST /pins`
  on the operator port with the bytes. From a page that needs the cookie,
  plus the page origin in `rpcAllowedOrigins`.
- **Read** with `await chain.getFile(cid, { maxBytes })`, which checks every
  chunk against its CID. Accept only an allowlist of raster image types
  under a size cap, and never render SVG (it can run script).

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
- [ ] Pin SDK and policy-crate dependencies by version, tag or commit, never
      by branch or sibling path.
- [ ] Ship a recent `checkpoint` with every release: reads fail `tooFar`
      once the chain is 2,000 blocks past it.
- [ ] In-page Ed25519 verification needs a recent browser (Chrome 137+,
      Firefox 130+, Safari 17+).
- [ ] Cross-chain trades: a deposit never expires, so make that clear in the
      UI.
