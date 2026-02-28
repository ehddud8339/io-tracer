# NVMe Driver ↔ Device 경계 정의 (Linux 6.6 기준)

기준 버전: v6.6.1

## 서론

NVMe driver layer entry는 **blk-mq에서 전달된 request가 NVMe 드라이버로 진입하는 지점**을 의미한다.
실무적으로는 `blk_mq_ops.queue_rq`에 연결된 `nvme_queue_rq`가 Driver entry의 1차 기준이 된다.
read/write/fsync는 block layer에서 request로 수렴한 뒤 동일한 `queue_rq` 제출 경로를 통과하며, NVMe SQ 제출 절차로 이어진다.
fsync 의도는 block layer request에서 `REQ_OP_FLUSH` 형태로 표현되고, NVMe 경계에서는 flush 명령/완료 지연으로 관측된다.

## I/O 흐름 개요

```text
Driver (submission)
     |
     v
Device (NVMe Controller)
     |
     v
Driver (completion)
```

## 1️⃣ Driver → Device (Submission 경계)

### `nvme_queue_rq`
- 호출 계층: blk-mq `queue_rq` 콜백에서 NVMe PCI 드라이버 진입
- 역할: blk request를 NVMe command/SQE로 준비하고 SQ 제출 흐름으로 전달
- latency 계측 시 의미: Driver entry의 대표 기준점이며 read/write/fsync(REQ_OP_FLUSH 포함) 공통 제출 지연 측정에 유리

### `nvme_poll` (polled I/O)
- 호출 계층: blk-mq poll 경로(`.poll`)에서 드라이버 진입
- 역할: IRQ 대신 폴링으로 CQ를 확인해 완료 이벤트를 회수
- latency 계측 시 의미: submission 이후 completion 회수 지연을 polling 기준으로 분리 측정 가능

### `nvme_submit_cmd`
- 호출 계층: NVMe 드라이버 내부 command 제출 계층(큐 엔트리 적재 단계)
- 역할: NVMe command를 SQ 엔트리로 반영해 디바이스가 소비 가능한 형태로 준비
- latency 계측 시 의미: request 준비 단계와 실제 MMIO doorbell notify 단계를 분리해 관찰할 수 있는 중간 경계

### `nvme_write_sq_db`
- 호출 계층: 드라이버 SQ doorbell MMIO write 경계
- 역할: SQ tail 업데이트를 디바이스에 통지하여 실제 처리 시작을 유도
- latency 계측 시 의미: "Device notify 경계"를 직접 포착하므로 Driver→Device 구간 측정의 핵심 타점

## 2️⃣ Device → Driver (Completion 경계)

### `nvme_irq`
- 인터럽트/폴링 진입 여부: 인터럽트 기반 진입점(IRQ handler)
- completion 처리 단계: IRQ 컨텍스트에서 CQ 처리 루틴으로 진입
- block layer로 반환되는 지점: CQ 처리 후 완료된 request가 block completion 경로로 전달되는 시작점

### `nvme_process_cq`
- 인터럽트/폴링 진입 여부: IRQ/폴링 공통 CQ 처리 단계에서 호출되는 completion 처리 루틴
- completion 처리 단계: CQE를 해석하고 완료 대상 request를 식별/정리
- block layer로 반환되는 지점: 개별 request 완료 처리를 통해 block layer 완료 호출로 연결되는 핵심 중간 단계

### `nvme_complete_rq`
- 인터럽트/폴링 진입 여부: IRQ/폴링 모두에서 최종 request 완료 처리 단계로 도달
- completion 처리 단계: 드라이버 단에서 request 완료 상태를 정리하고 완료 콜백으로 연결
- block layer로 반환되는 지점: block layer 완료 API로 반환되는 직접 경계로서 Device→Driver 구간 종점에 해당

## 요약 표

| 흐름 구간 | 함수 | 역할 | 계측 의미 |
|---|---|---|---|
| Driver → Device | nvme_queue_rq | blk-mq request를 NVMe 제출 경로로 진입시킴 | Driver entry 1차 기준, 공통 제출 지연 측정 |
| Driver → Device | nvme_poll | polled I/O에서 CQ를 폴링해 완료 회수 | IRQ와 분리된 polling 지연 분석 |
| Driver → Device | nvme_submit_cmd | command를 SQ 엔트리로 적재/제출 준비 | 준비 단계와 notify 단계 분리 관찰 |
| Driver → Device | nvme_write_sq_db | SQ doorbell MMIO write로 디바이스 통지 | Device notify 경계의 직접 계측 포인트 |
| Device → Driver | nvme_irq | IRQ 기반 completion 진입 | 인터럽트 기반 완료 지연 구간 시작점 |
| Device → Driver | nvme_process_cq | CQE 처리 및 완료 request 식별 | completion 중간 처리 오버헤드 분해 |
| Device → Driver | nvme_complete_rq | request 완료를 block layer로 반환 | Device→Driver 구간 종점/반환 경계 |

## 설계 메모

- `nvme_queue_rq`를 Driver entry의 1차 기준으로 보는 이유는 blk-mq에서 NVMe 드라이버로 넘어오는 공식 제출 경계이기 때문이다.
- `nvme_submit_cmd`/`nvme_write_sq_db`는 command 준비와 doorbell notify를 구분할 수 있어, 특히 `nvme_write_sq_db`는 "Device notify 경계"로 해석하는 것이 유효하다.
- completion 계측은 irq 기반(`nvme_irq`)과 polling 기반(`nvme_poll`)의 스케줄링/회수 특성이 달라 별도 지표로 분리해야 한다.
- Driver → Device latency(제출~notify)와 Device → Driver latency(완료 신호~완료 반환)는 함수 경계를 분리 계측하면 독립적으로 추정 가능하다.

## 참고문헌

[1] Linux v6.6.1 drivers/nvme/host/pci.c. Codebrowser. https://codebrowser.dev/linux/linux/drivers/nvme/host/pci.c.html
[2] Linux Kernel Documentation — NVMe Driver. https://docs.kernel.org/nvme/index.html
