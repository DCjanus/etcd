# Isolated freelist experiment

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
