# VFS Layer Entry Function 정의 (Linux 6.6 기준)

VFS layer entry function은 **VFS 공통 경로에서 file_operations로 디스패치되기 직전의 공통 진입 지점**을 뜻한다.
이 지점은 개별 파일시스템(ext4, xfs 등) 구현으로 분기되기 전, 공통 처리 구간을 계측할 수 있다는 점에서 중요하다.
syscall별 변형(read/readv/preadv2, write/writev/pwritev2)을 파일시스템별 내부 함수가 아닌 공통 경계에서 묶어 관찰할 수 있다.
또한 file->f_op->read_iter/write_iter/fsync 호출 직전/직후를 경계로 잡으면 VFS 오버헤드와 하위 파일시스템 처리 구간을 분리해 레이턴시를 해석하기 유리하다.

## Read 계열

| I/O 명령 | VFS entry 후보 | 성격 | 설명 |
|---|---|---|---|
| read 계열 | vfs_read | legacy + iter 브리지 | 전통 read 진입점이며, 내부에서 file->f_op->read 또는 file->f_op->read_iter 경로로 이어진다. 레이턴시 계측 시 사용자 read 계열의 상위 공통 경계로 사용 가능하다. |
| read/readv/preadv 계열 | vfs_iter_read | iter 기반 전역 진입점 | iov_iter 기반 동기 read 경로를 공통 처리하며 readv/preadv 계열 흡수에 유리하다. file->f_op->read_iter 디스패치 직전 경계로 적합하다. |
| read 계열(kiocb) | vfs_iocb_iter_read | iter + kiocb 기반 전역 진입점 | struct kiocb와 iov_iter를 함께 받아 call_read_iter를 통해 file->f_op->read_iter로 연결된다. kiocb 컨텍스트를 보존한 레이턴시 계측 경계로 유의미하다. |

## Write 계열

| I/O 명령 | VFS entry 후보 | 성격 | 설명 |
|---|---|---|---|
| write 계열 | vfs_write | legacy + iter 브리지 | 전통 write 진입점이며, 내부에서 file->f_op->write 또는 file->f_op->write_iter 경로로 이어진다. write 계열 상위 공통 경계 계측에 적합하다. |
| write/writev/pwritev 계열 | vfs_iter_write | iter 기반 전역 진입점 | iov_iter 기반 동기 write 경로를 공통 처리해 syscall 변형 흡수에 유리하다. file->f_op->write_iter 디스패치 경계로 레이턴시 측정 포인트를 일관화할 수 있다. |
| write 계열(kiocb) | vfs_iocb_iter_write | iter + kiocb 기반 전역 진입점 | struct kiocb + iov_iter 기반으로 call_write_iter를 통해 file->f_op->write_iter로 연결된다. 비동기 문맥과 연계된 write 레이턴시 경계 정의에 유리하다. |

## Fsync 계열

| I/O 명령 | VFS entry 후보 | 성격 | 설명 |
|---|---|---|---|
| fsync/fdatasync | vfs_fsync | fsync 상위 전역 진입점 | fsync/fdatasync 공통 래퍼로 vfs_fsync_range에 위임된다. 상위 동기화 요청 경계를 단일 포인트로 계측하기 좋다. |
| fsync/fdatasync(range) | vfs_fsync_range | fsync 범위 기반 전역 진입점 | start/end/datasync를 받아 file->f_op->fsync로 직접 디스패치한다. VFS에서 파일시스템 fsync 구현으로 넘어가는 최종 공통 경계로 레이턴시 계측 의미가 크다. |

## 설계 메모

- readv, preadv2, writev, pwritev2 같은 syscall 변형을 흡수하려면 vfs_iter_* 계열 중심 계측이 유리하다.
- vfs_iocb_iter_*는 kiocb 문맥을 포함하므로 AIO/io_uring 계열과의 경로 연관성을 해석할 때 보조 경계로 유용하다.
- static helper는 커널 버전/컴파일 최적화에 따라 심볼 안정성이 낮을 수 있어, 장기 유지보수 관점에서는 EXPORT되는 전역 함수 기준 계측이 안전하다.

## 참고문헌

[1] Linux v6.6.1, fs/read_write.c. rabexc.org.
[2] Linux v6.6.1, fs/sync.c. rabexc.org.
