# io-tracer Completion Plan (Linux 6.6.1)

## TL;DR
> **Summary**: Implement a libbpf CO-RE based eBPF tracer + userspace correlator that assigns an `op_id` at syscall entry and correlates buffered I/O through writeback, block, NVMe, and IRQ/completion contexts to produce per-op latency breakdowns.
> **Deliverables**: eBPF programs under `bpf/`, shared ABI headers under `include/`, userspace loader/correlator under `user/`, self-test workload + assertions, and JSONL output schema.
> **Effort**: XL
> **Parallel**: YES - 4 waves
> **Critical Path**: Build/tooling scaffold → event ABI + BPF hooks → userspace Range Ledger + correlation → selftest + verification

## Context
### Original Request
- “`docs/io-correlation-design.md`를 참고하여 프로젝트 완성 계획을 작성하라.”

### Interview Summary
- Repository is currently documentation-heavy; no build system detected; directories `bpf/`, `include/`, `user/`, `docs/` exist.
- Canonical design constraints are defined in `docs/io-correlation-design.md` and companion boundary docs.

### Metis Review (gaps addressed)
- Added explicit agent-executable acceptance criteria (build + run + jq assertions).
- Added guardrails for hook stability, event loss telemetry, and correlation confidence reporting.

## Work Objectives
### Core Objective
- Correlate each read/write/fsync operation (op_id) across:
  - syscall → VFS/FS → write path ledger update → writeback→bio creation → block submission → NVMe submission/completion → request/bio completion → IRQ context

### Deliverables
- Kernel/eBPF:
  - Event stream (ringbuf) for all required hooks with stable ABI structs.
  - Pointer-based transient maps: `tid->op_id`, `iocb->op_id`, `bio->op_id_set`, `rq->op_id_set` with reuse-safety.
  - IRQ/softirq nesting state for completion context classification.
- Userspace:
  - Loader/attacher (libbpf) + ringbuf consumer.
  - Interval-tree Range Ledger keyed by inode identity + (offset range) with epoch/versioning.
  - Correlator that assigns contributor sets to writeback bios and computes per-op latency metrics.
  - JSONL output (events + per-op spans) + loss/health telemetry.
- Verification:
  - `user/selftest.sh` workload using only base tools (`dd`, `sync`) and a small helper for fsync.
  - Deterministic assertions (jq) for schema presence, non-negative durations, and context classification presence.

### Definition of Done (verifiable conditions with commands)
- Build succeeds and produces runnable binary + BPF object:
  - `make build`
- Self-test runs end-to-end and exits 0:
  - `sudo ./user/io-tracer --selftest --output /tmp/io-tracer.jsonl`
- Output invariants hold (examples):
  - `test -s /tmp/io-tracer.jsonl`
  - `jq -e -s 'any(.[]; .type=="op" and .op_type=="WRITE" and (.op_id!=null) and (.inode!=null) and (.stages.syscall!=null) and (.stages.block!=null) and (.stages.nvme!=null) and (.completion.context!=null))' /tmp/io-tracer.jsonl >/dev/null`
  - `jq -e -s 'all(.[]; (.type!="op") or ((.stages|to_entries|map(.value.duration_ns)|all(.>=0))))' /tmp/io-tracer.jsonl >/dev/null`

### Must Have
- Linux v6.6.1 assumptions baked in; capability detection for missing hooks.
- Buffered write + writeback correlation via Range Ledger; many-to-many attribution support.
- Completion context classification via IRQ/softirq tracepoints + `in_hardirq()`/`in_serving_softirq()` markers.
- Loss counters and correlation failure reasons surfaced.

### Must NOT Have (guardrails, AI slop patterns, scope boundaries)
- Must not rely on a single pointer key without validation (inode/bio/rq reuse).
- Must not require manual/visual validation; all QA must be scripted.
- Must not claim additive latency decomposition unless explicitly defined as wall-clock vs overlapping spans.
- Must not expand to network filesystems or full dm/raid topology beyond best-effort labeling.

## Verification Strategy
> ZERO HUMAN INTERVENTION — all verification is agent-executed.
- Test decision: tests-after (selftest + jq assertions) + optional unit tests for interval tree.
- QA policy: Every task includes at least one runnable scenario and evidence artifact.
- Evidence: `.sisyphus/evidence/task-{N}-{slug}.txt` and `.sisyphus/evidence/task-{N}-{slug}.jsonl`

## Execution Strategy
### Parallel Execution Waves
Wave 1: Tooling + ABI + userspace scaffolding
Wave 2: Core hooks (syscall/VFS/FS/writeback/block/NVMe) + IRQ context
Wave 3: Correlation engine + latency model + attribution policies + fsync snapshot matching
Wave 4: Selftest, performance guardrails, docs polish, CI (optional)

### Dependency Matrix (full, all tasks)
- Wave 1 tasks block all later waves.
- Hook tasks can be implemented in parallel once ABI is fixed.
- Userspace correlator can be implemented in parallel with BPF hook implementation after ABI is fixed.

### Agent Dispatch Summary (wave → task count → categories)
- Wave 1: 6 tasks → quick/unspecified-low
- Wave 2: 9 tasks → unspecified-high
- Wave 3: 6 tasks → deep/ultrabrain
- Wave 4: 5 tasks → quick/writing

## TODOs
> Implementation + Test = ONE task. Never separate.
> EVERY task MUST have: Agent Profile + Parallelization + QA Scenarios.

- [ ] 1. Establish Build + Dependency Scaffold (libbpf CO-RE)

  **What to do**: Create a reproducible build pipeline that can compile BPF objects and a userspace loader.
  - Add a top-level `Makefile` with targets: `build`, `clean`, `test`, `selftest`, `gen`.
  - Vendor libbpf as `third_party/libbpf` (git submodule) and pin a known-good tag/commit.
  - Add `tools/` scripts:
    - `tools/gen_vmlinux_h.sh` (uses `/sys/kernel/btf/vmlinux` + `bpftool btf dump`)
    - `tools/gen_skel.sh` (uses `bpftool gen skeleton`)
  - Define build outputs:
    - BPF object: `bpf/io_tracer.bpf.o`
    - Skeleton header: `bpf/io_tracer.skel.h`
    - Userspace binary: `user/io-tracer`

  **Must NOT do**: Do not depend on system-installed libbpf headers without pinning; do not require manual steps beyond `make`.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: build system + libbpf integration has many edge cases
  - Skills: [`git-master`] — Reason: submodule pinning and clean history
  - Omitted: [`playwright`] — Reason: no browser workflows

  **Parallelization**: Can Parallel: NO | Wave 1 | Blocks: 2-26 | Blocked By: none

  **References**:
  - External: https://github.com/libbpf/libbpf-bootstrap — layout + bpftool skeleton workflow
  - External: https://github.com/anakryiko/bpf-ringbuf-examples — ringbuf reserve/submit patterns
  - Repo: `README.md` — directory intent

  **Acceptance Criteria** (agent-executable only):
  - [ ] `make build` exits 0 and creates `bpf/io_tracer.bpf.o`, `bpf/io_tracer.skel.h`, `user/io-tracer`

  **QA Scenarios**:
  ```
  Scenario: Build from clean tree
    Tool: Bash
    Steps: make clean && make build
    Expected: exit 0; outputs present
    Evidence: .sisyphus/evidence/task-1-build.txt

  Scenario: Rebuild is deterministic
    Tool: Bash
    Steps: make clean && make build && make build
    Expected: second build exits 0 without regeneration failures
    Evidence: .sisyphus/evidence/task-1-rebuild.txt
  ```

  **Commit**: YES | Message: `build: add libbpf CO-RE build scaffold` | Files: `Makefile`, `tools/*`, `third_party/libbpf/*`

- [ ] 2. Define Shared ABI + Event Schema (include/)

  **What to do**: Create the stable kernel↔userspace ABI used everywhere.
  - Add `include/io_tracer_abi.h` defining:
    - `enum io_op_type { READ, WRITE, FSYNC }`
    - `struct inode_id { __u64 inode_ptr; __u64 s_dev; __u64 i_ino; __u32 i_generation; __u32 epoch; }`
    - `enum completion_ctx { HARDIRQ, SOFTIRQ, PROCESS }`
    - `struct event_header { __u64 ts_ns; __u32 cpu; __u32 pid; __u32 tid; __u16 type; __u16 flags; }`
    - Event variants for each hook family (syscall, vfs/fs, ledger_update, wb_ioend, submit_bio, bio_to_rq, nvme, completion, irq_window, health)
  - Decide output record types (userspace JSONL) and map each to ABI event types.

  **Must NOT do**: Must not expose raw kernel pointers in output without also including stable identifiers + reuse safeguards; keep structs packed and versioned.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: ABI mistakes cascade
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 1 | Blocks: 7-21 | Blocked By: 1

  **References**:
  - Repo: `docs/io-correlation-design.md` — op_id, inode identity, Range Ledger, required events
  - Repo: `docs/completion-irq-strategy.md` — context classification fields

  **Acceptance Criteria**:
  - [ ] `make build` succeeds after adding ABI header
  - [ ] ABI header includes a version macro `IO_TRACER_ABI_VERSION` and all required types

  **QA Scenarios**:
  ```
  Scenario: ABI compiles in BPF and userspace
    Tool: Bash
    Steps: make clean && make build
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-2-abi-build.txt
  ```

  **Commit**: YES | Message: `feat(abi): define shared event schema and inode identity` | Files: `include/io_tracer_abi.h`

- [ ] 3. Userspace Loader Skeleton + JSONL Writer

  **What to do**: Implement a minimal but complete runner.
  - Add `user/io-tracer.c` with CLI:
    - `--duration <sec>` (default 10)
    - `--output <path>` (default stdout)
    - `--policy <last-writer|proportional|multi>` (default `multi`)
    - `--selftest` (runs built-in workload)
  - Load and attach `bpf/io_tracer.skel.h`.
  - Consume ringbuf and write JSONL records.
  - Always emit periodic `health` records (drops, map_fail, correlation_lost).

  **Must NOT do**: Must not buffer unbounded in RAM; write streaming JSONL.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: libbpf attach + streaming output
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 1 | Blocks: 16-21 | Blocked By: 1, 2

  **References**:
  - External: https://github.com/libbpf/libbpf-bootstrap
  - Repo: `user/README.md`

  **Acceptance Criteria**:
  - [ ] `./user/io-tracer --help` exits 0
  - [ ] `sudo ./user/io-tracer --duration 1 --output /tmp/trace.jsonl` exits 0 and creates a non-empty file

  **QA Scenarios**:
  ```
  Scenario: Attach and stream health records
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 1 --output /tmp/trace.jsonl
    Expected: /tmp/trace.jsonl exists and contains at least one JSON line
    Evidence: .sisyphus/evidence/task-3-run.txt
  ```

  **Commit**: YES | Message: `feat(user): add loader skeleton and JSONL streaming` | Files: `user/io-tracer.c`

- [ ] 4. Implement Userspace Range Ledger (Interval Tree)

  **What to do**: Implement the core correlation data structure in userspace.
  - Add `user/range_ledger.h` + `user/range_ledger.c` implementing:
    - Insert/update `[start,end]` for WRITE with split/replace semantics
    - `contributors` maintenance (op_id->bytes) per policy
    - Epoch bump API for truncate/hole-punch invalidation
    - Lookup by `(inode_id, range)` returning contributor set (possibly merged)
  - Add a deterministic unit test binary `user/range_ledger_test` run by `make test`.

  **Must NOT do**: Must not require external libraries; keep interval tree self-contained.

  **Recommended Agent Profile**:
  - Category: `ultrabrain` — Reason: interval tree correctness + many-to-many attribution
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 1 | Blocks: 18-21 | Blocked By: 2, 3

  **References**:
  - Repo: `docs/io-correlation-design.md` — Range Ledger definition + overwrite/truncate rules
  - Repo: `docs/fs-definition.md` — buffered write uses generic_* common paths

  **Acceptance Criteria**:
  - [ ] `make test` exits 0 and runs `user/range_ledger_test`
  - [ ] Unit tests cover: non-overlap insert, partial overlap split, full overwrite, epoch bump invalidation

  **QA Scenarios**:
  ```
  Scenario: Range Ledger unit tests
    Tool: Bash
    Steps: make test
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-4-ledger-test.txt
  ```

  **Commit**: YES | Message: `feat(corr): add interval-tree range ledger with policies` | Files: `user/range_ledger.*`, `user/range_ledger_test.*`

- [ ] 5. Implement BPF Program Skeleton + Core Maps

  **What to do**: Create the base eBPF program file and maps used across all hooks.
  - Add `bpf/io_tracer.bpf.c` including `vmlinux.h` and `include/io_tracer_abi.h`.
  - Define maps:
    - `events` ringbuf
    - `op_seq` (global op_id counter) as `BPF_MAP_TYPE_ARRAY` (single u64)
    - `tid_to_op` hash (key: tid u32, value: op_id + type + fd + ts_enter)
    - `irq_state` per-cpu array (hardirq_depth, softirq_depth)
    - `tracked_bios` hash (key: bio_ptr u64, value: inode_id + off_start + len + ts_wb_submit)
    - `tracked_rqs` hash (key: rq_ptr u64, value: bio_ptr u64 + ts_issue + ts_complete)
  - Implement helper functions:
    - `next_op_id()` using atomic fetch-add
    - `emit_event_*()` helpers for each event type
    - `ctx_classify()` using irq_state depths

  **Must NOT do**: Do not emit unfiltered `bio_endio`/block events for all bios; use `tracked_bios` / `tracked_rqs` filtering.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: BPF CO-RE + verifier constraints
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 6-15 | Blocked By: 1, 2

  **References**:
  - External: https://github.com/anakryiko/bpf-ringbuf-examples — reserve/submit pattern
  - Repo: `docs/io-correlation-design.md` — required maps and pointer mapping tables

  **Acceptance Criteria**:
  - [ ] `make build` succeeds and produces `bpf/io_tracer.bpf.o`

  **QA Scenarios**:
  ```
  Scenario: BPF compiles under CO-RE
    Tool: Bash
    Steps: make clean && make build
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-5-bpf-build.txt
  ```

  **Commit**: YES | Message: `feat(bpf): add core program skeleton and maps` | Files: `bpf/io_tracer.bpf.c`

- [ ] 6. Syscall Layer Hooks: op_id Creation + Timing

  **What to do**: Implement syscall entry/exit collection and op_id lifecycle.
  - Attach to syscall tracepoints for v6.6.1 x86-64:
    - `tracepoint/syscalls/sys_enter_read`, `.../sys_exit_read`
    - `tracepoint/syscalls/sys_enter_write`, `.../sys_exit_write`
    - `tracepoint/syscalls/sys_enter_fsync`, `.../sys_exit_fsync`
    - (optional but recommended) `*_pread64`, `*_pwrite64`, `*_readv`, `*_writev`, `*_preadv`, `*_pwritev`, `*_preadv2`, `*_pwritev2`, `*_fdatasync`
  - On sys_enter:
    - Allocate `op_id`
    - Store in `tid_to_op`
    - Emit `SYS_ENTER` event with `{op_id,type,pid/tid,fd,ts_enter}`
  - On sys_exit:
    - Lookup `tid_to_op`, emit `SYS_EXIT` with `{op_id,ret,ts_exit}`
    - Keep entry until userspace confirms close OR evict on timeout (userspace-driven preferred)

  **Must NOT do**: Must not compute inode identity at syscall layer; only fd is reliable here.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: syscall TP ABI and variants
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 7-21 | Blocked By: 5

  **References**:
  - Repo: `docs/syscall-definition.md` — syscall → __x64_sys_* mapping list
  - Repo: `docs/io-correlation-design.md` — op_id rules
  - External: https://android.googlesource.com/kernel/common/+/df2c1f38939aa/arch/x86/entry/syscalls/syscall_64.tbl

  **Acceptance Criteria**:
  - [ ] Running `sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl` while performing `dd ... conv=fsync` yields at least one `SYS_ENTER` + `SYS_EXIT` pair for WRITE

  **QA Scenarios**:
  ```
  Scenario: sys_enter/sys_exit captured for write
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_w bs=4096 count=16 conv=fsync; wait $pid
    Expected: /tmp/trace.jsonl contains syscall enter+exit events for WRITE
    Evidence: .sisyphus/evidence/task-6-syscall.txt
  ```

  **Commit**: YES | Message: `feat(trace): add syscall op_id lifecycle hooks` | Files: `bpf/io_tracer.bpf.c`

- [ ] 7. VFS/FS Hooks: Iter Entry + generic_* + Ledger Update Events

  **What to do**: Capture VFS/FS boundaries and write ranges for Range Ledger.
  - Attach (prefer fentry/fexit) to:
    - `vfs_iocb_iter_read`
    - `vfs_iocb_iter_write`
    - `generic_file_read_iter`
    - `generic_file_write_iter`
    - `generic_file_fsync`
    - `file_write_and_wait_range` (flush-start marker for FSYNC correlation)
  - For read/write iter hooks:
    - Load current `op_id` from `tid_to_op`
    - Extract inode from `file->f_mapping->host`
    - Extract `(offset_start,len)` from `kiocb->ki_pos` and `iov_iter` count
    - Emit `VFS_ENTER/VFS_EXIT` and `FS_ENTER/FS_EXIT` events with inode identity
  - For write enter:
    - Emit `LEDGER_WRITE` event `{inode_id,offset_start,offset_end,op_id,bytes}`
  - For `file_write_and_wait_range`:
    - If current thread has active FSYNC op_id, emit `FSYNC_FLUSH_START` event (used for snapshot timing)

  **Must NOT do**: Must not attempt kernel-side interval operations; only emit ranges.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: CO-RE struct reads (file, inode, iov_iter)
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 16-21 | Blocked By: 5, 6

  **References**:
  - Repo: `docs/vfs-definition.md` — VFS entry candidates
  - Repo: `docs/fs-definition.md` — FS entry candidates and ops slots
  - Repo: `docs/io-correlation-design.md` — offset-range and ledger update rules

  **Acceptance Criteria**:
  - [ ] For a write workload, JSONL contains at least one `LEDGER_WRITE` record with non-zero range and inode fields

  **QA Scenarios**:
  ```
  Scenario: Ledger write events emitted
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_w2 bs=4096 count=16 conv=fsync; wait $pid; jq -e 'select(.type=="event" and .event=="LEDGER_WRITE")' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-7-ledger-write.txt
  ```

  **Commit**: YES | Message: `feat(trace): add VFS/FS hooks and ledger range events` | Files: `bpf/io_tracer.bpf.c`, `include/io_tracer_abi.h`

- [ ] 8. Writeback→Bio Boundary: Track iomap ioend Submissions

  **What to do**: Capture the writeback boundary where inode+offset range is still available when a bio is submitted.
  - Attach fentry/fexit to `iomap_submit_ioend(struct iomap_writepage_ctx *, struct iomap_ioend *, int)`.
  - On entry (before submit):
    - Read `ioend->io_inode`, `ioend->io_offset`, `ioend->io_size`, `ioend->io_bio`.
    - Build `inode_id` (s_dev, i_ino, i_generation when available; plus epoch/generation fields).
    - Insert into `tracked_bios[bio_ptr] = {inode_id, off_start, len, ts_wb_submit}`.
    - Emit `WB_IOEND_SUBMIT` event with the same payload.

  **Must NOT do**: Must not assume `submit_bio` can recover inode identity for filesystem writeback; iomap boundary is authoritative.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: writeback internals + CO-RE safety
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 9-21 | Blocked By: 5

  **References**:
  - Repo: `docs/io-correlation-design.md` — folio->mapping→inode + submit_bio tagging strategy
  - External: https://codebrowser.dev/linux/linux/fs/iomap/buffered-io.c.html — `iomap_submit_ioend` and `ioend->io_inode/io_offset/io_size/io_bio`

  **Acceptance Criteria**:
  - [ ] A buffered write workload yields at least one `WB_IOEND_SUBMIT` event with a non-null `bio_ptr` and non-zero `len`

  **QA Scenarios**:
  ```
  Scenario: ioend submission events observed
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 3 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_wb bs=4096 count=256; sync; wait $pid; jq -e 'select(.type=="event" and .event=="WB_IOEND_SUBMIT")' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-8-wb-ioend.txt
  ```

  **Commit**: YES | Message: `feat(trace): track writeback ioend->bio submissions` | Files: `bpf/io_tracer.bpf.c`

- [ ] 9. Block Layer Hooks: submit_bio + bio_endio (Filtered)

  **What to do**: Capture block-layer entry and bio completion while filtering to tracked filesystem writeback bios.
  - Attach fentry to `submit_bio(struct bio *bio)` and emit `BIO_SUBMIT` only if `bio_ptr` exists in `tracked_bios`.
  - Attach fentry to `bio_endio(struct bio *bio)` and emit `BIO_ENDIO` only if `bio_ptr` exists in `tracked_bios`.
  - Include in events:
    - `bio_ptr`, `bi_opf`, `bi_iter.bi_sector`, `bi_iter.bi_size`, `bi_status`
    - context classification (from irq_state)

  **Must NOT do**: Do not emit for all bios; tracked-only is mandatory to avoid event storms.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: block core internals and CO-RE reads
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 16-21 | Blocked By: 5, 8

  **References**:
  - Repo: `docs/block-definition.md` — submit_bio boundary definition
  - Repo: `docs/completion-irq-strategy.md` — bio_endio is request-internal stage
  - External: https://blogs.oracle.com/linux/writing-a-block-io-filter-using-libbpf-and-ebpf — bio field access patterns

  **Acceptance Criteria**:
  - [ ] For a buffered write workload, output includes `BIO_SUBMIT` and later `BIO_ENDIO` for the same `bio_ptr`

  **QA Scenarios**:
  ```
  Scenario: Submit and endio events pair
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 4 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_blk bs=4096 count=1024; sync; wait $pid; jq -e 'select(.type=="event" and (.event=="BIO_SUBMIT" or .event=="BIO_ENDIO"))' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0; at least one BIO_SUBMIT and one BIO_ENDIO exist
    Evidence: .sisyphus/evidence/task-9-bio.txt
  ```

  **Commit**: YES | Message: `feat(trace): add filtered submit_bio and bio_endio hooks` | Files: `bpf/io_tracer.bpf.c`

- [ ] 10. Block MQ Correlation: bio→request and request completion

  **What to do**: Link bios to requests and record request completion timing.
  - Attach fentry to `blk_mq_bio_to_request(struct request *rq, struct bio *bio, ...)`.
    - If `bio_ptr` exists in `tracked_bios`, store `tracked_rqs[rq_ptr] = {bio_ptr, ts_issue=now}` and emit `RQ_FROM_BIO`.
  - Attach fentry to `blk_mq_end_request(struct request *rq, blk_status_t error)`.
    - If `rq_ptr` exists in `tracked_rqs`, set `ts_complete`, emit `RQ_END` with error and context.

  **Must NOT do**: Do not attempt to infer bio→rq linkage by sector/time; use `blk_mq_bio_to_request` as the authoritative join.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: blk-mq is subtle (merge, remote completion)
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 16-21 | Blocked By: 5, 8

  **References**:
  - Repo: `docs/io-correlation-design.md` — rq→op_id mapping requirement
  - Repo: `docs/completion-irq-strategy.md` — request completion path
  - External: https://codebrowser.dev/linux/linux/block/blk-mq.c.html — `blk_mq_submit_bio` and `blk_mq_bio_to_request`

  **Acceptance Criteria**:
  - [ ] For at least one tracked I/O, there is a `RQ_FROM_BIO` event followed by a `RQ_END` event for the same `rq_ptr`

  **QA Scenarios**:
  ```
  Scenario: Request lifecycle events
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 4 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_rq bs=4096 count=1024; sync; wait $pid; jq -e 'select(.type=="event" and (.event=="RQ_FROM_BIO" or .event=="RQ_END"))' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-10-rq.txt
  ```

  **Commit**: YES | Message: `feat(trace): correlate bios to requests and capture rq completion` | Files: `bpf/io_tracer.bpf.c`

- [ ] 11. NVMe Hooks: Driver Entry/Completion + Optional Notify Boundaries

  **What to do**: Observe NVMe driver boundaries for tracked requests.
  - Mandatory (per correlation design):
    - fentry `nvme_queue_rq` → emit `NVME_QUEUE_RQ` for `rq_ptr` in `tracked_rqs`
    - fentry `nvme_complete_rq` → emit `NVME_COMPLETE_RQ` for `rq_ptr` in `tracked_rqs`
  - Optional (best-effort; attach if symbol exists):
    - fentry `nvme_submit_cmd` → emit `NVME_SUBMIT_CMD`
    - fentry `nvme_write_sq_db` → emit `NVME_WRITE_SQ_DB`
    - fentry `nvme_poll` → emit `NVME_POLL`
    - fentry `nvme_irq` / `nvme_process_cq` → emit `NVME_IRQ` / `NVME_PROCESS_CQ`
  - If a function symbol is missing for fentry, fall back to NVMe tracepoints when available (document in code via capability flags).

  **Must NOT do**: Do not assume notify boundaries are always available; treat them as optional enrichment.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: driver symbols vary by config
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 16-21 | Blocked By: 10

  **References**:
  - Repo: `docs/nvme-driver-definition.md` — required boundary semantics
  - External: https://codebrowser.dev/linux/linux/drivers/nvme/host/pci.c.html
  - External: https://codebrowser.dev/linux/linux/drivers/nvme/host/core.c.html

  **Acceptance Criteria**:
  - [ ] For at least one tracked request, output contains `NVME_QUEUE_RQ` and `NVME_COMPLETE_RQ`

  **QA Scenarios**:
  ```
  Scenario: NVMe queue and completion events
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 4 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_nvme bs=4096 count=1024; sync; wait $pid; jq -e 'select(.type=="event" and (.event=="NVME_QUEUE_RQ" or .event=="NVME_COMPLETE_RQ"))' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-11-nvme.txt
  ```

  **Commit**: YES | Message: `feat(trace): add NVMe driver boundary hooks` | Files: `bpf/io_tracer.bpf.c`, `include/io_tracer_abi.h`

- [ ] 12. IRQ/SoftIRQ Windows + Completion Context Classification

  **What to do**: Implement hard/soft IRQ window tracking and attach context to completion-related events.
  - Attach to tracepoints:
    - `tracepoint/irq/irq_handler_entry` / `.../irq_handler_exit`
    - `tracepoint/irq/softirq_entry` / `.../softirq_exit`
  - Maintain `irq_state[cpu] = {hardirq_depth, softirq_depth}`.
  - For completion events (`BIO_ENDIO`, `RQ_END`, `NVME_COMPLETE_RQ`):
    - Compute context: hardirq if hardirq_depth>0; else softirq if softirq_depth>0; else process
    - Also record `in_hardirq()` and `in_serving_softirq()` when accessible

  **Must NOT do**: Must not classify solely from `in_hardirq()` without windows; record both for debugging.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: correctness across remote completion paths
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 2 | Blocks: 16-21 | Blocked By: 5

  **References**:
  - Repo: `docs/completion-irq-strategy.md` — context classification rules
  - External: https://codebrowser.dev/linux/linux/kernel/irq/handle.c.html
  - External: https://codebrowser.dev/linux/linux/kernel/softirq.c.html

  **Acceptance Criteria**:
  - [ ] At least one completion event includes a non-null `completion.context` field (HARDIRQ/SOFTIRQ/PROCESS)

  **QA Scenarios**:
  ```
  Scenario: Completion context present
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 3 --output /tmp/trace.jsonl; jq -e 'select(.type=="event") | select(.completion!=null) | .completion.context' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-12-ctx.txt
  ```

  **Commit**: YES | Message: `feat(trace): add IRQ windows and completion context classification` | Files: `bpf/io_tracer.bpf.c`

- [ ] 13. Userspace Correlator Core: State Machines + Pointer Reuse Safety

  **What to do**: Implement the core in-memory correlation model in userspace.
  - Add modules:
    - `user/corr_op.c` (op_id lifecycle, stage timestamps, fd→inode association, confidence)
    - `user/corr_ptr.c` (pointer-keyed maps with `(ptr, first_ts)` guard)
  - Enforce state machines:
    - `bio`: CREATED(at WB_IOEND_SUBMIT) → SUBMITTED(BIO_SUBMIT) → ENDED(BIO_ENDIO)
    - `rq`: CREATED(RQ_FROM_BIO) → QUEUED(NVME_QUEUE_RQ) → COMPLETED(NVME_COMPLETE_RQ/RQ_END)
  - On reuse collision (same ptr seen in CREATED while not finished):
    - mark old object `correlation_lost: PTR_REUSE`
    - increment loss counter and emit health record

  **Must NOT do**: Must not assume strict ordering across CPUs; allow out-of-order arrival and reconcile by timestamps.

  **Recommended Agent Profile**:
  - Category: `deep` — Reason: cross-layer join + async ordering
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 3 | Blocks: 15-18 | Blocked By: 3, 4, 6-12

  **References**:
  - Repo: `docs/io-correlation-design.md` — pointer mapping tables + correlation_lost
  - Repo: `docs/completion-irq-strategy.md` — remote completion reality
  - Oracle guidance embedded in `docs/io-correlation-design.md` updates (inode identity + epoch/version)

  **Acceptance Criteria**:
  - [ ] `sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl` produces at least one `health` record with counters

  **QA Scenarios**:
  ```
  Scenario: Health record always present
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl; jq -e 'select(.type=="health")' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-13-health.txt
  ```

  **Commit**: YES | Message: `feat(corr): add pointer-safe correlator state machines` | Files: `user/corr_*.c`, `user/corr_*.h`

- [ ] 14. Range Ledger Integration: Write→Writeback→Bio Attribution

  **What to do**: Connect buffered writes to writeback bios using Range Ledger.
  - On `LEDGER_WRITE` events:
    - Update Range Ledger with `{inode_id, range, op_id, bytes}`.
  - On `WB_IOEND_SUBMIT` events:
    - Lookup contributors for `{inode_id, [off_start, off_start+len)}`.
    - Store `bio_ptr -> contributor_set` (policy-dependent representation).
  - On `RQ_FROM_BIO`:
    - Copy contributor_set from `bio_ptr` to `rq_ptr`.

  **Must NOT do**: Must not silently attribute without a ledger hit; if lookup misses, emit `confidence=UNATTRIBUTED` with reason.

  **Recommended Agent Profile**:
  - Category: `ultrabrain` — Reason: many-to-many policy + correctness
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 3 | Blocks: 15-18 | Blocked By: 4, 7, 8, 10

  **References**:
  - Repo: `docs/io-correlation-design.md` — writeback→device rules, many-to-many policies
  - Repo: `docs/block-definition.md` — block entry semantics

  **Acceptance Criteria**:
  - [ ] At least one completed op record contains a non-empty `contributors` field (multi policy) OR `attribution` field (other policies)

  **QA Scenarios**:
  ```
  Scenario: Attribution present
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 3 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_attr bs=4096 count=512; sync; wait $pid; jq -e 'select(.type=="op") | (.contributors!=null or .attribution!=null)' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-14-attr.txt
  ```

  **Commit**: YES | Message: `feat(corr): connect ledger to writeback bios and requests` | Files: `user/*`

- [ ] 15. Latency Model: Per-op Stage Durations + End-to-End Metrics

  **What to do**: Implement the latency calculations specified by the design.
  - For each op_id, compute and emit an `op` JSON record containing:
    - syscall: enter/exit + `duration_ns`
    - vfs/fs: enter/exit + `duration_ns`
    - writeback queue: `t_submit_bio - t_ledger_update` (if applicable)
    - block: `t_rq_end - t_bio_submit` and/or `t_rq_end - t_rq_from_bio`
    - nvme device: `t_nvme_complete_rq - t_nvme_queue_rq`
    - completion: context classification and final completion ts
  - Explicitly label overlapping spans; do not claim strict additivity.

  **Must NOT do**: Must not output negative durations; clamp with error flags and emit health counter.

  **Recommended Agent Profile**:
  - Category: `deep` — Reason: multi-stage timestamps and overlaps
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 3 | Blocks: 16-18 | Blocked By: 13, 14

  **References**:
  - Repo: `docs/io-correlation-design.md` — latency formulas

  **Acceptance Criteria**:
  - [ ] `jq -e -s 'all(.[]; (.type!="op") or ((.stages|to_entries|map(.value.duration_ns)|all(.>=0))))' /tmp/trace.jsonl` exits 0 after a run

  **QA Scenarios**:
  ```
  Scenario: No negative durations
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 3 --output /tmp/trace.jsonl & pid=$!; dd if=/dev/zero of=/tmp/iotrace_lat bs=4096 count=256 conv=fsync; wait $pid; jq -e -s 'all(.[]; (.type!="op") or ((.stages|to_entries|map(.value.duration_ns)|all(.>=0))))' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-15-lat.txt
  ```

  **Commit**: YES | Message: `feat(metrics): compute per-op latency breakdowns` | Files: `user/*`

- [ ] 16. FSYNC Correlation: Snapshot-at-Flush-Start Matching

  **What to do**: Implement fsync-specific correlation.
  - On FSYNC syscall entry: create `fsync_op_id` (already via task 6) and mark op type FSYNC.
  - On `FSYNC_FLUSH_START` (from `file_write_and_wait_range`):
    - Snapshot current ledger intervals for the inode into an fsync-snapshot structure keyed by `fsync_op_id`.
  - On subsequent `WB_IOEND_SUBMIT` within the fsync window:
    - Match writeback I/O to snapshot ranges (version/epoch must match).
  - Emit fsync op record including:
    - syscall latency
    - fsync data completion time (max of matched writeback completions)
    - mismatch counters (stale version / missing range)

  **Must NOT do**: Must not treat syscall entry as snapshot time; snapshot at flush-start is required.

  **Recommended Agent Profile**:
  - Category: `ultrabrain` — Reason: snapshot semantics + version/epoch matching
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 3 | Blocks: 18 | Blocked By: 7, 14, 15

  **References**:
  - Repo: `docs/io-correlation-design.md` — FSYNC correlation design
  - Repo: `docs/vfs-definition.md` — fsync entry candidates

  **Acceptance Criteria**:
  - [ ] Running workload with explicit fsync produces an FSYNC op record with `flush_start_ts` and at least one matched writeback I/O OR explicit `UNATTRIBUTED` with reason

  **QA Scenarios**:
  ```
  Scenario: FSYNC snapshot and matching
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 4 --output /tmp/trace.jsonl & pid=$!; ./user/fsync_helper /tmp/iotrace_fsync; wait $pid; jq -e 'select(.type=="op" and .op_type=="FSYNC") | .flush_start_ts' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-16-fsync.txt
  ```

  **Commit**: YES | Message: `feat(corr): fsync snapshot matching at flush-start` | Files: `user/*`, `include/io_tracer_abi.h`

- [ ] 17. Health Telemetry + Drop/Loss Reporting (First-class)

  **What to do**: Make event loss and correlation gaps observable.
  - Kernel side:
    - ringbuf reserve failure counter
    - map update failure counters
  - Userspace:
    - count dropped records
    - count orphan events
    - count ptr reuse collisions
  - Emit `health` JSON records periodically and at exit.

  **Must NOT do**: Must not hide drops; health must always be present.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: end-to-end observability
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 4 | Blocks: 18 | Blocked By: 3, 5-15

  **References**:
  - Repo: `docs/io-correlation-design.md` — correlation_lost requirement
  - Metis guardrails (acceptance criteria)

  **Acceptance Criteria**:
  - [ ] `jq -e 'select(.type=="health") | (.dropped_events|type=="number") and (.correlation_lost|type=="number")' /tmp/trace.jsonl` exits 0

  **QA Scenarios**:
  ```
  Scenario: Health fields present
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 2 --output /tmp/trace.jsonl; jq -e 'select(.type=="health") | (.dropped_events|type=="number") and (.correlation_lost|type=="number")' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-17-health-fields.txt
  ```

  **Commit**: YES | Message: `feat(obs): add health telemetry and loss counters` | Files: `bpf/io_tracer.bpf.c`, `user/*`

- [ ] 18. End-to-End Selftest Mode + Deterministic Assertions

  **What to do**: Provide a single command that validates the tracer.
  - Add `user/fsync_helper` (create file, write, fsync) if not already.
  - Implement `--selftest` in `user/io-tracer`:
    - run tracer capture in-process for a short duration
    - run workloads:
      - write: `dd ... conv=fsync`
      - read: `dd if=<file> of=/dev/null`
      - fsync: `fsync_helper`
    - run jq assertions and exit non-zero on failure

  **Must NOT do**: Must not depend on fio; base utilities only.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: stable CI-like validation
  - Skills: []

  **Parallelization**: Can Parallel: NO | Wave 4 | Blocks: F-wave | Blocked By: 1-17

  **References**:
  - Repo: `docs/io-correlation-design.md` — required output fields
  - Repo: `docs/completion-irq-strategy.md` — context fields must exist

  **Acceptance Criteria**:
  - [ ] `sudo ./user/io-tracer --selftest --output /tmp/io-tracer.jsonl` exits 0
  - [ ] The jq assertions in the DoD section pass on `/tmp/io-tracer.jsonl`

  **QA Scenarios**:
  ```
  Scenario: Selftest passes
    Tool: Bash
    Steps: sudo ./user/io-tracer --selftest --output /tmp/io-tracer.jsonl
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-18-selftest.txt

  Scenario: Selftest produces correlated ops
    Tool: Bash
    Steps: test -s /tmp/io-tracer.jsonl && jq -e 'select(.type=="op") | .op_id and .completion.context' /tmp/io-tracer.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-18-schema.txt
  ```

  **Commit**: YES | Message: `test(selftest): add end-to-end workload and assertions` | Files: `user/*`

- [ ] 19. Capability Detection + Graceful Degradation

  **What to do**: Make optional hooks safe across configs.
  - At startup, detect availability/attach success for each hook group and emit a `capabilities` JSON record.
  - For missing optional NVMe hooks (submit_cmd/write_sq_db/poll/irq/process_cq), do not fail run; omit those stages.
  - For missing filesystem writeback hook (`iomap_submit_ioend`), run in "uncorrelated" mode and explicitly label `confidence=UNATTRIBUTED` for writeback/device stages.

  **Must NOT do**: Must not silently drop stages; always emit capability flags.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: runtime robustness
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 4 | Blocks: 22 | Blocked By: 3, 5-12

  **References**:
  - Repo: `docs/nvme-driver-definition.md` — optional enrichment hooks

  **Acceptance Criteria**:
  - [ ] Every run emits exactly one `capabilities` record with booleans for each hook group

  **QA Scenarios**:
  ```
  Scenario: Capabilities record present
    Tool: Bash
    Steps: sudo ./user/io-tracer --duration 1 --output /tmp/trace.jsonl; jq -e 'select(.type=="capabilities")' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0
    Evidence: .sisyphus/evidence/task-19-caps.txt
  ```

  **Commit**: YES | Message: `feat(user): add capability detection and stage gating` | Files: `user/*`

- [ ] 20. Performance Guardrails: Filtering + Sampling + Ringbuf Sizing

  **What to do**: Prevent overload and make behavior predictable.
  - Add CLI filters:
    - `--pid <pid>` (track a single process)
    - `--inode <dev:ino>` (track a single file)
    - `--min-len <bytes>` (drop tiny I/O)
  - Add sampling knobs:
    - `--sample-rate <N>` (keep 1/N ops)
  - Add ringbuf sizing option and report drops.

  **Must NOT do**: Must not change default behavior silently; default should be full capture with tracked-only block events.

  **Recommended Agent Profile**:
  - Category: `unspecified-high` — Reason: performance vs fidelity tradeoffs
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 4 | Blocks: 18 | Blocked By: 3, 5-15

  **References**:
  - Repo: `docs/block-definition.md` — tracked-only principle for block layer

  **Acceptance Criteria**:
  - [ ] Running with `--pid $$` filters output to that pid (verify by jq)

  **QA Scenarios**:
  ```
  Scenario: PID filter reduces scope
    Tool: Bash
    Steps: pid=$$; sudo ./user/io-tracer --duration 1 --pid $pid --output /tmp/trace.jsonl; jq -e --argjson pid $pid 'select(.type=="event") | .pid == $pid' /tmp/trace.jsonl >/dev/null
    Expected: jq exits 0 (or file empty but health/capabilities present)
    Evidence: .sisyphus/evidence/task-20-filter.txt
  ```

  **Commit**: YES | Message: `feat(user): add filters and sampling guardrails` | Files: `user/*`, `include/io_tracer_abi.h`

- [ ] 21. Project Docs + Usage Contract

  **What to do**: Document how to run and interpret outputs.
  - Add `docs/README.md` linking:
    - `docs/io-correlation-design.md`
    - boundary docs (`docs/vfs-definition.md`, `docs/fs-definition.md`, `docs/block-definition.md`, `docs/nvme-driver-definition.md`)
    - `docs/completion-irq-strategy.md`
  - Update root `README.md` with:
    - build prerequisites
    - quickstart commands (`make build`, `sudo ./user/io-tracer --selftest ...`)
    - output schema brief (record types: capabilities/health/event/op)

  **Must NOT do**: Must not diverge from actual CLI; docs must be generated from real flags.

  **Recommended Agent Profile**:
  - Category: `writing` — Reason: technical doc polish
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 4 | Blocks: none | Blocked By: 18

  **References**:
  - Repo: `docs/io-correlation-design.md`
  - Repo: `README.md`

  **Acceptance Criteria**:
  - [ ] `README.md` contains runnable quickstart commands matching actual CLI flags

  **QA Scenarios**:
  ```
  Scenario: README quickstart works
    Tool: Bash
    Steps: make clean && make build && sudo ./user/io-tracer --duration 1 --output /tmp/trace.jsonl
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-21-readme.txt
  ```

  **Commit**: YES | Message: `docs: add quickstart and docs index` | Files: `README.md`, `docs/README.md`

- [ ] 22. CI (Optional): Build + Unit + Selftest Gate

  **What to do**: Add a CI workflow (if GitHub is used) that validates build and unit tests.
  - Add `.github/workflows/ci.yml`:
    - Build userspace + BPF (no root)
    - Run `make test`
    - (Optional) selftest behind privileged runner flag, otherwise skip with clear message

  **Must NOT do**: Must not require privileged CI by default; selftest can be gated.

  **Recommended Agent Profile**:
  - Category: `writing` — Reason: CI YAML + docs
  - Skills: []

  **Parallelization**: Can Parallel: YES | Wave 4 | Blocks: none | Blocked By: 1, 4

  **References**:
  - External: https://github.com/libbpf/libbpf-bootstrap — CI patterns

  **Acceptance Criteria**:
  - [ ] `gh workflow run` is not required; YAML validates and `make test` can run locally

  **QA Scenarios**:
  ```
  Scenario: Local CI-equivalent checks
    Tool: Bash
    Steps: make clean && make build && make test
    Expected: exit 0
    Evidence: .sisyphus/evidence/task-22-local-ci.txt
  ```

  **Commit**: YES | Message: `ci: add build and unit test workflow` | Files: `.github/workflows/ci.yml`

## Final Verification Wave (4 parallel agents, ALL must APPROVE)
- [ ] F1. Plan Compliance Audit — oracle
- [ ] F2. Code Quality Review — unspecified-high
- [ ] F3. Real Manual QA (agent-executed) — unspecified-high (+ playwright not needed)
- [ ] F4. Scope Fidelity Check — deep

## Commit Strategy
- Prefer small, reviewable commits aligned to waves.
- Commit message style: `feat(trace): ...`, `feat(corr): ...`, `test(selftest): ...`, `docs: ...`, `build: ...`

## Success Criteria
- The tracer runs on Linux 6.6.1 with NVMe and produces correlated per-op spans for read/write/fsync.
- Buffered writeback is attributed using Range Ledger with explicit confidence and loss telemetry.
- Completion context is classified and included in output.
