# Block Layer Entry Function 정의 (Linux 6.6 기준)

기준 버전: v6.6.1

## 서론

Block layer entry function은 **상위 레이어가 구성한 struct bio가 블록 레이어 코어로 제출되는 지점**을 의미한다.
read/write 경로는 최종적으로 bio 제출 경로에서 수렴하고, 블록 코어는 이를 큐/드라이버 경계로 전달한다.
fsync는 파일시스템 동기화 의도가 블록 계층에서 flush/FUA 의미(예: REQ_PREFLUSH, REQ_FUA)로 관측되며, 필요 시 flush 요청 처리로 이어진다.
따라서 block layer 계측은 상위 VFS/FS 의도를 실제 디바이스 제출 의미로 변환하는 경계를 포착하는 데 핵심적이다.

## 정의 및 계측 범위

- 범위 시작점: bio가 `submit_bio` 계열로 들어오는 시점
- 범위 종점(선택): blk-mq 제출(`blk_mq_submit_bio`) 또는 `disk->fops->submit_bio` 드라이버 경계
- 계층 경계 관점: VFS/FS 이후의 공통 block core 처리와 스태킹 재제출(re-submit) 경로를 분리 관찰
- fsync 관점: 파일시스템의 동기화 의도가 block 계층에서는 flush/FUA 플래그/요청 형태로 나타나는지 확인

## Block Core Entry 후보군

### `submit_bio`
- 호출 위치 의미: 파일시스템/상위 블록 계층에서 최종 제출 API로 사용하는 공개 진입점
- 계층 경계 역할: 상위 레이어 I/O를 block core 공통 경로로 넘기는 1차 경계
- latency 계측 장단점: 장점은 커버리지가 넓고 해석이 단순함, 단점은 내부 재제출/분기를 구분하기 어렵다는 점
- stacking 드라이버 의미: dm/md 등 재제출 경로가 이어져도 상위 제출 시점 기준 통합 관찰이 가능

### `submit_bio_noacct`
- 호출 위치 의미: 스태킹 드라이버가 하위 계층으로 I/O를 재제출할 때 사용하는 경로
- 계층 경계 역할: 사용자/파일시스템 최초 제출과 재제출 제출을 구분하는 실무 경계
- latency 계측 장단점: 장점은 stacking 환경에서 재제출 지연 분리가 가능, 단점은 최초 제출 대비 카운팅/의미 정합성 주의 필요
- stacking 드라이버 의미: dm 경로 분석에서 핵심이며, 어느 계층에서 재제출이 발생하는지 파악 가능

### `submit_bio_noacct_nocheck`
- 호출 위치 의미: noacct 경로에서 각종 체크 이전/이후 흐름을 더 가볍게 통과시키는 내부 제출 단계
- 계층 경계 역할: block core 내부 정책/체크와 실제 제출 루프 사이 경계
- latency 계측 장단점: 장점은 block core 내부 오버헤드 분해 가능, 단점은 외부 의미가 약해 해석 난이도 상승
- stacking 드라이버 의미: 재제출이 누적될 때 내부 루프 진입 지점으로 유용

### `__submit_bio`
- 호출 위치 의미: 내부 실제 제출 루틴으로, mq 경로 또는 `disk->fops->submit_bio`로 분기
- 계층 경계 역할: block core에서 드라이버/큐 제출로 넘어가는 핵심 분기점
- latency 계측 장단점: 장점은 제출 직전 핵심 경계 계측 가능, 단점은 내부 함수라 버전/최적화 민감도 존재
- stacking 드라이버 의미: 여러 계층 재진입 뒤 최종 제출 시점 관찰에 유리

### `__submit_bio_noacct`
- 호출 위치 의미: `bd_has_submit_bio` 디바이스 경로에서 bio_list 루프를 통해 재귀 재제출을 제어
- 계층 경계 역할: stacking/재진입 환경에서 동일/하위 큐 정렬 및 재처리 경계
- latency 계측 장단점: 장점은 스태킹 루프 지연 분석에 강함, 단점은 경로가 복잡해 단독 지표 해석이 어려움
- stacking 드라이버 의미: dm 같은 계층형 드라이버에서 재제출 누적 구조를 파악하는 핵심 포인트

### `__submit_bio_noacct_mq`
- 호출 위치 의미: `bd_has_submit_bio`가 없는 경로에서 mq 제출 루프를 처리
- 계층 경계 역할: block core에서 blk-mq 중심 처리로 이어지는 내부 경계
- latency 계측 장단점: 장점은 mq 경로 지연을 직접 포착, 단점은 상위 의미(최초 제출/재제출)와 결합 해석이 필요
- stacking 드라이버 의미: 스태킹이 적은 mq 직결 경로와 stacking 경로를 비교하는 기준점

## Stacking / Re-submit 경로

- 스태킹 드라이버(dm/md)는 상위에서 받은 bio를 하위 디바이스로 재구성/재제출한다.
- 이때 `submit_bio_noacct` 계열이 핵심 경로가 되며, 단일 `submit_bio`만 보면 계층별 재제출 비용이 숨겨질 수 있다.
- `__submit_bio_noacct`의 bio_list 루프는 재귀 제출을 제어하면서 동일 큐/하위 큐 흐름을 정리한다.
- 계측 시에는 "최초 제출(`submit_bio`)"과 "재제출(`submit_bio_noacct*`)"을 분리 태깅해야 지연 원인 분리가 쉬워진다.

## Driver 경계 (block_device_operations)

### `struct block_device_operations::submit_bio`
- 호출 위치 의미: block core 내부 `__submit_bio`가 `disk->fops->submit_bio`로 디스패치하는 드라이버 경계
- 계층 경계 역할: block core 공통 처리 이후, 디바이스/드라이버 고유 처리로 넘어가는 직접 경계
- latency 계측 장단점: 장점은 드라이버 책임 구간을 명확히 분리, 단점은 드라이버별 구현 편차로 비교가 어려울 수 있음
- stacking 드라이버 의미: 상위 stacking 처리 이후 최종 하위 디바이스 제출 지점으로서 end-to-end 분해에 중요

## 요약 표

| 계층 경계 | Entry 후보 | 성격 | 계측 의미 |
|---|---|---|---|
| Block core 공개 진입 | submit_bio | 상위 레이어 공용 제출 API | block layer 진입의 1차 기준점, 전체 커버리지 확보 |
| Block core 재제출 | submit_bio_noacct | stacking 재제출용 공개 API | dm/md 재제출 지연 및 계층별 비용 분리 |
| Block core 내부 단계 | submit_bio_noacct_nocheck | noacct 내부 제출 단계 | 체크/정책 구간과 제출 루프 구간 분해 |
| Block core 내부 핵심 | __submit_bio | 실제 제출 분기(blk-mq 또는 fops) | 드라이버/큐 경계 직전의 핵심 타점 |
| Block core stacking 루프 | __submit_bio_noacct | bio_list 기반 재진입 제어 | stacking 루프 지연과 재귀 제출 구조 분석 |
| Block core mq 루프 | __submit_bio_noacct_mq | mq 경로 내부 루프 | blk-mq 중심 내부 제출 지연 포착 |
| Driver 경계 | struct block_device_operations::submit_bio | disk fops 디스패치 엔트리 | block core 이후 드라이버 책임 구간 분리 |

## 설계 메모

- `submit_bio`를 1차 기준으로 보는 이유는 상위 레이어 bio 제출이 가장 먼저 공통 block core 의미로 수렴하는 공개 진입점이기 때문이다.
- `submit_bio` vs `__submit_bio` 계측 차이: 전자는 상위 관점(커버리지/비교성), 후자는 내부 제출 직전 관점(정밀 분해/복잡도 증가)이다.
- stacking driver 환경에서는 `submit_bio_noacct` 계열이 재제출 흐름을 드러내므로, dm 경로 병목 분석에 사실상 필수 포인트다.
- block core 계측은 계층 경계 가시성이 좋고, blk-mq 계측은 큐/하드웨어 경로 지연 분석에 유리해 목적에 따라 병행하는 것이 좋다.
- fsync는 block layer에서 flush/FUA 의도로 관측되며(예: REQ_PREFLUSH/REQ_FUA), 실제 요청 처리에서는 flush 동작으로 구체화되는지 함께 확인해야 한다.

## 참고문헌

[1] Linux v6.6.1 block/blk-core.c. Codebrowser. https://codebrowser.dev/linux/linux/block/blk-core.c.html
[2] Linux v6.6.1 include/linux/blkdev.h. rabexc.org. https://sbexr.rabexc.org/latest/sources/83/a709fe8a086b02.html
