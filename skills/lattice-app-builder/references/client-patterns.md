# Client patterns (JavaScript)

These snippets follow the public moniker, fanfare and stoa clients. `sdk` is
a vendored, pinned build of https://github.com/0xLattice/lattice-sdk.

## Tip → verified state

```js
export async function tip(sdk, nodeURL, chainPath, volumes) {
  const url = new URL("/api/block/latest", nodeURL);
  url.searchParams.set("chainPath", chainPath.join("/"));
  const { hash } = await (await fetch(url, { cache: "no-store" })).json();
  const block = sdk.decodeCanonicalBlock((await volumes.get(hash)).rootBytes());
  const state = sdk.decodeCanonicalLatticeState((await volumes.get(block.postStateCID)).rootBytes());
  return { hash, generalStateCID: state.generalStateCID };
}
const volumes = new sdk.VolumeClient(new sdk.HTTPVolumeTransport(`${node}/volumes`, { chainPath }));
```

## Trie get

In the radix trie, each child is filed under the first character of its
prefix.

```js
export async function get(load, rootCID, key) {
  let { node, volume } = await load(rootCID);
  let rest = key;
  while (rest.length > 0) {
    const cid = node.children.find((c) => c.key === String.fromCodePoint(rest.codePointAt(0)))?.value.rawCID;
    if (cid === undefined) return undefined;
    ({ node, volume } = await load(cid, volume));
    if (!rest.startsWith(node.prefix)) return undefined;
    rest = rest.slice(node.prefix.length);
  }
  return node.value;
}
```

## Compare-and-swap actions from intended writes

```js
export async function changes(state, writes) {          // writes: [[key, newValue | undefined]]
  const actions = await Promise.all(writes.map(async ([key, newValue]) => {
    const oldValue = await state.get(key);
    return { key, ...(oldValue !== undefined && { oldValue }), ...(newValue !== undefined && { newValue }) };
  }));
  return actions.filter((a) => "oldValue" in a || "newValue" in a);
}
```

## Sign and submit without a wallet (scripts, e2e tests)

```js
const acct = await api(`/api/state/account/${who.address}`, chain);   // cache: "no-store"
const nonce = BigInt(acct?.nonce ?? 0);
const body = buildBody({ owner: who.address, actions, fee, nonce, chainPath: chain });
const privateKey = Uint8Array.from(who.privateKeyHex.match(/../g), (h) => parseInt(h, 16)); // key files store hex
const { publicKey, signature } = sdk.signTransactionBody(body, privateKey);
await fetch(new URL("/transactions", node), {
  method: "POST",
  headers: { "Content-Type": "application/json" },   // operator port: also send the cookie (Basic auth)
  body: JSON.stringify(sdk.transactionPayload({ [publicKey]: signature }, body)),
});
```

Without a cookie, this only works on a public host whose
`/api/chain/info?chainPath=…` reports `acceptsSubmit: true`.

## Media as a one-block Volume, pinned on your own node

```js
const block = sdk.encodeDagCbor({ bytes, type });
const cid = sdk.cidV1DagCbor(block);
await fetch(new URL("/pins", rpc), {
  method: "POST",
  headers: { Authorization: `Basic ${btoa(cookie)}`, "Content-Type": "application/json" },
  body: JSON.stringify({ cid, entries: { [cid]: base64(block) } }),
});
```

## Find a node for any chain

```js
const nodes = new sdk.ChainNodes(settings.node, { fetch: (...a) => fetch(...a) });
const state = await nodes.read(chain, (node) => readStateAt(node.url, chain)); // throws ChainOfflineError if none
```

## Failover and budgets

- Content is addressed by CID, so any host may serve it. Try hosts in order
  and verify every Volume.
- Cache immutable trie nodes for the lifetime of the loader. Reuse a state
  snapshot for about 10 s. Batch sibling fetches.
- On 429 or 503 + `Retry-After`, back off and retry.
