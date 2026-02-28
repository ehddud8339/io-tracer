# Draft: io-tracer Project Completion Plan

## Requirements (confirmed)
- Create a project completion plan referencing `docs/io-correlation-design.md`.
- Target: Linux kernel v6.6.1 (x86-64) and NVMe-based block I/O.
- Correlate per-operation (`op_id`) across: syscall → VFS → FS → writeback → block → NVMe → IRQ/completion.
- Include buffered write + writeback correlation using Range Ledger (inode + offset-range).
- Support completion context classification: Hard IRQ / Soft IRQ / Process.

## Technical Decisions
- Implement with libbpf CO-RE (BTF required) and BPF skeleton generation.
- Kernel side: stream events; user space maintains interval tree Range Ledger and performs correlation.
- Use stable inode identity validation: `(inode*, i_ino, sb->s_dev, i_generation)` when available; otherwise `(inode*, i_ino, s_dev)` + epoch/generation.
- Support many-to-many attribution (write<->bio) with policy selection (default documented in plan).

## Research Findings
- Repo currently contains documentation only (no build system / no source code beyond docs): `docs/*.md`, `bpf/README.md`, `include/README.md`, `user/README.md`.
- Required hook sets and boundaries are already specified in:
  - `docs/syscall-definition.md`
  - `docs/vfs-definition.md`
  - `docs/fs-definition.md`
  - `docs/block-definition.md`
  - `docs/nvme-driver-definition.md`
  - `docs/completion-irq-strategy.md`
  - `docs/io-correlation-design.md`

## Open Questions
- Attribution policy default for many-to-many: last-writer vs proportional vs multi-attribution.
- Scope of filesystems: assume regular files on common local filesystems (ext4/xfs) using iomap; treat others as best-effort.
- Output format: console + JSONL/CSV; default to JSONL for event/evidence.

## Scope Boundaries
- INCLUDE: tracing read/write/fsync, buffered writeback correlation, block/NVMe submission+completion, IRQ context.
- EXCLUDE: network filesystems, exotic stacking beyond dm best-effort, full correctness across all kernel configs.
