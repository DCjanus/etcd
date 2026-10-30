# Isolated freelist persistence results


The baseline is etcd main at 1573d59416efa220421c451f5266b24a5d8374ef.
Both variants use Go 1.27.3, bbolt 1.5.0, the same embedded etcd service/client harness,
and identical additional stage timers. The only product change is setting the temporary
database's NoFreelistSync to false immediately before the final catch-up transaction commit.
No new transaction, explicit sync, mapping change, background catch-up, or handle handoff is added.

Single-member Linux/XFS runs use four pinned physical CPU cores, 32 GiB cgroup memory,
zero swap, and a shared physical disk limited to 100 MiB/s and 4,000 IOPS in each direction.
The embedded server and client share this cgroup; the backend quota is 12 GiB.
This is a shared host, not a dedicated benchmark machine. Host page caches are not cleared.

The two fixtures have 300,000 and 600,000 keys with 8 KiB values. Ten percent of keys
are overwritten and the revision history is physically compacted before freezing each fixture.
Initial database sizes are 3,385,425,920 and 6,770,753,536 bytes (3.15 and 6.31 GiB).
Each case starts from a fresh copy of the same stopped member directory, then opens and
hashes it before defrag. Results therefore describe a warm-cache workload, not cold-start behavior.

Three scenarios are run: smaller fixture with no writes, larger fixture with no writes,
and larger fixture with a target 200 writes/s of 1 KiB values to unique keys.
Each scenario has three paired repetitions, ordered baseline/optimized, optimized/baseline,
baseline/optimized. A separate closed-loop reader targets 100 reads/s. These probes measure
actual RPC latency; they do not model queued arrivals or saturated throughput.
The requested write cadence is not an achieved-throughput claim: compute and report
successful write completions within the Defragment RPC window per second from the raw
request log. Completion time is reconstructed from request start and measured duration.
The constrained-I/O concurrent-write cases may
fall substantially below the target, and whole-operation read tails can remain high.

Report the upstream stopTheWorldDuration metric, the common bolt.Open stage timer,
the Defragment RPC duration, and read P99/max separately. Summaries use medians of
per-run metrics with their ranges; requests from separate runs are not merged into one percentile.

Every case checks the original revision's HashKV after defrag, reads and verifies every
fixture value, reconciles every successful load write, and checks HashKV after restarting
the member. Source assertions, raw request logs, container exit/OOM state, runtime identity,
and actual cgroup limits are retained. A separate audit replays request counts and percentiles.
Both member lifecycles are recorded in `server.jsonl`: embedded etcd reuses the
configured logger across the restart. The audit checks both startup and ready events.

These experiments do not establish production latency bounds, multi-member Raft behavior,
cold-cache performance, or power-loss durability. Benefits can vary with workload and hardware.

## Reproduction

Use a fresh checkout of the baseline commit, Go 1.27.3, and a Linux host with Docker,
XFS and the CPU IDs used by the launcher available. Copy `harness.go.txt` to
`server/experiment/freelist/main.go`. Apply `common-instrumentation.patch` with
`git apply`, then build the baseline with
`CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o baseline ./server/experiment/freelist`.
Apply `freelist-only.patch` and build the optimized executable with the same command,
changing only the output name to `freelist`.

Copy both executables, `Dockerfile`, and `run-portable.sh` into a fresh experiment
directory, generate its `binaries.sha256` using `sha256sum baseline freelist`, and
run `bash run-portable.sh`. The portable launcher differs from the measured launcher
only in locating its working directory, to avoid publishing a private host path.
The manifest retains the measured launcher and binary identities; locally rebuilt
binary hashes may differ because the original builds did not use `-trimpath`.
Raw data can also be audited without rerunning the workload using
`sha256sum -c SHA256SUMS`, `ruby audit.rb .`, and `ruby summarize.rb .`.

## Summary

All 18 cases passed the full audit. Durations below are medians of per-run metrics; no request samples are pooled across runs.

### 3.15 GiB, none

| Metric | Baseline median [range] | Optimized median [range] |
|---|---:|---:|
| Reopen | 583.0 ms [579.9 ms, 583.7 ms] | 211.5 ms [210.9 ms, 221.6 ms] |
| Final pause | 668.1 ms [664.9 ms, 668.4 ms] | 302.3 ms [297.5 ms, 306.8 ms] |
| Complete RPC | 30.753 s [30.735 s, 30.799 s] | 30.346 s [30.338 s, 30.626 s] |
| Get P99 | 0.5 ms [0.5 ms, 0.5 ms] | 0.5 ms [0.5 ms, 0.5 ms] |
| Per-run Get maximum | 662.7 ms [659.0 ms, 665.5 ms] | 298.3 ms [295.9 ms, 306.0 ms] |

### 6.31 GiB, none

| Metric | Baseline median [range] | Optimized median [range] |
|---|---:|---:|
| Reopen | 1.289 s [1.275 s, 1.296 s] | 426.3 ms [421.0 ms, 434.0 ms] |
| Final pause | 1.460 s [1.457 s, 1.470 s] | 604.7 ms [593.4 ms, 611.6 ms] |
| Complete RPC | 62.001 s [61.914 s, 62.032 s] | 60.802 s [60.703 s, 61.397 s] |
| Get P99 | 0.5 ms [0.5 ms, 0.6 ms] | 0.6 ms [0.5 ms, 0.6 ms] |
| Per-run Get maximum | 1.453 s [1.452 s, 1.461 s] | 601.3 ms [587.4 ms, 603.7 ms] |

### 6.31 GiB, single writer

| Metric | Baseline median [range] | Optimized median [range] |
|---|---:|---:|
| Reopen | 1.278 s [1.275 s, 1.292 s] | 428.9 ms [425.4 ms, 429.8 ms] |
| Final pause | 1.462 s [1.454 s, 1.471 s] | 608.3 ms [607.0 ms, 614.4 ms] |
| Complete RPC | 61.698 s [61.674 s, 61.698 s] | 61.037 s [60.639 s, 61.147 s] |
| Get P99 | 880.8 ms [879.2 ms, 882.1 ms] | 881.0 ms [880.3 ms, 883.2 ms] |
| Per-run Get maximum | 1.468 s [1.454 s, 1.924 s] | 1.681 s [942.8 ms, 1.878 s] |

Requested writes: 200/s. Observed successful write completions during the RPC window: baseline 27.07/s [27.05, 27.16], optimized 26.77/s [24.26, 27.33]. These are constrained-I/O single-writer runs, not proof of sustaining the requested rate.

## Per-run results

| Run | Reopen | Final pause | RPC total | Get P99 | Get max | Completed writes/s |
|---|---:|---:|---:|---:|---:|---:|
| n300000-rate0-r1-baseline | 583.7 ms | 668.4 ms | 30.753 s | 0.5 ms | 662.7 ms | 0.00 |
| n300000-rate0-r1-freelist | 221.6 ms | 306.8 ms | 30.346 s | 0.5 ms | 306.0 ms | 0.00 |
| n300000-rate0-r2-baseline | 579.9 ms | 664.9 ms | 30.799 s | 0.5 ms | 659.0 ms | 0.00 |
| n300000-rate0-r2-freelist | 210.9 ms | 302.3 ms | 30.338 s | 0.5 ms | 298.3 ms | 0.00 |
| n300000-rate0-r3-baseline | 583.0 ms | 668.1 ms | 30.735 s | 0.5 ms | 665.5 ms | 0.00 |
| n300000-rate0-r3-freelist | 211.5 ms | 297.5 ms | 30.626 s | 0.5 ms | 295.9 ms | 0.00 |
| n600000-rate0-r1-baseline | 1.275 s | 1.457 s | 62.032 s | 0.6 ms | 1.452 s | 0.00 |
| n600000-rate0-r1-freelist | 421.0 ms | 593.4 ms | 61.397 s | 0.6 ms | 587.4 ms | 0.00 |
| n600000-rate0-r2-baseline | 1.296 s | 1.470 s | 62.001 s | 0.5 ms | 1.461 s | 0.00 |
| n600000-rate0-r2-freelist | 434.0 ms | 604.7 ms | 60.802 s | 0.6 ms | 601.3 ms | 0.00 |
| n600000-rate0-r3-baseline | 1.289 s | 1.460 s | 61.914 s | 0.5 ms | 1.453 s | 0.00 |
| n600000-rate0-r3-freelist | 426.3 ms | 611.6 ms | 60.703 s | 0.5 ms | 603.7 ms | 0.00 |
| n600000-rate200-r1-baseline | 1.275 s | 1.454 s | 61.698 s | 882.1 ms | 1.454 s | 27.07 |
| n600000-rate200-r1-freelist | 429.8 ms | 607.0 ms | 60.639 s | 881.0 ms | 1.681 s | 24.26 |
| n600000-rate200-r2-baseline | 1.278 s | 1.462 s | 61.674 s | 879.2 ms | 1.924 s | 27.16 |
| n600000-rate200-r2-freelist | 425.4 ms | 614.4 ms | 61.147 s | 883.2 ms | 1.878 s | 27.33 |
| n600000-rate200-r3-baseline | 1.292 s | 1.471 s | 61.698 s | 880.8 ms | 1.468 s | 27.05 |
| n600000-rate200-r3-freelist | 428.9 ms | 608.3 ms | 61.037 s | 880.3 ms | 942.8 ms | 26.77 |

Raw metrics and provenance: [summary.json](summary.json), [audit.json](audit.json), and the per-run files under `results/`.
