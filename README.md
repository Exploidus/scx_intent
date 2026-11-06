# scx_intent

`scx_intent` is an intent-aware, multi-tier CPU scheduler for Linux built on `sched_ext` using in-kernel BPF.

Instead of treating all threads with homogeneous timeslices or relying solely on runtime heuristics, `scx_intent` assigns tasks into 5 dedicated BPF Dispatch Queues (DSQs) based on their execution profile. Interactive UI, compositor, and audio tasks receive short, low-latency slices, while heavy throughput/batch tasks run on wider slices to minimize cache thrashing and context-switch overhead.

## Architecture

Tasks are classified inside BPF at enqueue time (`ops.enqueue`) and dispatched across 5 priority-ranked DSQs:

| Intent Tier | Target Workloads | DSQ ID | Time Slice | Precedence |
|-------------|------------------|--------|------------|------------|
| `INTENT_AUDIT` | Security logging, auditd, live system monitors | 0 | 10 ms | 1 (Highest) |
| `INTENT_INTERACTIVE` | Wayland compositors, X11, audio engines (PipeWire), shells | 1 | 4 ms | 2 |
| `INTENT_IO` | Storage I/O workers, database sync/flush operations | 2 | 5 ms | 3 |
| `INTENT_NETWORK` | Socket handlers, network packet ingress/egress workers | 3 | 8 ms | 4 |
| `INTENT_COMPUTE` | Compilers (rustc, gcc), batch computations, rendering | 4 | 20 ms | 5 (Lowest) |

### Dispatching & Multi-Tier Anti-Starvation

1. **In-Kernel BPF Routing**: Dispatch decisions happen entirely in kernel space via BPF iterator and `scx_bpf_dispatch()`. No userspace IPC trips in the critical path.
2. **Prioritized Drain**: Higher-ranked DSQs are evaluated first on dispatch (`Audit` -> `Interactive` -> `IO` -> `Network` -> `Compute`).
3. **Starvation Prevention**: Under sustained compute saturation, lower-priority DSQs track missed dispatch opportunities. Once a tier's starvation counter reaches the configured threshold (default: 32 ticks), it is injected directly into the dispatch queue to guarantee forward progress across all tasks.

## Requirements

- Linux Kernel 6.12 or newer with `CONFIG_SCHED_CLASS_EXT=y`
- Clang / LLVM 17+
- Rust 1.80+ (with Cargo)
- `bpftool` and `libbpf` development headers

## Building & Running

Clone the repository and build with release optimizations:

```bash
git clone https://github.com/Exploidus/scx_intent.git
cd scx_intent

# Run with standard desktop preset
cargo run --release

# Run with gaming / low-latency preset
cargo run --release -- --preset gaming

# Run with custom JSON configuration
cargo run --release -- --config intent.example.json
```

To inspect runtime parameters and options:

```bash
cargo run --release -- --help
```

## Origins

The concept of dedicated intent classification originally stems from design work on the [Exploidus](https://github.com/Exploidus) kernel, reimagined and natively rewritten for the Linux `sched_ext` infrastructure.

## License

GPL-2.0-only
