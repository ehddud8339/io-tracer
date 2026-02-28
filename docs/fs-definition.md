# FS Layer Entry Function 정의 (Linux 6.6 기준)

기준 버전: v6.6.1

## 서론

FS layer entry function은 **VFS가 file->f_op->read_iter/write_iter/fsync로 디스패치할 때, 해당 ops 슬롯이 가리키는 실제 구현 함수**를 의미한다.
즉, VFS 공통 계층 바깥에서 각 파일시스템 구현으로 진입하는 첫 실제 처리 함수 집합이다.
Linux v6.6.1에서는 generic_* 계열 함수가 다수 파일시스템에서 공용 구현체로 연결되는 대표 사례가 많다.
이 특성 덕분에 generic_* 계열은 FS 레이어 레이턴시 계측 기준점으로 재사용성이 높고 비교 가능성이 좋다.
반면 파일시스템별 특화 경로는 별도 구현체를 사용하므로, 공용 구현체와 특화 구현체를 함께 구분해 보는 것이 정확하다.

## 정의 및 추적 범위

- 추적 경계는 VFS 함수(vfs_iter_read/vfs_iter_write/vfs_fsync_range 등)가 `file->f_op` 슬롯으로 넘긴 직후의 FS 실제 구현 함수다.
- ops slot 계측(`.read_iter/.write_iter/.fsync`)은 파일시스템 전반을 공통 방식으로 관찰하기에 유리하다.
- 실제 구현체 계측(generic_* 또는 FS 전용 함수)은 내부 처리 비용을 더 직접적으로 분리해 볼 수 있다.

## Read 계열 후보군

### 범용 regular file에서 흔한 실제 구현체

- `generic_file_read_iter`

### VFS→FS 디스패치 대상(ops slot)

- `struct file_operations`의 `.read_iter`

## Write 계열 후보군

### 범용 regular file에서 흔한 실제 구현체

- `generic_file_write_iter`
- `__generic_file_write_iter`

`__generic_file_write_iter`는 내부 핵심 루틴 성격, 계측 범위 축소 시 고려 대상이다.

### VFS→FS 디스패치 대상(ops slot)

- `struct file_operations`의 `.write_iter`

## Fsync 계열 후보군

### 범용 regular file에서 흔한 실제 구현체

- `generic_file_fsync`
- `__generic_file_fsync`

`__generic_file_fsync`는 내부 핵심 루틴 성격, 계측 범위 축소 시 고려 대상이다.

### 특수/가상 파일에서 흔한 구현체

- `noop_fsync` (no-op 동작)

### VFS→FS 디스패치 대상(ops slot)

- `struct file_operations`의 `.fsync`

## 요약 표

| I/O | FS entry 후보(실제 구현체) | 비고 |
|---|---|---|
| read | generic_file_read_iter | 범용 regular file read_iter 경로에서 흔한 공용 구현체 |
| write | generic_file_write_iter | 범용 regular file write_iter 경로에서 흔한 공용 구현체 |
| write | __generic_file_write_iter | 내부 핵심 루틴 성격, 계측 범위 축소 시 고려 |
| fsync | generic_file_fsync | 범용 regular file fsync 경로에서 흔한 공용 구현체 |
| fsync | __generic_file_fsync | 내부 핵심 루틴 성격, 계측 범위 축소 시 고려 |
| fsync | noop_fsync | 특수/가상 파일 계열에서 사용하는 no-op 동작 |

## 설계 메모

- generic_*을 FS entry 후보로 보는 이유는, 다수 파일시스템이 공용 구현체를 `file_operations`의 ops 슬롯에 연결해 사용하기 때문이다.
- ops slot(`.read_iter/.write_iter/.fsync`) 계측은 커버리지가 넓고 인터페이스 안정성이 높지만, 실제 내부 구현 비용 분해는 제한적일 수 있다.
- 실제 구현체 계측은 레이턴시 원인 분해에 유리하나 파일시스템별 함수 다양성으로 인해 이식성과 일반성이 낮아질 수 있다.
- `__generic_*`을 entry로 삼으면 더 좁은 범위와 내부 경로 중심 계측이 가능하지만, 상위 래퍼에서 수행되는 검증/전처리 구간이 제외될 수 있어 해석 시 주의가 필요하다.

## 참고문헌

[1] Linux v6.6.1 include/linux/fs.h. rabexc.org. https://sbexr.rabexc.org/latest/sources/5a/0a48e80e45a2df.html
