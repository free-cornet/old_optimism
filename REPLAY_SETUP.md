# Replay testbed patches

This fork carries a small set of op-node changes that let the OP Stack replay a
**historical, frozen fork** of a chain for benchmarking. They are gated behind a single
flag and are **off by default**.

> These changes must never be enabled on a real (non-replay) node. They deliberately
> weaken derivation safety guarantees in ways that only make sense when L1 is frozen and
> the L2 safe head is treated as final.

Branch: `replay-setup` — based on `op-node/v1.19.5`.
Previous iteration: `replay-setup-changes` — based on `op-node/v1.18.2`.

## The flag

```
--sequencer.replay-pacing            (env: OP_NODE_SEQUENCER_REPLAY_PACING, default: false)
```

Defined in [`op-node/flags/flags.go`](op-node/flags/flags.go) (`SequencerReplayPacingFlag`),
read into `driver.Config.ReplayPacing` in [`op-node/service.go`](op-node/service.go), and
fanned out from [`op-node/rollup/driver/driver.go`](op-node/rollup/driver/driver.go) to two
independent consumers.

## What it changes

### 1. Sequencer pacing — `op-node/rollup/sequencing/sequencer.go`

When replaying a historical fork, the chain timestamp is far behind wall-clock, so
`payloadTime` is always in the past and `remainingTime < sealingDuration` always holds.
Stock op-node then sets `nextAction = now` and seals immediately, racing to "catch up" and
producing near-empty blocks every 50–100 ms.

With the flag set, the seal is scheduled at `result.BuildStarted + (BlockTime - sealingDuration)`
instead, so the execution engine gets the same fill window it has in production
(≈1950 ms for a 2 s chain). Block *timestamps* are untouched — they remain
`parent.Time + BlockTime`, so the replayed chain stays consistent.

### 2. Derivation: skip the channel-timeout walk-back — `op-node/rollup/derive/pipeline.go`

`initialReset` normally walks the L2 chain backwards until it is more than one channel
timeout behind the safe head's L1 origin, so it can buffer channel data that spans the
reset point. In the testbed L1 is frozen and the safe head is treated as final, so there
is no pre-safe-head channel data to buffer. Worse, the walk-back rewinds L1 traversal
*below* the fork origin and asks the frozen beacon node for blobs it does not have,
which turns into an endless reset loop.

`replayMode` turns the walk-back loop into a no-op (`for !dp.replayMode`), so the pipeline
starts exactly at the safe head's L1 origin.

### 3. Derivation: tolerate missing blobs — `op-node/rollup/derive/blob_data_source.go`

`GetBlobsByHash` returning `ethereum.NotFound` is normally a `ResetError`. The frozen
beacon node only holds the checkpoint slot's sidecars, so this fires for any other slot.
In replay mode the block's batch data is skipped with a warning instead, because those
(past) batches never need to be derived.

`op-node/rollup/derive/data_source.go` only threads the flag from the factory to the
blob source.

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

## Note on multi-pass block building

`--sequencer.replay-pacing` gives the execution engine a full block-time window; it does
**not** make it select from the mempool more than once per block. op-reth rebuilds the
block from scratch on each payload-job tick and keeps the highest-fee result, so the final
payload is a single global fee sort of one mempool snapshot.

Reproducing OP Mainnet's segmented intra-block ordering requires a **subblock/flashblock
builder**, i.e. `rust/op-rbuilder` (vendored at this tag) behind `rust/rollup-boost`.
See `../hypothesis/h2_multi_passes_in_bloc_building/` in the ReplaySetup repo.
