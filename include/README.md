# include Directory

This directory contains header files shared between kernel-space eBPF code and user-space control code.

Responsibilities:
- Define common data structures used by both `bpf/` and `user/`.
- Provide required kernel-structure-related headers for tracing.
- Keep ABI-compatible event/layout definitions for safe data exchange.
