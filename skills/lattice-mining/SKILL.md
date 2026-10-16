---
name: lattice-mining
description: Mine Lattice (Nexus plus every hosted child chain at once) with lattice-node — the `mine` block in lattice.json, `lattice mine start/stop/status`, running `lattice-mining-coordinator` directly or under systemd, reward recipients, checking rewards, GPU or remote workers and the worker contract, and reading mine.log. Use when a user wants to mine, set payout addresses, check mining rewards, plug in a GPU miner, or debug why mining isn't producing blocks.
---

# Mining Lattice

Mining is **external** to the node, and three roles share the work:

| Role | Binary | Does |
|---|---|---|
| node | `lattice-node` | builds templates (Nexus + all hosted children), validates solutions, merged-mining proofs |
| coordinator | `lattice-mining-coordinator` | fetches templates, splits nonce ranges, detects stale work, submits |
| worker | `lattice-miner` (CPU), or any binary that follows the contract | searches one nonce range, prints one JSON line |

One hash is checked against every hosted chain's target. **You mine exactly
the chains your node hosts.** To mine a child, host it.

## Quick start (CLI-managed)

```bash
lattice key generate --out ./miner-wallet.json             # copy the address:
```

Add a `mine` block to `<root>/lattice.json`:

```json
"mine": {
  "worker": "cpu",
  "workers": 4,
  "recipients": { "Nexus": "<address>", "Nexus/Alpha": "<address>" }
}
```

Then:

```bash
lattice mine start  --root R      # background; log at R/log/mine.log
lattice mine status --root R      # running + recipients ("none configured (rewards and fees burn)" is a red flag)
lattice mine stop   --root R      # finishes the batch, 60 s grace
```

**Every chain without a recipient burns its reward and fees.** No key is
needed on the mining host, only addresses. Rewards and fees go to the
header's `rewardRecipient`; there are no reward transactions.

**Restart mining after any config change.** `lattice mine` reads
`lattice.json` once at start. A child added later gets no recipient until
you run `lattice mine stop && lattice mine start`, and its rewards burn in
the meantime.

All `mine` keys (`batchSize`, `minWork`, `minBlockIntervalSeconds`,
`templateTimeoutSeconds`, `roundDeadlineMultiplier`) are documented in the
`lattice-node-operator` skill, `references/lattice-json.md`.

## Production: coordinator under systemd

```bash
lattice-mining-coordinator \
  --node http://127.0.0.1:8080 \
  --rpc-cookie-file /var/lib/lattice/chains/Nexus/.cookie \
  --worker-executable /usr/local/bin/lattice-miner \
  --workers 2 \
  --recipient Nexus=<address> --recipient Nexus/Alpha=<address>
```

| Flag | Default | Notes |
|---|---|---|
| `--node` | required | node RPC base URL |
| `--rpc-cookie-file` | none | re-read on every request |
| `--workers` | CPU count − 1 | |
| `--batch-size` | 2e9 with an external worker | with `--once` this is the total across all workers |
| `--worker-executable` | in-process search | path to the worker binary |
| `--recipient P=addr` | none | repeat once per chain |
| `--min-work P=work` | none | local filter (`2^N` or a decimal) |
| `--once` | off | one batch, one JSON line, then exit |
| `--no-stale-probe` | off | disables the 2 s freshness probe |

The repo's `deploy/lattice-mining-coordinator.service` reads `RECIPIENT_ARGS`
from `/etc/lattice/mining.env` (mode 0640) and refuses to start if the value
is empty. Alternatively, `lattice emit-systemd` prints a
`lattice-mine.service` that runs `lattice mine run --root R`. Split it out
of the output, as described in the operator skill.

## Checking rewards

```bash
C="$(cat R/chains/Nexus/.cookie)"; B=http://127.0.0.1:8080
curl -s --user "$C" "$B/api/block/latest?chainPath=Nexus/Alpha" | jq '{height, rewardRecipient, rewardCredited}'
curl -s --user "$C" "$B/api/blocks?limit=25" | jq '.blocks[] | {height, rewardRecipient}'
curl -s --user "$C" "$B/api/state/account/<address>?chainPath=Nexus/Alpha" | jq .balance
```

`rewardCredited` is the reward plus fees; `"0"` means it was burned. There is
no rewards CLI. Remember there is no finality: a block you mined can be
reorged out.

## How merged mining picks targets

- The template carries Nexus and the latest candidate block of each hosted
  child, recursively. The node sets `searchTarget` to the **easiest** target
  that advances at least one chain.
- A hit that clears only child targets returns the disposition `childOnly`.
  The coordinator submits it and then **continues** toward harder targets.
- Child geneses use the easiest target, so a new child is mined in almost
  immediately once its parent is ready.
- `minWork` only filters which hits the miner keeps. It never changes
  consensus difficulty. A binding `minWork` cuts the difficulty feedback
  loop, so prefer `minBlockIntervalSeconds` for pacing. Either one throttles
  the **whole subtree**, because children are mined together with Nexus.

## GPU and remote workers

Set `mine.worker` (or `--worker-executable`) to any executable that follows
the contract in `references/worker-contract.md`. The main rules:

- Print exactly one JSON line on stdout.
- Report the **lowest** winning nonce.
- Exit 0 whether or not you found a solution.

No GPU worker ships in the lattice-node repo.

## Reading mine.log

| Line | Meaning / action |
|---|---|
| `accepted tip=…` | A block was accepted. |
| `NOT MINING: no usable template answer …` | The node is bootstrapping (409), unavailable (503) or slow (more than `templateTimeoutSeconds`, max 60). Wait for sync, then check `lattice status`. It retries every 5 s. |
| `ROUND DEADLINE EXCEEDED` | The round ran longer than the deadline. Raise `roundDeadlineMultiplier` if your rounds really are that long. |
| `ROUND TEARDOWN DEGRADED … stray lattice-miner` | A worker subtree may have leaked. Run `pgrep lattice-miner` and kill any strays. |
| `the node refused this round's work; retrying` | The work went stale or was invalid. One occurrence is normal; if it repeats, check node and worker versions match. |
| `pacing: holding …` | `minBlockIntervalSeconds` is in effect. |

Mining dispositions from `POST /mining/work`: `canonicalized` (new tip),
`acceptedSide` (a valid side block), `childOnly`, `duplicate`, `invalid`.

## Gotchas

- Running `mine start` twice fails with `mining already running`.
- `no [mine] section in lattice.json`: add the `mine` block.
- `mining configuration names unhosted chain P`: either host P, or remove it
  from `recipients` and `minWork`.
- A nested child can't start until its parent changes state. A parent whose
  rewards burn (no recipient) and which gets no transactions never will.
- Use one release for all four binaries.
