# bpf Directory

This directory contains eBPF programs that attach to kernel functions and tracepoints.

Responsibilities:
- Collect I/O latency-related events from the Linux kernel path.
- Store and update tracing data in BPF maps.
- Share structured tracing data with user space through shared definitions in `include/`.
