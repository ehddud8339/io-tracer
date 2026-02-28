# I/O Correlation & Latency Design (Linux 6.6)

기준 버전: Linux v6.6.1

## 1. 설계 목표

- read / write / fsync 요청을 `op_id` 단위로 추적한다.
- buffered write + writeback 경로를 하나의 상관관계 모델로 연결한다.
- syscall, VFS/FS, writeback, block, NVMe, completion 컨텍스트로 레이어별 latency를 분해한다.
- completion 컨텍스트를 Hard IRQ / Soft IRQ / Process(또는 Workqueue)로 구분한다.

## 2. 전역 개념 정의

### 2.1 op_id 정의

- `op_id`는 syscall entry에서 생성되는 단조 증가 64-bit ID다.
- 생성 위치: `__x64_sys_read` / `__x64_sys_write` / `__x64_sys_fsync` 계열 entry.
- 최소 포함 필드:
  - `op_id`
  - `type` (`READ` / `WRITE` / `FSYNC`)
  - `pid` / `tid`
  - `fd`
  - inode 식별자
  - `ts_enter` / `ts_exit`
- 기본 규칙:
  - syscall entry에서 `tid -> op_id` 매핑 생성
  - syscall exit에서 `ts_exit` 기록 후 상태를 완료 상태로 전환

### 2.2 파일 식별자 정의

- 기본 식별자는 아래 조합을 사용한다.
  - `inode*` 포인터
  - `i_ino`
  - `sb->s_dev`
  - `i_generation` (가능한 경우)
- 포인터 재사용 대비 검증 전략:
  - `inode*` 단독 키를 금지하고 `(inode*, i_ino, s_dev, i_generation)` 검증을 기본으로 사용한다.
  - `i_generation`을 얻기 어려운 경로에서는 `(inode*, i_ino, s_dev)` + inode epoch으로 보강한다.
  - ledger 값에 `generation`을 두고, truncate/hole-punch/inode 재활용 의심 이벤트 시 세대를 증가시킨다.
  - 조회 시 3중 키 불일치 또는 세대 불일치가 발생하면 매칭을 폐기하고 orphan 이벤트로 분류한다.

### 2.3 Range Ledger 정의 (핵심 설계)

- 키: `(inode*, offset_start, offset_end)`
- 값:
  - `writer_op_id`
  - `timestamp`
  - `generation`
  - `contributors` (op_id -> bytes)
- 설계 원칙:
  - 커널(eBPF)은 이벤트를 스트리밍만 수행한다.
  - 유저스페이스가 interval tree 기반 Range Ledger를 유지/갱신한다.
  - 커널에서 복잡한 구간 병합/탐색을 하지 않아 verifier 부담과 map contention을 낮춘다.

## 3. 레이어별 계측 이벤트 정의

### 3.1 Syscall Layer

- 계측 대상: `__x64_sys_read` / `__x64_sys_write` / `__x64_sys_fsync` 계열
- 처리:
  - entry: `op_id` 생성, syscall 컨텍스트 메타데이터 기록
  - exit: syscall 종료 시각 및 결과 코드 기록

### 3.2 VFS / FS Layer

- 계측 대상:
  - `vfs_iocb_iter_read`
  - `vfs_iocb_iter_write`
  - `generic_file_read_iter`
  - `generic_file_write_iter`
  - `generic_file_fsync`
- offset-range 계산:
  - `kiocb->ki_pos`를 시작 오프셋으로 사용
  - `iov_iter` 길이를 바이트 단위 길이로 사용
  - `offset_end = offset_start + len - 1`
  - 부분 I/O 시 실제 처리 바이트 기준으로 보정 이벤트를 추가 기록

### 3.3 Write Path Ledger 업데이트 규칙

- write entry에서 `(inode, offset, len)`을 추출해 range를 계산한다.
- Range Ledger에 `(inode, range) -> writer_op_id`를 기록한다.
- overwrite 처리:
  - 기본 정책: 새 write가 같은 구간을 덮으면 기존 엔트리를 split/replace한다.
  - 겹침 구간은 최신 write 기준으로 우선순위를 부여하되, `contributors`는 정책별(last-writer/proportional/multi)로 유지한다.
- truncate/hole-punch 처리:
  - 해당 inode의 겹치는 구간을 invalidate하거나 `generation++` 후 재기록한다.
  - 이전 세대 엔트리는 조회 대상에서 제외한다.

### 3.4 Writeback → Device 연결 규칙

- folio 기반 offset 복원:
  - `folio->mapping`으로 address_space 확인
  - `mapping->host`로 inode 복원
  - `folio->index << PAGE_SHIFT`로 folio 시작 오프셋 계산
- writeback I/O 이벤트에서 inode+range를 얻어 Range Ledger를 조회한다.
- ledger 조회 결과로 `op_id`(또는 op_id 집합)를 결정한다.
- `submit_bio` 이벤트에 `op_id`를 태깅한다.
- 병합/분할 이전 원인 보존을 위해 `submit_bio`에 단일 `op_id` 대신 `op_id` 집합(또는 압축된 contributor 벡터)을 태깅할 수 있다.
- caveat:
  - bio가 여러 folio/여러 write를 합칠 수 있으므로 단일 op_id가 아닐 수 있다.
  - 이 경우 many-to-many 정책(섹션 6)을 적용한다.

### 3.5 Block Layer

- 계측 대상:
  - `submit_bio`
  - `blk_mq_end_request`
- 목적:
  - writeback이 실제 block I/O로 제출되는 시점과 request 완료 시점을 분리 측정

### 3.6 NVMe Layer

- 계측 대상:
  - `nvme_queue_rq`
  - `nvme_complete_rq`
- 목적:
  - block request가 NVMe 드라이버로 들어가는 시점과 드라이버 완료 반환 시점을 연결

### 3.7 Completion Context

- 계측 대상:
  - `trace_irq_handler_entry` / `trace_irq_handler_exit`
  - `trace_softirq_entry` / `trace_softirq_exit`
  - completion 시점 `in_hardirq()` / `in_serving_softirq()`
- 목적:
  - 동일 완료 이벤트를 IRQ 컨텍스트 관점으로 분류

## 4. FSYNC 상관관계 설계

- `vfs_fsync_range` entry에서 `fsync_op_id`를 생성한다.
- flush 시작 경계에서 해당 inode의 Range Ledger 스냅샷을 생성한다.
- fsync 구간 내 writeback I/O를 스냅샷과 매칭한다.
- 분리 방법:
  - `fsync latency = t_fsync_exit - t_fsync_enter`
  - `device I/O latency = t_nvme_complete_rq - t_submit_bio` (매칭된 writeback I/O 집합)
  - fsync 대기 구간은 "스냅샷에 포함된 dirty range(version 일치)가 실제 completion에 도달할 때까지"로 계산한다.

## 5. 레이턴시 계산 모델

각 `op_id`별 필수 타임스탬프:

- syscall enter/exit
- vfs/fs enter/exit
- ledger update 시각
- submit_bio 시각
- nvme_queue_rq 시각
- nvme_complete_rq 시각
- bio_endio 시각
- completion context (hardirq/softirq/process 분류 시각)

레이어별 계산식:

- syscall latency
  - `L_syscall = t_sys_exit - t_sys_enter`
- fs latency (VFS/FS 처리)
  - `L_fs = t_fs_exit - t_fs_enter`
- writeback latency
  - `L_wb = t_submit_bio - t_ledger_update`
- device latency
  - `L_dev = t_nvme_complete_rq - t_nvme_queue_rq`
- end-to-end latency
  - read/write: `L_e2e = t_bio_endio - t_sys_enter` (또는 policy에 따라 request 완료 시각 사용)
  - fsync: `L_e2e_fsync = t_fsync_exit - t_fsync_enter`

## 6. Many-to-Many 처리 전략

- 케이스 A: 여러 write -> 하나의 bio
- 케이스 B: 하나의 write -> 여러 bio

정책 옵션:

1. last-writer attribution
- 정의: 겹치는 구간에서 가장 최근 write의 `op_id`에 귀속
- 장점: 구현 단순, 실시간성 우수
- 단점: 이전 write 기여도가 사라져 편향 발생

2. proportional attribution
- 정의: bio range와 겹치는 바이트 비율로 각 `op_id`에 분배
- 장점: 물리 I/O 비용 분배가 비교적 공정
- 단점: 계산 비용 증가, 해석 복잡도 상승

3. multi-attribution
- 정의: 관련 `op_id` 집합 전체를 연결하고 가중치는 별도 저장
- 장점: 정보 손실 최소화, 사후 분석 유연성 높음
- 단점: 저장량 증가, 후처리 필수

선택 기준:

- 온라인/저오버헤드 우선: last-writer
- 정량 분석 정확도 우선: proportional
- 재현성과 포렌식 우선: multi-attribution

## 7. 구현 전략

- 커널(eBPF) 최소 작업:
  - 이벤트 타임스탬프/포인터/핵심 식별자 스트리밍
  - 경량 포인터 매핑 유지
  - 포인터 재사용 충돌 시 `correlation_lost` 플래그 이벤트 발행
  - 복잡한 interval 연산은 수행하지 않음
- 유저스페이스 작업:
  - interval tree 기반 Range Ledger 관리
  - many-to-many 매칭/정책 적용
  - latency 계산 및 컨텍스트 분류 집계

pointer 기반 매핑 테이블:

- `tid -> op_id`
- `iocb -> op_id`
- `bio -> op_id` (또는 op_id set)
- `rq -> op_id` (또는 op_id set)

## 8. 설계 요약

- 포인터 기반 단일 키만으로는 buffered I/O 연결이 불가능하다.
  - write 시점 객체와 writeback 시점 객체가 다르고, 병합/분할로 관계가 변형되기 때문이다.
- offset-range 기반 ledger가 필요한 이유는 "시간차를 두고 비동기적으로 재구성되는 writeback I/O"를 원래 write 요청과 연결하기 위해서다.
- IRQ + request + bio + ledger 조합이 완전한 이유:
  - IRQ는 컨텍스트 원인,
  - request/bio는 완료 단위,
  - ledger는 buffered write와 writeback 사이의 잃어버린 연결고리를 제공한다.

이 네 축을 결합해야 syscall->VFS->FS->writeback->block->NVMe->completion 전 경로의 레이어별 latency 분해가 일관되게 가능하다.
