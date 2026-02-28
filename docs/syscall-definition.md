## Syscall entry function 목록 (read / write / fsync)

### Read 계열

| Syscall | Entry function (x86-64) | 비고 |
|---|---|---|
| read | __x64_sys_read | 기본 read |
| pread64 | __x64_sys_pread64 | 오프셋 지정 |
| readv | __x64_sys_readv | vectored read |
| preadv | __x64_sys_preadv | vectored read + 오프셋 지정 |
| preadv2 | __x64_sys_preadv2 | vectored read + 오프셋/플래그 포함 |

### Write 계열

| Syscall | Entry function (x86-64) | 비고 |
|---|---|---|
| write | __x64_sys_write | 기본 write |
| pwrite64 | __x64_sys_pwrite64 | 오프셋 지정 |
| writev | __x64_sys_writev | vectored write |
| pwritev | __x64_sys_pwritev | vectored write + 오프셋 지정 |
| pwritev2 | __x64_sys_pwritev2 | vectored write + 오프셋/플래그 포함 |

### Fsync 계열

| Syscall | Entry function (x86-64) | 비고 |
|---|---|---|
| fsync | __x64_sys_fsync | 파일 데이터+메타데이터 동기화 |
| fdatasync | __x64_sys_fdatasync | 데이터 중심 동기화(메타데이터 일부 제외) |

### 참고문헌

[1] arch/x86/entry/syscalls/syscall_64.tbl. Android Open Source Project (kernel/common). https://android.googlesource.com/kernel/common/+/df2c1f38939aa/arch/x86/entry/syscalls/syscall_64.tbl
