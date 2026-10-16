---
name: lattice-transactions
description: Lattice wallet keys, balances and transactions through lattice-node — `lattice key generate`, `lattice tx send/deposit/receipt/withdraw`, how fees and nonces really work, checking inclusion, cross-chain parent↔child exchange, raw POST /transactions, and the node HTTP API. Use when a user wants to create a wallet, send coins, check a balance or transaction, move value between a parent and child chain, or call the lattice-node HTTP API.
---

# Keys, transactions and the HTTP API

Written against lattice-node `main` and Lattice 45.2.0. Every 64/128-bit
number in the JSON API (balances, nonces, heights, amounts) is a **decimal
string**. A JSON number is rejected with 400.

## Keys

```bash
lattice key generate --out ./wallet.json    # prints address: and publicKey:; file is 0600
```

The file holds `{address, privateKey, publicKey}`. Keys are Ed25519, the
public key is Multikey hex, and the address is the CID of the public key
(`bafyrei…`). The command refuses to overwrite an existing file. **This file
is spendable money**: keep it at mode 0600, back it up, and never paste
`privateKey` into a chat, log or repo. `tx` commands refuse a key file that
is group- or world-readable.

The node's `identity/Nexus.key` is a P2P identity, not a wallet.

## Balances and nonces

```bash
C="$(cat <root>/chains/Nexus/.cookie)"; B=http://127.0.0.1:8080
curl -s --user "$C" "$B/api/state/account/<address>?chainPath=Nexus/Alpha" | jq
# {"owner":"…","balance":"1500","nonce":"3"}   # nonce is already the NEXT one to use
```

A 404 here means the node has no tip, or doesn't hold the state for that
chain. It does not mean the account is empty: an unseen address on a served
chain returns balance `"0"`.

## Send

```bash
lattice tx send --root R --chain Nexus/Alpha --key ./wallet.json \
  --to <address> --amount 25 [--fee 1] [--nonce N]
# submitted <txcid> / chain … signer … nonce … / mempool N transactions
```

Common options for every `tx` subcommand:

- `--chain` (required): an absolute path, either `Nexus` or a chain in
  `hostedChains`.
- `--key` (required).
- `--fee` (default 0).
- `--nonce` (default: the larger of the committed nonce and the signer's
  highest pending nonce + 1).

The CLI submits to the local node's loopback RPC, so **the chain must be
hosted by this node**.

## How fees actually work

- **Transactions have no fee field.** The fee is the transaction's *surplus*:
  `debits + withdrawals − credits − deposits`. It goes to the block's reward
  recipient. A transaction that creates value (negative surplus) is never
  accepted.
- `--fee N` adds an extra debit of N from the signer that nothing credits.
- Nodes enforce their own `minRelayFee` floor (default 0). Read it with
  `GET /api/chain/info?chainPath=…` → `minRelayFee`. Below the floor, the
  submit fails with 400 `belowMinRelayFee`. Different nodes can set
  different floors.
- **Replace-by-fee**: resubmitting with the same signers and nonce replaces
  the pending transaction only if the fee is strictly higher. Otherwise it
  fails with 400 `feeTooLow`.

## Nonces

Each signer has one nonce, and nonces must be contiguous. If the nonce is
too low, the transaction is invalid. If it is too high, the node **accepts it
with a 200 but parks it as `future`** until the gap fills. A transaction with
several signers advances every signer's nonce.

## Confirming inclusion

**A 200 from submit is not inclusion.** The transaction may be parked
(`future` or `unavailable`), and it can still be replaced, evicted or
reorged out. Check:

```bash
curl -s --user "$C" "$B/api/transaction/<txcid>?chainPath=Nexus/Alpha" | jq '{blockHeight, blockHash}'
```

`blockHeight` and `blockHash` appear only once the transaction is in the
canonical executed chain. `GET /api/mempool?chainPath=…` lists pending CIDs
(up to 200). There is no finality, so for high-value transfers wait several
blocks and re-check.

## Cross-chain exchange (parent ↔ direct child)

This is a **two-party exchange**, not a bridge. The **demander** sells
child-chain coin for parent-chain coin. The **withdrawer** pays on the
parent and claims the coin on the child. The two parties agree the rate as
`amountDemanded` (parent units) against the locked amount (child units).

```bash
# 1. CHILD: demander locks 50 child-coin and asks for 100 parent-coin. Picks a fresh swap nonce.
lattice tx deposit  --root R --chain Nexus/Alpha --key demander.json \
  --swap-nonce 7 --demand 100 --lock 50

# 2. PARENT: withdrawer pays the demander 100 and records a receipt for directory "Alpha"
lattice tx receipt  --root R --chain Nexus --key withdrawer.json \
  --swap-nonce 7 --demand 100 --demander <demander-addr> --directory Alpha

# 3. Wait until AT LEAST ONE MORE parent block lands after the receipt's block.

# 4. CHILD: withdrawer claims exactly the locked amount
lattice tx withdraw --root R --chain Nexus/Alpha --key withdrawer.json \
  --swap-nonce 7 --demand 100 --demander <demander-addr> --amount 50 [--to <addr>]
```

The step-4 wait exists because the child validates the withdrawal against the
parent state *entering* the carrying parent block, so the receipt must already
be in that state. Step 4 needs the node to host both the parent and the
child.

**Warn the user before step 1.** A deposit has **no timeout, cancel or
refund**. If nobody pays the receipt and withdraws, the coins stay locked
forever.

More rules:

- `--directory` is the child's single label (`Alpha`), not a path.
- `--amount` on withdraw must equal the locked amount exactly.
- Deposits and withdrawals on Nexus are invalid, since Nexus has no parent.
- The parent does not check that a matching deposit exists. A withdrawer who
  pays a receipt for a deposit that doesn't exist loses that payment, so
  **check the deposit first**:

  ```bash
  curl -s --user "$C" "$B/api/deposit-state?key=<demander>/<amountDemanded>/<nonce>&chainPath=Nexus/Alpha" | jq
  #  value "50" = locked, "0" = already withdrawn, absent = never deposited
  curl -s --user "$C" "$B/api/receipt-state?demander=<addr>&amount=<amountDemanded>&nonce=<n>&chainPath=Nexus/Alpha" | jq
  #  chainPath here is the CHILD path; answered from the parent
  curl -s --user "$C" "$B/api/deposits?chainPath=Nexus/Alpha" | jq   # open offers
  ```

- Moving value from a grandchild to Nexus takes two separate exchanges.

## Raw transactions (no CLI)

For a script or app that signs with its own key (the SDK takes the private
key as bytes; key files store hex) and calls `POST /transactions`
directly, `references/transaction-format.md` gives the JSON body, the
signing envelope and how to build fee and nonce. That route is on the
operator port (cookie + `Content-Type: application/json`), or on a public
host that runs `publicSubmit`.

## Errors you'll see

| Response | Meaning |
|---|---|
| 400 `invalidState` | Every consensus refusal: bad signature, stale nonce, insufficient balance, missing deposit or receipt, wrong `chainPath`, a policy reject. Re-read the balance, nonce and keys to find which. |
| 400 `belowMinRelayFee` | Raise `--fee` to at least that node's `minRelayFee`. |
| 400 `feeTooLow` / `conflictingNonce` | Replacing a pending transaction without a higher fee, or the signer sets overlap at that nonce. |
| 404 `unknownChain` | This node doesn't host that `chainPath`. |
| 429 `full` | The mempool is full, or the signer has 64 parked transactions. |
| 503 `contextChanged` / `busy` | The tip moved, or the node is temporarily busy. Retry. |
| CLI `X is not in the tree` | `--chain` isn't hosted. Add it to `hostedChains`. |
| CLI `the chain has no readable tip yet` | Wait until `lattice status` shows `active`. |

For every route, see `references/http-api.md`.
