---
name: lattice-mining
description: Mine Lattice (Nexus plus every hosted child chain at once) with lattice-node — the miner's settings in miner.json, `lattice mine start/stop/status/run`, running `lattice-mining-coordinator` directly or under systemd, mining through a node on another machine, reward recipients, checking rewards, GPU or remote workers and the worker contract, and reading the miner's log. Use when a user wants to mine, set payout addresses, check mining rewards, plug in a GPU miner, or debug why mining isn't producing blocks.
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

The operator page is https://lattice.build/docs/run/mining/.

## Quick start (CLI-managed)

```bash
lattice key generate --out ./miner-wallet.json             # copy the address:
```

Write `<root>/miner.json`. It is the miner's own file: the node never reads
it, and mining settings do not go in `lattice.json`.

```json
{
  "worker": "cpu",
  "workers": 4,
  "recipients": { "Nexus": "<address>", "Nexus/Alpha": "<address>" }
}
```

Then:

```bash
lattice mine start  --root R      # background; pidfile R/run/mine.pid, log R/log/mine.log
lattice mine status --root R      # running, the node it mines through, recipients
lattice logs --miner --root R     # the mining loop's recent output
lattice mine stop   --root R      # finishes or bounds the active round, then stops
```

`recipients: none configured (rewards and fees burn)` in `mine status` is a
red flag. **Every chain without a recipient burns its reward and fees.** No
key is needed on the mining host, only addresses. Rewards and fees go to the
block's `rewardRecipient`; there are no reward transactions.

**A changed `miner.json` applies to the next round without a restart.** The
loop reads the file again before each round. A file that no longer checks is
logged and the last good settings are kept. So after hosting a new child,
add its recipient to `miner.json`; until then its rewards burn.

## miner.json keys

| Key | Default | Meaning |
|---|---|---|
| `worker` | `cpu` | `cpu` for the bundled worker, or the path of an executable that follows the worker contract. |
| `workers` | 1 | How many workers to run. |
| `batchSize` | 2000000000 | Nonces a worker tries per batch. |
| `recipients` | none | The address each chain's reward and fees go to, by chain path. |
| `minWork` | scheduled target | A minimum work per block by chain path, such as `{"Nexus": "2^32"}`. A local filter on what the miner submits. |
| `minBlockIntervalSeconds` | none | The shortest gap between two blocks this miner produces. |
| `templateTimeoutSeconds` | 60 | How long to wait for the node to answer a template request. 60 is also the most it can be. |
| `roundDeadlineMultiplier` | 10 | Headroom on the round deadline. |

An unknown key is refused. `--config <file>` names another settings file.

## Mining through a node on another machine

```bash
# Beside the node: a token with the three routes a miner needs.
lattice tokens create miner --out miner.token --root /var/lib/lattice \
  --route "POST /mining/templates" --route "POST /mining/work" --route "GET /status"

# On the mining machine, with its own root for settings, pidfile and log:
lattice mine start --root /var/lib/lattice-miner \
  --node http://10.0.0.5:8090 --token-file miner.token
```

The node must have its remote operator listener on (`operatorListen`; see
the `lattice-node-operator` skill).

## Under a process manager

- `lattice emit-systemd --root R` prints `lattice-mine.service` beside the
  node's unit when the root holds miner settings. It runs
  `lattice mine run`, the same loop in the foreground.
- Or run the coordinator yourself, as the lattice-node repository's
  `deploy/lattice-mining-coordinator.service` does. That unit reads
  `RECIPIENT_ARGS` from `/etc/lattice/mining.env` (mode 0640) and refuses to
  start when it names no recipient.

Use one or the other for a node, not both.

```bash
lattice-mining-coordinator \
  --node http://127.0.0.1:8080 \
  --rpc-cookie-file /var/lib/lattice/chains/Nexus/.cookie \
  --worker-executable /usr/local/bin/lattice-miner \
  --workers 2 \
  --recipient Nexus=<address> --recipient Nexus/Alpha=<address>
```

Check `lattice-mining-coordinator --help` for the other flags (`--once`,
`--batch-size`, `--min-work`, `--token-file`).

## Checking rewards

```bash
C="$(cat R/chains/Nexus/.cookie)"; B=http://127.0.0.1:8080
curl -s --user "$C" "$B/api/block/latest?chainPath=Nexus/Alpha" | jq '{height, rewardRecipient, rewardCredited}'
curl -s --user "$C" "$B/api/blocks?limit=25" | jq '.blocks[] | {height, rewardRecipient}'
curl -s --user "$C" "$B/api/state/account/<address>?chainPath=Nexus/Alpha" | jq .balance
```

`rewardCredited` is the reward plus fees; `"0"` means it was burned. There is
no rewards CLI. Remember there is no finality: a block you mined can be
reorganized out.

## How merged mining picks targets

- The template carries Nexus and the latest candidate block of each hosted
  child, recursively. Its `searchTarget` is the **easiest** target it
  carries.
- A hit that clears only some targets returns the disposition `childOnly`.
  The coordinator submits it and then **continues** toward the harder ones.
- A new transaction or child candidate changes the template, and the miner
  fetches new work even when the Nexus tip did not move. Work already
  issued stays valid: a solution on an old tip becomes a side block.
- Child geneses use the easiest target, so a new child is mined in almost
  immediately once its parent is ready.
- `minWork` only filters which hits the miner submits. Blocks still commit
  their scheduled target. A filter on a chain the work still carries stops
  the whole search at that filter, so prefer `minBlockIntervalSeconds` for
  pacing.

## GPU and remote workers

Set `worker` in `miner.json` (or `--worker-executable`) to any executable
that follows the contract in `references/worker-contract.md`. The main
rules:

- Print exactly one JSON line on stdout.
- Report the **lowest** winning nonce.
- Exit 0 whether or not you found a solution.

No GPU worker ships in the lattice-node repo.

## Reading the miner's log

| Line | Meaning / action |
|---|---|
| `accepted tip=…` | A block was accepted. |
| `NOT MINING: no usable template answer …` | The node gave no usable answer within `templateTimeoutSeconds`. Check that the node runs and is synced (`lattice status`). It retries. |
| `ROUND DEADLINE EXCEEDED` | The round ran past its deadline and was abandoned; the next round retries. Raise `roundDeadlineMultiplier` in `miner.json` if rounds here really run that long. |
| `ROUND TEARDOWN DEGRADED …` | A worker's processes may have leaked. Run `pgrep lattice-miner` and kill any strays. |
| `the node refused this round's work; retrying` | The work went stale or was invalid. One occurrence is normal; if it repeats, check that node and worker versions match. |
| `pacing: holding …` | `minBlockIntervalSeconds` is in effect. |
| `mining on with the settings read before: …` | `miner.json` no longer checks; fix it. |

Mining dispositions from `POST /mining/work`: `canonicalized` (new tip),
`acceptedSide` (a valid side block), `childOnly`, `duplicate`, `invalid`.

## Gotchas

- Running `mine start` twice fails with `mining already running`.
- `no miner settings at …/miner.json`: write the file.
- An older `lattice.json` may hold a `mine` object. The node ignores it.
  While there is no `miner.json`, the miner copies it there once.
- A nested child can't start until its parent changes state. A parent whose
  rewards burn (no recipient) and which gets no transactions never will.
- Use one release for all four binaries.
