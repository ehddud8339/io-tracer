# Completion & IRQ 추적 전략 (Linux 6.6 기준)

기준 버전: v6.6.1

## 서론

NVMe 기반 블록 I/O의 completion 레이턴시를 정확히 분해하려면 IRQ 계층과 I/O 완료 계층을 동시에 추적해야 한다.
Hard IRQ/Soft IRQ 이벤트는 "어떤 컨텍스트에서 완료 처리가 실행되었는가"를 보여주는 원인(컨텍스트) 신호이고,
`nvme_complete_rq`/`blk_mq_end_request`/`bio_endio`는 실제 I/O 단위가 어디까지 완료되었는지를 보여주는 결과(완료) 신호다.
단일 포인트만 추적하면 컨텍스트와 완료 단위를 혼동하기 쉽고, Driver→Device와 Device→Driver 지연을 분리하기 어렵다.
따라서 IRQ(원인) + request 완료 + bio 완료를 함께 잡는 다층 추적이 필요하다.

## 1️⃣ 추적 대상 정의

### IRQ 레벨

#### `trace_irq_handler_entry`
- 계층 위치: Hard IRQ 핸들러 진입 지점(`kernel/irq/handle.c` 경로)
- 의미: 특정 IRQ 핸들러 실행 시작
- 레이턴시 역할: completion 이벤트가 Hard IRQ 컨텍스트에서 시작되었는지 판별하는 기준점

#### `trace_irq_handler_exit`
- 계층 위치: Hard IRQ 핸들러 종료 지점
- 의미: IRQ 핸들러 실행 종료
- 레이턴시 역할: IRQ 핸들러 구간 길이와 completion 처리 구간의 중첩 여부 분석

#### `trace_softirq_entry`
- 계층 위치: SoftIRQ 실행 진입(`kernel/softirq.c`, `__do_softirq` 루프)
- 의미: softirq 벡터 처리 시작
- 레이턴시 역할: completion이 softirq 컨텍스트로 지연/전달되었는지 분류

#### `trace_softirq_exit`
- 계층 위치: SoftIRQ 실행 종료
- 의미: softirq 벡터 처리 종료
- 레이턴시 역할: softirq 처리 시간 및 completion 소모 구간 계산

### Request 완료 레벨

#### `nvme_complete_rq`
- 계층 위치: NVMe 드라이버 완료 경계(`drivers/nvme/host/core.c`)
- 의미: NVMe request 완료를 block completion 경로로 넘기는 드라이버 레벨 완료 지점
- 레이턴시 역할: Device→Driver 완료 도착 시점의 1차 기준

#### `blk_mq_end_request` (선택적으로 병행)
- 계층 위치: blk-mq request 완료 API(`block/blk-mq.c`)
- 의미: request 단위 완료 처리 시작(내부적으로 `blk_update_request` 진행)
- 레이턴시 역할: 드라이버 완료와 block request 완료 사이 지연을 정량화

### Bio 완료 레벨

#### `bio_endio`
- 계층 위치: block bio 완료 처리 단계(`blk_update_request` 경유)
- 의미: bio 단위 상위 콜백/완료 통지
- 레이턴시 역할: request 내부의 개별 bio 완료 시점을 관측해 세부 완료 분해 가능

## 2️⃣ 완료 경로 포함 관계 설명

Driver IRQ 처리
  → nvme_complete_rq
     → blk_mq_end_request
        → blk_update_request
           → bio_endio

- `bio_endio`는 request 완료 내부 단계다. 즉, request가 완료되는 과정에서 포함되어 호출되는 bio 단위 완료 처리다.
- request 단위 완료는 "하나의 request 수명 종료"를 의미하고, bio 단위 완료는 request에 연결된 개별 bio 조각의 완료를 의미한다.
- 따라서 request 완료 시점과 bio 완료 시점은 1:1이 아닐 수 있으며, 부분 완료/다중 bio 구성에서 시차가 발생할 수 있다.
- 원격 completion(remote completion) 가능성(예: IPI/softirq 경유) 때문에, 제출 CPU와 완료 CPU가 다를 수 있음을 전제로 분석해야 한다.

## 3️⃣ 컨텍스트 분류 전략

- `trace_irq_handler_entry` ~ `trace_irq_handler_exit` 사이면 Hard IRQ 컨텍스트로 분류
- `trace_softirq_entry` ~ `trace_softirq_exit` 사이면 Soft IRQ 컨텍스트로 분류
- 그 외 completion은 Process/Workqueue 컨텍스트로 분류

completion 이벤트에서 함께 기록 권장:
- `in_hardirq()`
- `in_serving_softirq()`
- CPU ID
- request pointer 또는 bio pointer

이 조합을 기록하면 동일 request/bio의 완료가 어떤 컨텍스트에서 소비되었는지, 그리고 CPU 이동(로컬/원격 완료)이 있었는지 상관분석이 가능하다.

## 4️⃣ 권장 계측 조합 결론

### 최소 권장 세트
- IRQ: `trace_irq_handler_entry/exit`, `trace_softirq_entry/exit`
- 완료: `nvme_complete_rq`, `bio_endio`

### 확장 세트
- 최소 세트 + `blk_mq_end_request`
- 필요 시 request lifecycle 상관키(request pointer)와 bio pointer를 동시 저장

IRQ + request + bio를 함께 잡아야 안정적인 이유:
- IRQ 계층은 "왜/어디서 완료가 실행됐는지"를,
- request 계층은 "블록 단위 완료 경계"를,
- bio 계층은 "상위 I/O 조각 완료 경계"를 제공한다.

세 계층을 결합해야 completion 지연을 컨텍스트 지연, request 완료 지연, bio 완료 지연으로 분해할 수 있고,
단일 포인트 계측에서 생기는 해석 모호성을 크게 줄일 수 있다.

## 참고문헌

[1] Linux v6.6.1 drivers/nvme/host/core.c
[2] Linux v6.6.1 block/blk-mq.c
[3] Linux v6.6.1 kernel/irq/handle.c
[4] Linux v6.6.1 kernel/softirq.c
