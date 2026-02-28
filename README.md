# io-tracer

Linux I/O stack latency tracing project based on eBPF.

## Directory Structure

- [`bpf/`](bpf/): eBPF programs that attach to kernel functions/tracepoints and collect latency events.
- [`include/`](include/): shared headers (kernel/user common structs, tracing-related definitions, and vmlinux-related headers).
- [`user/`](user/): user-space controller that loads eBPF programs, receives events, prints results, and stores output files.
