# Replay testbed patches

Reference document for the replay-specific changes carried by this fork. It supersedes
`OP-NODE_PATCH.md` / `OP-NODE_PATCH_SOLUTION.md`, which lived in an earlier repo and
described the same work against `op-node/v1.18.2`.

This fork carries a small set of op-node changes that let the OP Stack replay a
**historical, frozen fork** of a chain for benchmarking. They are gated behind a single
flag and are **off by default**.

> These changes must never be enabled on a real (non-replay) node. They deliberately
> weaken derivation safety guarantees in ways that only make sense when L1 is frozen and
> the L2 safe head is treated as final.

| Branch | Base | Notes |
|---|---|---|
| `replay-setup` | `op-node/v1.19.5` | current — Karst; last tag that still vendors `rust/op-rbuilder` + `rust/rollup-boost` |
| `replay-setup-changes` | `op-node/v1.18.2` | previous iteration |

## The flag

```
--sequencer.replay-pacing            (env: OP_NODE_SEQUENCER_REPLAY_PACING, default: false)
```

Defined in [`op-node/flags/flags.go`](op-node/flags/flags.go) (`SequencerReplayPacingFlag`),
read into `driver.Config.ReplayPacing` in [`op-node/service.go`](op-node/service.go), and
fanned out from [`op-node/rollup/driver/driver.go`](op-node/rollup/driver/driver.go) to two
independent consumers: the sequencer and the derivation pipeline. One flag means
"replay mode" for the whole node.

---

## 1. Sequencer pacing — `op-node/rollup/sequencing/sequencer.go`

### Why

The block cycle is: `SequencerActionEvent` → start build → **seal** → block becomes
canonical → schedule next build → repeat. In a healthy node the whole 2 s cadence lives
in the *seal wait*, and that wait is also the window during which the execution engine
fills the block from the mempool.

The driver arms its timer with `time.Until(nextAction)`, which uses the **real** clock. On
a historical fork the chain timestamp is months behind wall-clock, so `payloadTime` is
always in the past, `remainingTime < sealingDuration` always holds, and stock op-node
takes the "I fell behind, catch up" branch: `nextAction = now`, seal instantly. The result
is a burst of ~900 empty blocks in a couple of minutes until the L1 sequencer-drift limit
stops it, with the historical transactions still sitting in the mempool.

### What changed

The seal is scheduled at `result.BuildStarted + (BlockTime - sealingDuration)` instead of
`now`, so the execution engine gets the same fill window it has in production (≈1950 ms
for a 2 s chain). `result.BuildStarted` is the real wall-clock instant the build began.

Block **timestamps** are untouched — still `parent.Time + BlockTime`. So are block
contents, tx selection, deposits, `NoTxPool`, the sequencer-drift rule, and every engine
API call. Only the fire time of one timer differs.

### Why this single point is enough

"Schedule next build" (`onForkchoiceUpdate`) already fires immediately in a healthy node
and is left untouched. The frozen fork only ever hit the instant-seal branch, so restoring
a real seal wait there restores the whole cadence.

### A rejected alternative

The sequencer has a mockable `timeNow` field, so one might shift it by `-offset` to make
the node believe it is at fork time. That does not work: `nextAction` is an **absolute**
timestamp anchored to the chain head, and the *driver* measures the delay to it with the
real clock. The driver has no injectable clock. Pacing the seal is the smaller change.

---

## 2. Derivation: skip the channel-timeout walk-back — `op-node/rollup/derive/pipeline.go`

### Why

On every startup and reset, `initialReset` rewinds L1 traversal to
`safe_head.L1Origin − channel_timeout` (50 blocks on this chain) so it can buffer a
channel that spans the reset point. In the testbed L1 is frozen at the fork origin and the
beacon node has no history *below* the checkpoint, so the walk-back asks for blobs of L1
blocks below the fork origin, gets a `404`, and op-node treats that as a **fatal reset** —
an endless ~2 s reset loop that seals zero blocks.

This requires the `set_finalized.py` step to have run first, so the walk-back starts from
`B_o` rather than the old finalized head.

### What changed

The walk-back loop becomes `for !dp.replayMode`, a no-op in replay mode. The pipeline then
starts derivation exactly at the safe head's L1 origin (= the frozen L1 head = the beacon
checkpoint slot), whose blobs the beacon node *does* have.

### Why it is safe here

The walk-back exists to reconstruct channels that began before the safe head. In the
replay, L1 never advances and `B_o` is declared final, so there is nothing new to derive
and no pre-safe-head channel to reconstruct — the buffer is pointless. Derivation still
runs (reads the L1 head block, drops the past batches, then idles), so steady-state
behaviour is unchanged; only the one-time reset stops rewinding into history the frozen
node cannot serve.

---

## 3. Derivation: tolerate missing blobs — `op-node/rollup/derive/blob_data_source.go`

> Not described in the old `OP-NODE_PATCH.md`, which mentioned the blob 404 only as the
> *cause* of the reset loop in §2. The code carries this as a separate, second guard.

`GetBlobsByHash` returning `ethereum.NotFound` is normally a `ResetError`. The frozen
beacon node only holds the checkpoint slot's sidecars, so this fires for any other slot
that derivation happens to reach. In replay mode the block's batch data is skipped with a
warning (`return []blobOrCalldata{}, nil`) instead, because those (past) batches never
need to be derived.

`op-node/rollup/derive/data_source.go` only threads the flag from the factory to the blob
source.

---

## Files touched

| File | Change |
|---|---|
| `op-node/flags/flags.go` | `SequencerReplayPacingFlag` + registration in `optionalFlags` |
| `op-node/service.go` | reads the flag into `driver.Config.ReplayPacing` |
| `op-node/rollup/driver/config.go` | `ReplayPacing bool` field |
| `op-node/rollup/driver/driver.go` | passes it to `NewDerivationPipeline` and `NewSequencer` |
| `op-node/rollup/sequencing/sequencer.go` | `replayPacing` field, ctor param, seal scheduling |
| `op-node/rollup/derive/pipeline.go` | `replayMode` field, ctor param, walk-back skip |
| `op-node/rollup/derive/data_source.go` | `replayMode` plumbing |
| `op-node/rollup/derive/blob_data_source.go` | `replayMode` + skip on blob 404 |
| `op-node/rollup/sequencing/sequencer_test.go` | call site (`false`) |
| `op-node/rollup/derive/altda_data_source_test.go` | call sites (`false`) |
| `op-e2e/actions/helpers/l2_sequencer.go` | call site (`false`) |
| `op-e2e/actions/helpers/l2_verifier.go` | call site (`false`) |

No changes to `rust/op-reth` or any other Rust component.

---

## Running it

```bash
op-node ... --sequencer.replay-pacing=true
# or
OP_NODE_SEQUENCER_REPLAY_PACING=true op-node ...
```

Companion settings for the replay rig:

- **Sequencer enabled and not stopped.** The flag only affects a running sequencer. The
  testbed starts with `--sequencer.stopped=true` and calls `admin_startSequencer` when the
  benchmark begins.
- **Leave `--sequencer.max-safe-lag` at `0`.** With a frozen L1 the L2 *safe* head never
  advances, so any non-zero value stalls the sequencer after that many unsafe blocks.
- **Run `set_finalized.py` first**, while op-reth is up and op-node is stopped. §2 depends
  on the safe head already being at `B_o`.
- **Prime op-reth's mempool** with the first batch of historical transactions before
  starting the sequencer, or accept that the first block may be near-empty — the first
  build starts immediately on sequencer start.

Operational expectations:

- Blocks are produced ~one per `BlockTime` in **real** time.
- Historical transactions from the mempool are included as normal.
- The run still stops at the inherent **~900-block / ~30-minute sequencer-drift limit**.
  This was deliberately *not* removed. After that, blocks become empty (`NoTxPool`) —
  that is the replay-window boundary, not a malfunction.

## Validating a build

```bash
go build ./op-node/...
go test ./op-node/rollup/sequencing/...   # sequencer unit tests
go test ./op-node/rollup/derive/...       # pipeline + data source
go test ./op-node/flags/...               # flag uniqueness / env-var format
```

The flag tests (`TestUniqueFlags`, `TestBetaFlags`, `TestEnvVarFormat`, `TestHasEnvVar`)
are the ones most likely to catch a naming slip; the flag is named to satisfy them
(`sequencer.replay-pacing` ↔ `OP_NODE_SEQUENCER_REPLAY_PACING`).

Then, in the rig, confirm three things:

1. Blocks land ~`BlockTime` apart in real wall-clock time — compare block timestamps
   against the log timestamps of "Sequencer sealed block".
2. Historical transactions from op-reth's mempool appear in the expected blocks, i.e.
   op-reth really does fill past-dated payloads during the window. This is the one
   cross-component assumption the patch rests on.
3. The run stops cleanly at the drift limit rather than misbehaving.

## Reverting / maintenance

- Runtime: drop the flag, the default is off.
- Permanently: revert the edits above. They are self-contained and touch no shared logic.
- On a future op-node upgrade, re-apply and re-thread the flag. The things to re-check
  upstream are that the seal is still scheduled from `BuildStarted` / `payloadTime`, and
  that `initialReset` still performs the channel-timeout walk-back in the same loop.
  Both moved between v1.18.2 and v1.19.5 — the seal scheduling migrated from the
  `onBuildStarted` event handler into `startBuildingBlock`, and `NewDerivationPipeline`
  lost its `managedBySupervisor` parameter.

---

## Note on multi-pass block building

`--sequencer.replay-pacing` gives the execution engine a full block-time window; it does
**not** make it select from the mempool more than once per block. op-reth rebuilds the
block from scratch on each payload-job tick and keeps the highest-fee result, so the final
payload is a single global fee sort of one mempool snapshot.

**op-reth does not build subblocks, at any version.** `rust/op-reth/crates/flashblocks/`
is a consumer — its own header says "A downstream integration of Flashblocks" — and
`--flashblocks-url` / `--subblocks-url` subscribe to a stream produced elsewhere.
`rust/op-reth/crates/payload/` contains zero references to flashblocks or subblocks. The
spec agrees: `specs.optimism.io/protocol/flashblocks.html` defines the builder as an
*External Block Builder*, separate from the execution client, and names the reference
implementations as rollup-boost, op-rbuilder, flashblocks-websocket-proxy and
reth-flashblocks — the last being "the execution client modification", i.e. the consumer.

Reproducing OP Mainnet's segmented intra-block ordering requires a **subblock/flashblock
builder**: `rust/op-rbuilder` (vendored at this tag) behind `rust/rollup-boost`. Both were
removed from `develop` in `e92bda111c`, which is the main reason this branch is based on
`op-node/v1.19.5`.

See `../hypothesis/h2_multi_passes_in_bloc_building/` and `../Modifications_OP_STACK.md`
in the ReplaySetup repo.
