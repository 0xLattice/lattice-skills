# Client patterns for Lattice applications (JavaScript)

Recipes for an application's client: reading a chain's key–value state,
writing to it, storing files, and finding a node. They use the
[Lattice SDK](https://github.com/0xLattice/lattice-sdk/blob/main/docs/getting-started.md)
(`npm install @adalin/lattice-sdk`). Each snippet is complete unless it says
it is a fragment and names what it assumes.

The examples use the public test chain `Nexus/testnet`. Replace the chain
path and node URLs with your application's, and keep both configurable.

## Connect once

`connect` gives one chain to read and write. Run with the CID of a recent
block of the chain as the argument; a page ships that CID with each release.

```js
import { connect } from "@adalin/lattice-sdk";

const chain = await connect({
  chain: "Nexus/testnet",
  // Nodes of different operators: a tip is read once two of them hold it.
  node: ["https://ams.testnet.lattice.build", "https://lattice-testnet-apps-iad.fly.dev"],
  // Reads must descend from this block.
  checkpoint: process.argv[2],
});
console.log(chain.address);
```

What `connect` checks, and what it cannot, is in the
[`connect` reference](https://github.com/0xLattice/lattice-sdk/blob/main/docs/connect.md).

## Read a key, list a prefix

The fragments from here on assume `chain` from [Connect once](#connect-once).

```js
const profile = await chain.get(`user/${chain.address}/profile`); // a string, or undefined
const posts = await chain.list("user/", { limit: 50 }); // [{ key, value }] in key order
```

Both read at the chain's tip and check the block, its state and every trie
node against their CIDs. Values are strings; parse JSON yourself, and treat
every value as untrusted input written by someone else.

A node has no HTTP route that returns one key's value. State is read as
content: each trie node is one Volume fetched from `/volumes`. `chain.get`
and `chain.list` do that walk for you.

## Write one key

```js
import { WriteError } from "@adalin/lattice-sdk";

try {
  const { transactionCID } = await chain.set(`user/${chain.address}/profile`, '{"name":"Ada"}');
  console.log(transactionCID); // resolved once the write is in a block
} catch (error) {
  if (!(error instanceof WriteError)) throw error;
  console.error(error.code); // rejected, refused, insufficientBalance, sponsorUnavailable, notConfirmed
}
```

`chain.set` reads the key's current value and states it in the write, so a
change made by someone else in between makes the write fail (`rejected`)
instead of being overwritten. Who signs and who pays depends on the `key`
and `sponsors` options of `connect`
([Getting started](https://github.com/0xLattice/lattice-sdk/blob/main/docs/getting-started.md#3-write)).

`chain.set` writes one string under one key. Build the transaction yourself,
as in the next three sections, when a write must:

- change several keys together;
- delete a key;
- pay someone in the same transaction;
- be signed by the user's wallet.

## Build compare-and-swap actions

An action is `{ key, oldValue?, newValue? }`. It applies only if the key's
current value equals `oldValue`. Leave `oldValue` out to create a key, and
`newValue` out to delete one.

```js
// writes: [[key, newValue | undefined]]; undefined deletes the key.
export async function actionsFor(chain, writes) {
  const actions = await Promise.all(
    writes.map(async ([key, newValue]) => {
      const oldValue = await chain.get(key);
      return {
        key,
        ...(oldValue !== undefined && { oldValue }),
        ...(newValue !== undefined && { newValue }),
      };
    }),
  );
  // Deleting a key that does not exist is no action at all.
  return actions.filter((action) => "oldValue" in action || "newValue" in action);
}
```

Read the old values just before you build the write. A stale one makes the
whole transaction fail.

## Write through the user's wallet

Use this when the page should hold no key. A wallet that injects
`window.lattice` picks the nonce, shows the write to the user, signs and
submits. A fragment: it assumes a browser with such a wallet, `chain` and
`actionsFor` from above, and `seller`, an address to pay.

```js
// The wallet may inject window.lattice after the page loads.
const wallet =
  window.lattice ??
  (await new Promise((resolve) =>
    window.addEventListener("lattice#initialized", () => resolve(window.lattice), { once: true }),
  ));

// Call from a click: the wallet asks the user once per site.
const address = await wallet.connect();

const { transactionCID } = await wallet.writeRecords({
  chainPath: ["Nexus", "testnet"],
  actions: await actionsFor(chain, [[`user/${address}/profile`, '{"name":"Ada"}']]),
  payments: [{ to: seller, amount: "250" }], // optional, in the same transaction
  fee: "1", // optional; a decimal string
});
```

- Amounts and fees are decimal strings.
- If the wallet never answers, treat the write as one that may still land,
  and do not submit it again.
- For an off-chain proof, `wallet.signMessage({ text })` returns
  `{ publicKey, signature }` over
  `"lattice-message:" + origin + "\n" + text`.

## Sign and submit with a local key

For scripts, tests and bots. Save it as `write.mjs` beside a file `bot.key`
holding 64 hex digits (`lattice key generate` writes the same hex as
`privateKey` in its JSON). The key must hold coin for the fee.

```js
import { readFileSync } from "node:fs";
import {
  HTTPTransactionSubmitter,
  NodeClient,
  SubmissionError,
  accountFromPrivateKey,
  connect,
  hexToBytes,
  signTransactionBody,
  transactionPayload,
} from "@adalin/lattice-sdk";

const node = "https://ams.testnet.lattice.build";
const chainPath = ["Nexus", "testnet"];
const privateKey = hexToBytes(readFileSync("./bot.key", "utf8").trim());
const account = accountFromPrivateKey(privateKey);

const chain = await connect({ chain: chainPath.join("/"), node, checkpoint: process.argv[2] });
const reads = new NodeClient(node, chainPath);
const { minRelayFee: fee = 1n } = await reads.chainInfo(); // this node's fee floor
const { nonce } = await reads.account(account.address); // already the next nonce

const key = `user/${account.address}/profile`;
const oldValue = await chain.get(key);

const body = {
  // There is no fee field: the fee is what the signer is debited and nobody is credited.
  accountActions: [{ owner: account.address, delta: -fee }],
  actions: [{ key, ...(oldValue !== undefined && { oldValue }), newValue: '{"name":"Ada"}' }],
  depositActions: [],
  receiptActions: [],
  withdrawalActions: [],
  signers: [account.address],
  nonce,
  chainPath,
};
const { publicKey, signature } = signTransactionBody(body, privateKey);

try {
  const submitter = new HTTPTransactionSubmitter(`${node}/transactions`);
  const { transactionCID } = await submitter.submit(
    transactionPayload({ [publicKey]: signature }, body),
  );
  console.log(transactionCID); // pooled, not yet in a block
} catch (error) {
  if (!(error instanceof SubmissionError)) throw error;
  console.error(error.status, error.reason ?? error.refusal);
}
```

- A public node takes the submit only when its
  `/api/chain/info?chainPath=…` reports `acceptsSubmit: true`.
- For your own node's operator port, pass
  `{ authorization: nodeCookieAuthorization(cookieFileContent) }` to both
  `NodeClient` and `HTTPTransactionSubmitter`.
- Confirm by polling `reads.transaction(transactionCID)` until `blockHeight`
  is set, or by reading the keys back.
- The body's fields and the signature are specified in
  [Transaction format](../../lattice-transactions/references/transaction-format.md).

## Store files

Put a file's CID in state, never a URL. Anyone can then fetch the bytes
from any node that holds them and check them against the CID.

With a sponsor that sells storage, `connect` stores and reads files. A
fragment: it assumes `chain` was made with a `sponsors` option, `fileInput`,
an `<input type="file">`, and `img`, an image element.

```js
const root = await chain.put(fileInput.files[0]); // paid for in proof of work; the root CID
await chain.set(`user/${chain.address}/avatar`, root);

const blob = await chain.getFile(root, { maxBytes: 2 * 1024 * 1024 });
if (["image/png", "image/jpeg", "image/webp"].includes(blob.type)) {
  img.src = URL.createObjectURL(blob);
}
```

- `getFile` checks every chunk against its CID and refuses a file over
  `maxBytes` before fetching it.
- Accept only an allowlist of raster image types. Never render SVG from a
  chain: it can run script.

To hold a file on a node you run instead, pin it there. The operator
command is `lattice pins add <cid>`; to supply the bytes yourself, post them
to the node's operator port. A fragment: it assumes `bytes` (a
`Uint8Array`), `type` (a media type), and `cookie`, the content of the
node's cookie file.

```js
import { cidV1DagCbor, encodeDagCbor, nodeCookieAuthorization } from "@adalin/lattice-sdk";

// One block { type, size, bytes }: the layout chain.getFile reads, for files up to 1 MiB.
const block = encodeDagCbor({ bytes, size: BigInt(bytes.length), type });
const cid = cidV1DagCbor(block);

let binary = "";
for (let at = 0; at < block.length; at += 0x8000) {
  binary += String.fromCharCode(...block.subarray(at, at + 0x8000));
}
await fetch("http://127.0.0.1:8080/pins", {
  method: "POST",
  headers: { Authorization: nodeCookieAuthorization(cookie), "Content-Type": "application/json" },
  body: JSON.stringify({ cid, entries: { [cid]: btoa(binary) } }),
});
```

A browser page can reach the operator port only when its origin is listed
in the node's `rpcAllowedOrigins`. A pin holds that one Volume, not the
Volumes it links to.

## Find a node for any chain

`ChainNodes` starts from nodes you choose and finds the hosts of any chain
below them, through what each parent's hosts declare.

```js
import { ChainNodes, ChainOfflineError, NodeClient } from "@adalin/lattice-sdk";

const chainPath = ["Nexus", "testnet"];
const nodes = new ChainNodes(["https://ams.testnet.lattice.build"]);

try {
  // Tries the next node in turn, then the others, until one answers.
  const info = await nodes.read(chainPath, (node) =>
    new NodeClient(node.url, chainPath).chainInfo(),
  );
  console.log(info.height, info.acceptsSubmit);
} catch (error) {
  if (!(error instanceof ChainOfflineError)) throw error;
  console.error("no node serves", chainPath.join("/"));
}
```

A found host is declared by its operator and not otherwise verified. Check
everything read from it by CID, and read a fee floor (`minRelayFee`) from
the node you submit to, not from one you merely discovered.

## Read state at a block you choose

`chain.get` and `chain.list` read at the tip that `connect` chooses.
Hand-roll the read only when you need something else: the state at an older
block, your own rule for choosing the tip, or a read without `connect`.

General state is a radix trie. Each node has a `prefix`, an optional
`value`, and `children`, each filed under the first character of its own
prefix. Run with a block's CID and a key:

```js
import {
  HTTPVolumeTransport,
  VolumeClient,
  decodeCanonicalBlock,
  decodeCanonicalLatticeState,
  decodeDagCbor,
} from "@adalin/lattice-sdk";

const node = "https://ams.testnet.lattice.build";
const chainPath = ["Nexus", "testnet"];
const volumes = new VolumeClient(new HTTPVolumeTransport(`${node}/volumes`, { chainPath }));

// The general-state root of the block `blockCID` names, each step checked by CID.
async function generalStateRoot(blockCID) {
  const block = decodeCanonicalBlock((await volumes.get(blockCID)).rootBytes());
  const state = decodeCanonicalLatticeState((await volumes.get(block.postStateCID)).rootBytes());
  return state.generalStateCID;
}

// One trie node. A node inside a Volume already fetched is read from it.
async function load(cid, within) {
  const volume = within?.has(cid) ? within : await volumes.get(cid);
  return { node: decodeDagCbor(volume.bytes(cid)), volume };
}

export async function getAt(blockCID, key) {
  let { node: at, volume } = await load(await generalStateRoot(blockCID));
  let rest = key;
  while (rest.length > 0) {
    // The child filed under the longest key that starts what is left.
    let next;
    for (const child of at.children) {
      if (rest.startsWith(child.key) && child.key.length > (next?.key.length ?? 0)) next = child;
    }
    if (next === undefined) return undefined;
    ({ node: at, volume } = await load(next.value.rawCID, volume));
    const prefix = at.prefix ?? "";
    if (!rest.startsWith(prefix)) return undefined;
    rest = rest.slice(prefix.length);
  }
  return at.value;
}

console.log(await getAt(process.argv[2], process.argv[3]));
```

This checks content against CIDs and nothing else. Which block to trust is
then your decision: a node's answer to `/api/block/latest` is only its word.

## Failover and request budgets

- Content is addressed by CID, so any host may serve it. Try hosts in order
  and verify every Volume. `connect` and `ChainNodes` do this for you.
- A Volume never changes. Keep verified Volumes in a cache
  ([`IndexedDBVolumeCache`](https://github.com/0xLattice/lattice-sdk/blob/main/docs/api.md#caches))
  and reuse a state read for a few seconds rather than repeating it.
- A public node limits requests for all its clients together. On `429`, or
  `503` with `Retry-After`, back off and retry. A `503` on `/volumes` means
  the node is reading the content from its peers.
- Fetch a tip or a nonce with `cache: "no-store"`: a node's live answers are
  cached for a few seconds. The SDK does this where it matters.
